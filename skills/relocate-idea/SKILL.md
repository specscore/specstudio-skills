---
name: relocate-idea
description: |
  Thin wrapper over the `specscore idea relocate` CLI verb. Relocates
  an Idea or sidekick-seed artifact from the current repo to another
  SpecScore-managed repo by shell-execing the CLI and surfacing its
  output verbatim. On success, appends one JSON line to
  `.specscore/destination-resolution-log.jsonl` in the source-repo
  cwd so future destination-resolution tuning can learn from
  misroute corrections.
  Triggers: "specstudio:relocate-idea", "/relocate-idea", "relocate
  this idea", "move this seed to another repo".
aliases: [relocate-idea]
---

# Relocate Idea

A thin shell over [`specscore idea relocate`](https://github.com/specscore/specscore-cli/blob/main/spec/features/cli/idea/relocate/README.md). All relocation mechanics — pre-flight clean-tree checks, file copy + in-file rewrite, cross-repo link cleanup, per-repo commits, rollback on failure — live in the CLI. This skill exists to:

1. Make the verb conversation-triggerable (`/relocate-idea`).
2. Prompt for missing arguments before shelling out.
3. Surface the CLI's stdout/stderr verbatim and propagate its exit code.
4. Append an opt-in mismatch-log line on success so a future Feature can tune the sidekick destination-resolution prompt against real correction signal.

## Hard Gate

<HARD-GATE>
This skill MUST NOT replicate any of the CLI verb's logic. No file copy, no in-file rewrite, no link rewriting, no git commit/stage, no rollback computation happens in this skill — every one of those concerns lives inside `specscore idea relocate` and stays there. The skill's job is argument collection, shell-out, output surfacing, and a single best-effort log-line append. If the CLI is not on PATH, surface the install path (`/specscore:install`) and stop — do NOT fall back to an ad-hoc reimplementation.
</HARD-GATE>

## When to Use

- The user (or you, on the user's behalf) determines that an Idea or seed in the current repo belongs in a different SpecScore-managed repo.
- The user is reacting to a sidekick-capture destination mismatch: the seed went to repo A but should be in repo B.
- The user explicitly types `/relocate-idea` or one of the other triggers.

## Pre-flight

1. **CLI present (mandatory-class detection).** `relocate-idea` is a thin wrapper over `specscore idea relocate` with no fallback — a **mandatory-class** skill per [`../shared/cli-detection.md`](../shared/cli-detection.md). Do **not** run a standalone `command -v` probe; detect via the relocate call's exit status (see `## Invocation`). If that call exits `127` (binary not installed), emit the standardized install message verbatim and stop without writing:
   > The `specscore` CLI is not installed. Invoke `/specscore:install` to see install options, or install from <https://specscore.md/install>. Then retry your command.
   On any other non-zero exit, surface the CLI's error. Stop. Do not proceed.

2. **Collect arguments.** The skill needs two values:
   - **slug** — the artifact's slug (basename of `spec/ideas/<slug>.md` or `spec/ideas/seeds/<slug>.md` in the source repo).
   - **target** — value for `--to-repo`. Either a repo slug (no `/` — resolved via sibling-dir scan against each candidate's `project.repo`) or a path (contains `/` — resolved relative to source project root, or absolute).

   If either is missing from the trigger arguments, **ask the user**, one batched question, before shelling out. Suggested phrasing:
   > Which artifact do you want to relocate (slug) and where to (`--to-repo` value)?

   Do not infer either value silently from cwd or recent history. The CLI's slug-resolution (REQ `slug-resolves-idea-or-seed` — Idea first, then seed) handles the path lookup once you have the bare slug.

3. **Optional flag.** If the user passed `--no-commit` (verbatim, no synonyms), pass it through to the CLI. Otherwise, omit it.

## Invocation

Shell-exec the CLI verb:

```bash
specscore idea relocate <slug> --to-repo=<target> [--no-commit]
```

Run it from the source repo's working directory (the user's cwd at invocation time). Capture both stdout and stderr. Capture the exit code.

The skill MUST NOT add additional flags, environment variables, or pre/post commands beyond the shell-exec itself. The CLI's [`cli/idea/relocate`](https://github.com/specscore/specscore-cli/blob/main/spec/features/cli/idea/relocate/README.md) Feature is the contract — don't paper over it.

## Output handling

### On exit 0 (success)

1. Surface the CLI's full stdout to the host conversation verbatim. The format the CLI emits is the [stdout-format contract](https://github.com/specscore/specscore-cli/blob/main/spec/features/cli/idea/relocate/README.md#req-stdout-format) — per-repo lines plus a summary line. Do not paraphrase, summarize, or add inference.

2. Append one JSON line to `.specscore/destination-resolution-log.jsonl` (see "Mismatch log" below).

3. Exit 0.

### On any non-zero exit

1. Surface the CLI's full stderr to the host conversation **verbatim**, including any user-runnable rollback commands the CLI printed (e.g., `git -C <repo> reset HEAD~1 --hard`). Do NOT paraphrase, summarize, or strip whitespace/formatting.

2. Propagate the CLI's exit code as the skill's exit code. The CLI's [exit-code contract](https://github.com/specscore/specscore-cli/blob/main/spec/features/cli/idea/relocate/README.md#exit-codes) defines the semantics; don't reinterpret them.

3. Do NOT append a mismatch-log line on failure — the log records corrections, not attempts.

## Mismatch log

On exit-0 success only, append one single-line JSON object to `.specscore/destination-resolution-log.jsonl` in the user's cwd at the moment the skill was invoked (the source repo — i.e., the repo where the misfiled artifact lived before the relocate). Create the `.specscore/` directory lazily if it doesn't exist.

### Record schema

```json
{
  "ts": "<ISO-8601 UTC, e.g., 2026-05-21T14:32:00Z>",
  "kind": "idea" | "seed",
  "slug": "<artifact-slug>",
  "original_repo": "<source-repo's project.repo value>",
  "correct_repo": "<target-repo's project.repo value>"
}
```

Field sources:

- **ts** — current time, UTC, ISO-8601.
- **kind** — parse from the CLI's stdout: the `moved`/`received`/`updated-links` lines are formatted `<repo-slug>: <action> <kind> <slug>  [<sha>]`. The `<kind>` token is the third field after the colon (`idea` or `seed`).
- **slug** — the slug argument the user supplied.
- **original_repo** — the `<repo-slug>` on the line whose action is `moved` (the source). If parsing fails, fall back to reading `project.repo` from the source repo's `specscore.yaml`.
- **correct_repo** — the `<repo-slug>` on the line whose action is `received` (the target).

Implementations MAY add additional fields (e.g., the agent's pick at original-write time, retrieved via session state or absent if unknown). Consumers tolerate unknown fields. Schema evolution is permissive at this stage.

### Best-effort discipline (REQ `relocate-skill-writes-mismatch-log` + AC `relocate-skill-log-write-failure-non-blocking`)

The log write is **best-effort**. On any failure (directory creation fails, file unwritable, disk full, permission denied):

1. Display a single short warning line to the host conversation:
   > Warning: could not append destination-resolution log line: `<error>`. The relocate succeeded.

2. Do NOT modify the skill's exit code — still propagate the CLI's exit-0.
3. Do NOT retry. The seed write being durable matters; the log being durable does not.

## Anti-patterns

| Anti-pattern | Why it's wrong |
|---|---|
| Reading the source artifact, doing the in-file `specscore/` → `specscore/` rewrite locally, then "letting the CLI commit" | The CLI's contract is the *whole* relocate, not a tail end. Splitting the rewrite into the skill creates two implementations of the same substitution rules. |
| Paraphrasing the CLI's stderr ("The CLI hit a conflict; you may want to ...") | The CLI's stderr includes exact rollback commands per [REQ:stop-on-first-commit-failure](https://github.com/specscore/specscore-cli/blob/main/spec/features/cli/idea/relocate/README.md#req-stop-on-first-commit-failure). Paraphrasing strips actionability. |
| Auto-running `git commit` in the source repo because "the CLI's --no-commit left things staged" | If the user passed `--no-commit`, they want to commit manually. The skill's exit reproduces the CLI's behavior — staged-not-committed is the requested outcome. |
| Adding `--include-code` or other flags the CLI doesn't yet support | The CLI's scope is the SpecScore-doc relocate. Code-annotation cleanup is deferred to a later CLI version. The skill should not invent flags the CLI rejects. |
| Hardcoding `--to-repo=specscore` because the most common target is the specscore repo | Forecloses on every other target. The skill is a generic wrapper, not a `specscore`-targeted shortcut. |

## See also

- CLI verb spec: [`cli/idea/relocate`](https://github.com/specscore/specscore-cli/blob/main/spec/features/cli/idea/relocate/README.md) — the source of truth for behavior, exit codes, and stdout/stderr format.
- Companion Feature: [`sidekick-capture/destination-resolution`](../../spec/features/sidekick-capture/destination-resolution/README.md) — the broader context in which this skill exists; the sidekick pre-write hook + this recovery skill ship together.
- Source Idea: [`idea-skills-destination-resolution`](../../spec/ideas/idea-skills-destination-resolution.md) — the rationale for the two-Feature change and the design constraints both halves inherit.
