---
name: sidekick
description: |
  Captures a sideline idea as a lint-clean seed at spec/ideas/seeds/<slug>.md
  without derailing the host task. Validates a one-liner, writes the seed with
  required frontmatter and an H1 heading, emits the sidekick-idea.captured
  event, and exits. No deliberation, no dedupe across sessions, no
  auto-promotion happens at this layer — those belong to later Features that
  subscribe to the event.
  Triggers: "specstudio:sidekick", "/sidekick", "capture a sidekick idea",
  "side-kick this", "park this idea".
aliases: [sidekick]
---

# Sidekick

Single-mode skill: **capture-and-exit**. Writes one seed, emits one event, returns. The host invokes this skill from inside its own task to durably record a sideline idea without breaking flow.

For *when* a host should invoke this skill, read [`shared/sidekick-capture.md`](../shared/sidekick-capture.md). This SKILL.md describes the skill's own behavior — what it accepts, what it writes, and what it emits.

## When to Use

- A host skill (`specstudio:ideate`, `specstudio:specify`, third-party adopter) has detected a sideline idea per `shared/sidekick-capture.md` and is invoking on the host's behalf.
- A user has typed `/sidekick <one-liner>` directly to park an idea.

## Anti-Pattern: Content-Deliberation at Capture

This skill MUST NOT deliberate the *merits* of the seed, ask follow-up questions about the idea's content, scan existing seeds for content duplicates, or trigger any review pipeline. Content-deliberation (is this idea worth capturing? does it duplicate an existing one? does it need scoping?) is the consilium's job (a separate Feature, not yet built). If a host invocation arrives that requires content-deliberation, that is a contract violation in the host — capture the seed and return.

The destination-resolution UX in multi-repo workspaces (see [Destination resolution](#destination-resolution-multi-repo-workspaces) below) is NOT content-deliberation: it is a pre-write confirmation of *where* the seed is durably stored, never *whether* it should be. The skill still writes (or aborts cleanly) on a single user response; no follow-up clarifying questions about the idea's substance ever happen.

## Input

The skill accepts:

- **One-liner** (required): a string of 1–500 characters after trimming leading/trailing whitespace.
- **`--body <markdown>`** (optional): additional markdown content that follows the H1 heading. Total body length (H1 line + body) MUST NOT exceed 2000 characters.

## Validation rules (REQs `input-validation`, `writes-seed-artifact`)

Reject with a clear error message and exit non-zero in any of these cases:

| Case | Error message |
|---|---|
| One-liner is empty or whitespace-only | `Empty one-liner. Provide a one-line description, 1–500 chars.` |
| One-liner exceeds 500 chars after trimming | `One-liner too long (<N> chars). Max is 500.` |
| Body, combined with the H1 line, exceeds 2000 chars total | `Body too long (<N> chars). Max body (incl. H1 line) is 2000 chars.` |
| Unknown flag (anything starting with `--` that is not `--body`) | `Unknown flag: <flag>. Supported: --body.` |

On rejection, the skill MUST NOT create any file, MUST NOT emit any event, and MUST exit non-zero.

## Slug derivation (REQ `seed-slug-derivation`)

The skill does **not** derive the slug — `specscore sidekick new` does, from the one-liner. The canonical algorithm **and** the slug-format validation live in the CLI (the `cli/sidekick/new` Feature), so the skill carries no copy to keep in sync. In the common case the skill omits `--slug` and lets the CLI derive. It supplies `--slug <slug>` to override — either a specific slug the user asked for, or a `-N` disambiguator for a [collision](#collision-disambiguation-req-writes-seed-artifact) — and lets the CLI validate it (an invalid slug exits `2`). The skill does **not** restate the format rules.

## Destination resolution (multi-repo workspaces)

Implements [`sidekick-capture/destination-resolution`](../../spec/features/sidekick-capture/destination-resolution/README.md). The output of this section is `<destination-repo-root>` — the absolute path of the repo the seed will be written into. Every subsequent section (Collision disambiguation, Writing the seed file, Output) uses that value.

In a **single-repo workspace** (the source project is the only `specscore.yaml`-bearing directory in its parent), this entire section is skipped: `<destination-repo-root>` is the source project root, and the skill goes directly to "Collision disambiguation". Today's behavior preserved.

### Step 1 — Sibling-dir scan (REQ `sibling-dir-scan`)

Before any seed-file write, scan the parent of the source project for sibling repos containing `specscore.yaml`:

- Walk the source project's parent directory (`..`).
- For each immediate-child entry:
  - **Skip** if the name starts with `.` (hidden dirs like `.git`, `.specscore`).
  - **Skip** if the entry is a symlink whose target resolves outside `..` (no symlink-out follows).
  - **Include** if the entry is a directory containing a `specscore.yaml`.
- Always include the source project itself as a candidate (unconditional).

The scan MUST complete in under 500ms for ≤20 siblings. For each candidate, read its `specscore.yaml` and capture `project.repo` — that's the candidate identity used downstream.

If **zero siblings** are matched (only the source project is a candidate), skip every step below this and proceed to "Collision disambiguation" using the source project root as `<destination-repo-root>`. REQ `single-repo-bypass`.

If **≥1 siblings** are matched, continue to step 2.

### Step 2 — Invoke the deliberation helper (REQ `invokes-helper-before-write`)

Read [`shared/destination-resolution.md`](../shared/destination-resolution.md) and assemble the prompt by substituting:

- `<seed-one-liner>` in section (a): the user's actual one-liner (verbatim, no normalization).
- The candidates table in section (b): one row per candidate. Each row contains:
  - **`project.repo` cell:** the value from the candidate's `specscore.yaml`. If `project.repo` is unset, fall back to the candidate's directory basename (the agent treats the fallback identifier identically — see section (b)'s fallback clause in `shared/destination-resolution.md`).
  - **Features cell:** the names of `spec/features/*/` top-level directories AND their immediate sub-directories (where they exist), formatted as `<top-level>` for plain top-level entries and `<top-level>/<sub-dir>` for nested ones. Comma-separated, capped at 20 entries per candidate with `, …` truncation when more exist. Use the literal `(no Features)` when the candidate has no `spec/features/*/` directories.

Pass the assembled prompt to the host AI agent. No seed file is written until this and the confirmation UX complete.

### Step 3 — Parse the agent's response (REQ `parses-agent-response`)

Parse per the helper's section (c) contract:

- **Exactly one line.** Multi-line response → malformed.
- **≤120 characters total** (counting the `; ` separator and reason).
- **Shape `<repo>; <reason>`** — `<repo>` (case-insensitive, whitespace-trimmed) MUST match exactly one candidate's `project.repo`.
- The literal token **`UNCERTAIN`** (case-sensitive, alone on its line, no other content) is the helper's section (d) escape clause.

Routing:

- Well-formed → step 5 (pick-with-reason confirmation).
- Malformed OR `UNCERTAIN` → step 4 (one retry).

### Step 4 — Retry, then fall through (REQ `malformed-or-uncertain-response`)

On the FIRST malformed/`UNCERTAIN` response only, re-invoke the helper. Same assembled prompt, with this corrective instruction appended at the end of the prompt body:

> Your previous response was unparseable; reply EXACTLY in `<repo>; <reason>` ≤120 chars, where `<repo>` is one of: `<comma-separated-candidate-slugs>`.

Parse the retry per step 3.

- Well-formed → step 5.
- Malformed OR `UNCERTAIN` (second time) → step 6 (ask-without-pre-fill).

There is NO second retry. The agent gets two chances; the user picks next.

### Step 5 — Pick-with-reason confirmation (REQ `shows-pick-with-reason`)

Display this exact line to the user (whitespace and punctuation literal):

```
Routing to <repo> because <reason> — press enter to accept, type other to override.
```

Where `<repo>` is the parsed pick and `<reason>` is the agent's parsed reason verbatim (including the agent's own punctuation choices). The prompt MUST appear as the next message in the host conversation and MUST block further sidekick progress until user input arrives.

On user input:

- **Empty input** (whitespace-only, including bare enter): route to the agent's picked candidate. `<destination-repo-root>` becomes that candidate's path. Proceed to "Collision disambiguation". REQ `accepts-enter-as-route-to-agent-pick`.
- **Non-empty input**: treat as an override per step 7.

### Step 6 — Ask-without-pre-fill (REQ `shows-ask-without-pre-fill`)

Display the candidate list followed by the prompt line. Format (whitespace and punctuation literal):

```
1. <repo-slug-1>
2. <repo-slug-2>
3. <repo-slug-3>
...
Type a number, a repo slug, or a path to override — or press enter to abort the capture.
```

Each `<repo-slug-N>` is a candidate's `project.repo`. Ordering: source project FIRST, then siblings alphabetically by `project.repo`. Block until user input arrives.

On user input:

- **Empty input** (whitespace-only): abort the capture. REQ `enter-aborts-in-ask-flow`. The skill MUST NOT write a seed file, MUST NOT emit the `sidekick-idea.captured` event, and MUST print one short line:
  ```
  Capture aborted: no destination chosen.
  ```
  Exit 0 (clean abort, not an error).
- **Numeric input `N` where 1 ≤ N ≤ candidate-count**: route to that list position. `<destination-repo-root>` becomes that candidate's path. Proceed to "Collision disambiguation". REQ `accepts-numbered-selection-in-ask-flow`.
- **Numeric input out of range, or any non-numeric non-empty input**: treat as an override per step 7.

### Step 7 — Override (REQ `accepts-override-input`)

Applies to both step 5 (pick-with-reason) and step 6 (ask-without-pre-fill) when input is non-empty and not a list-position number.

Interpret per the same form rules as the [`cli/idea/relocate --to-repo`](https://github.com/specscore/specscore-cli/blob/main/spec/features/cli/idea/relocate/README.md#req-target-repo-resolution) flag — `/` is the discriminator:

- **No `/` in input** → repo slug. Match against the sibling-dir scan result (step 1) by `project.repo`:
  - **Single match** → use that repo as `<destination-repo-root>`. Proceed to "Collision disambiguation".
  - **Zero matches** → display `No SpecScore-managed candidate has project.repo=<input>. Try one of: <comma-separated-candidate-slugs>, or a path containing /.` and **re-display the same confirmation prompt** the user was on (step 5 or step 6).
  - **Multiple matches** → display `Multiple candidates declare project.repo=<input>: <paths>. Use a path with / to disambiguate.` and re-display the same prompt.
- **`/` in input** → path. Resolve relative to the source project root (or absolute if starting with `/`). The path MUST be a directory containing a `specscore.yaml`:
  - **Valid path** → use that directory as `<destination-repo-root>`. Proceed to "Collision disambiguation".
  - **Path missing, not a directory, or lacks `specscore.yaml`** → display `<input> is not a SpecScore-managed directory (no specscore.yaml at that path).` and re-display the same confirmation prompt.

No seed file is written on invalid override. Re-prompts always use the SAME confirmation form the user was on (don't switch flows mid-resolution).

## Collision disambiguation (REQ `writes-seed-artifact`)

The skill never overwrites an existing seed, and it does **not** pre-scan the filesystem: it lets the creation call surface a collision and resolves it through the CLI.

- The default call omits `--slug`. The CLI derives the slug and, when `<destination-repo-root>/spec/ideas/seeds/<slug>.md` already exists, exits `1` (Conflict) naming that path — nothing is written.
- On Conflict, take the base slug from the reported path and retry the same call with `--slug <base>-2`; if that also conflicts, `-3`, `-4`, … until it succeeds. **Never** pass `--force`.

(The same `--slug` seam carries a user-requested slug — see [Slug derivation](#slug-derivation-req-seed-slug-derivation); a `-N` disambiguator is just one use of it.) The slug of the written seed (with any `-N` suffix) is read back from the seed for the event payload.

## Creating the seed (required-CLI) (REQ `writes-seed-artifact`, REQ `seed-frontmatter-schema`)

**Creation is required-CLI.** `specscore sidekick new` is the ONLY way to write the seed file — it produces a lint-clean seed (the closed 8-key `sidekick-seed` frontmatter + the `# <one-liner>` H1 body) by construction and bootstraps `spec/ideas/seeds/` and the ancestor indexes for you. The skill carries **no embedded seed template**: the CLI scaffold is the single source of truth for the seed's structure. This follows the Required-CLI Artifact Creation policy — see the **Creation-class row** in [`../shared/cli-detection.md`](../shared/cli-detection.md).

The skill still owns the *orchestration the CLI does not*: *where* the seed lands ([Destination resolution](#destination-resolution-multi-repo-workspaces) → `--project`), the capture metadata (`--captured-by`, `--captured-during`, `--trigger`), an optional slug override (`--slug` — a user-requested slug or a `-N` [collision](#collision-disambiguation-req-writes-seed-artifact) suffix; the CLI derives the slug otherwise), the [back-link](#source-artifact-back-link-reqs-writes-back-link-to-source-artifact-source-artifact-path-resolution-back-link-section-format-back-link-best-effort), and the [event](#event-emission-req-emits-captured-event-req-event-payload-schema). It passes these as flags rather than hand-writing the file.

### Determining `captured_by`

- If invoked from inside another skill, `captured_by` is the invoking skill's id in `<plugin>:<skill>` form (e.g., `specstudio:specify`).
- If invoked directly by the user (typed `/sidekick`), `captured_by` is the literal string `user`.
- The skill does not validate the format — the caller supplies the value, passed verbatim via `--captured-by`. (Free-form per REQ `seed-frontmatter-schema`.)

### Determining `captured_during`

- Invoking skill or caller supplies the spec path of the active artifact (e.g., `spec/features/skills/init`), or `null` if there is no active spec context (e.g., a bare `/sidekick` from the user outside a host session).
- Pass it verbatim via `--captured-during` when present; **omit the flag** when it is `null` (the CLI then writes the literal `null`).

### Determining `trigger`

- If the invocation came from a host skill's heuristic-capture path (matching a cue from `sidekick-capture.md`), `--trigger heuristic`.
- If the invocation came from `/sidekick` (user-typed) or an explicit `specstudio:sidekick` invocation in any host context, `--trigger explicit`.

### Invoke the creation call and branch on exit status

With `<destination-repo-root>` resolved, invoke the default call — **no `--slug`**, so the CLI derives the slug from the one-liner:

```bash
specscore sidekick new "<one-liner>" \
  --project "<destination-repo-root>" \
  --captured-by "<captured_by>" \
  --trigger "<heuristic|explicit>"
  # --captured-during "<active-spec-path>"   # add ONLY when captured_during is not null
  # --body "<markdown>"                       # add ONLY when the host supplied --body
  # --slug "<slug>"                           # add ONLY to override: a user-requested slug, or a -N collision suffix
```

The one-liner becomes the `# <one-liner>` H1; the CLI emits the frontmatter and enforces the ≤500-char one-liner and ≤2000-char body caps (and validates any `--slug`). Do **not** run a standalone `command -v` probe, and do **not** pass `--force`. Branch on the exit status per the **Creation-class row** in [`../shared/cli-detection.md`](../shared/cli-detection.md) (which carries the per-outcome rationale): **`0`** → the CLI prints the seed path, continue to the back-link and event steps; **exit `1`** (Conflict — the slug is already taken) → resolve via [Collision disambiguation](#collision-disambiguation-req-writes-seed-artifact) (retry with `--slug <base>-N`); **`127`** → install message (`/specscore:install`), then **install-then-retry**; **exit `8`** → **upgrade-then-retry**, naming the missing `sidekick new` / `--slug`; **any other non-zero** → surface verbatim, **never** a direct-write fallback.

After a successful write, read `captured_at` and `slug` back from the written seed's frontmatter — the [back-link](#source-artifact-back-link-reqs-writes-back-link-to-source-artifact-source-artifact-path-resolution-back-link-section-format-back-link-best-effort) and [event](#event-emission-req-emits-captured-event-req-event-payload-schema) steps need them, and the CLI owns the `captured_at` timestamp. The CLI's printed path (relative to `<destination-repo-root>`) is the skill's return value.

## Source-artifact back-link (REQs `writes-back-link-to-source-artifact`, `source-artifact-path-resolution`, `back-link-section-format`, `back-link-best-effort`)

After the seed file is written but **before** emitting the event, the skill updates the source artifact's back-link section so reviewers see the generated seed alongside the source. The skill performs the back-link write only when the resolved `captured_during` path points at an existing markdown file; otherwise it skips and proceeds to event emission. Back-link write failures do not block the seed or the event.

### Resolving `captured_during` to a markdown file

Apply these rules in order:

1. If `captured_during` is `null`, skip the back-link write.
2. If the value ends in `.md` and that file exists, use it directly.
3. If the value is a directory and `<value>/README.md` exists, use that file.
4. Otherwise, treat as non-existent and skip (not an error).

Reject (skip back-link write, no error) for paths that:
- resolve outside the repo root via symlinks
- traverse into hidden directories (any path component starting with `.`)

### Locating the section

In the source artifact's markdown body:

1. Search for an existing `## Sidekick Seeds Generated` H2 heading anywhere in the file.
2. If found, append the new entry as the last bullet in that section, **in place**. Do NOT relocate the section.
3. If not found, create the section. Placement:
   - If the file contains a SpecScore footer line (begins with `*This document follows the https://specscore.md/`), place the new section immediately before that footer line.
   - Otherwise, place at end-of-file.

### Entry format

Each entry is a single bullet line:

    - [<slug>](<relative path from source artifact to seed file>) — captured <YYYY-MM-DD> by <captured_by>

- `<slug>` matches the seed's frontmatter `slug` (after any `-N` disambiguator).
- The relative path is computed from the source artifact's directory to `spec/ideas/seeds/<slug>.md`. For a source at `spec/features/foo/README.md`, the relative path is `../../ideas/seeds/<slug>.md`.
- The date is the date portion of `captured_at` (YYYY-MM-DD only, no time).
- `<captured_by>` is the verbatim frontmatter value.

Append-only: newest entry at the bottom of the section. The skill MUST NOT reorder existing entries, remove entries, or modify any content in the source artifact outside this section.

### Failure semantics

If the back-link write fails (filesystem error, parse error, concurrent modification, write permission denied on the source artifact), the skill:

1. MUST NOT roll back the seed write.
2. MUST proceed with event emission as if the back-link write had succeeded.
3. MUST report the back-link write failure to the caller as a warning, e.g., `Warning: back-link write to <source-path> failed: <error>; seed and event are recorded.`
4. MUST exit 0 (success). The seed and event are the load-bearing artifacts; the back-link is a discoverability convenience that a future `specscore spec lint --fix` rule will reconcile (deferred per the Feature's Open Questions).

## Event emission (REQ `emits-captured-event`, REQ `event-payload-schema`)

On successful write — and only on successful write — emit `sidekick-idea.captured` via the convention in [`shared/events.md`](../shared/events.md).

The event uses the standard envelope+payload structure. REQ `event-payload-schema` lists 8 conceptual fields; they map to the envelope and payload as follows:

```yaml
event: sidekick-idea.captured
version: 1
uuid: <generated>
timestamp: <captured_at>
actor:
  kind: skill | user
  id: <captured_by>          # e.g., "skill:specstudio:specify" or "user:<username>"
artifact:
  type: idea-seed
  id: <slug>
  path: <seed_path>
  revision: <git SHA at time of emission, or "uncommitted">
payload:
  slug: <slug>
  captured_during: <captured_during or null>
  trigger: <heuristic|explicit>
  content_hash: <SHA-256 lowercase hex of normalized one-liner>
```

### Computing `content_hash`

```bash
content_hash=$(printf '%s' "$ONE_LINER" \
  | python3 -c "import sys; sys.stdout.write(sys.stdin.read().strip().lower())" \
  | shasum -a 256 \
  | awk '{print $1}')
```

(Python is used to ensure full Unicode casefolding; a pure-bash `tr` is acceptable for ASCII-only one-liners.)

### Transport

Per `events.md`:

- **Default:** append the event as a single line of JSON to `.specscore/events.jsonl` at repo root.
- **Hook:** if `command -v specscore` resolves, prefer `specscore event emit <event.yaml>` (CLI). Otherwise fall back to the file append.

### Failure semantics

- If the file write succeeds but event emission fails, the skill MUST report the emission failure to the caller but MUST NOT roll back the seed file. The seed exists; the event is recoverable by re-emission, but a missing seed is not recoverable. (The lint rule will discover the seed regardless; the consilium can dedupe.)
- If the file write fails, no event is emitted. The skill exits non-zero with the write error.

## Output (success)

On success, the skill prints one short line that the host echoes verbatim (REQ `post-write-success-line`):

```
Captured: <slug> at spec/ideas/seeds/<slug>.md
```

The path is relative to `<destination-repo-root>`. The destination repo identity is already visible to the user from the pre-write confirmation UX (multi-repo workspaces) or implicit in cwd (single-repo); the success line confirms the write completed at that destination.

It returns the relative seed path as its programmatic return value. The `sidekick-idea.captured` event payload's `path` field reflects the seed's path within the resolved destination repo (per the existing [REQ:emits-captured-event](../../spec/features/sidekick-capture/README.md#req-emits-captured-event)).

## Output (error)

On any validation or write failure, the skill prints one error line per the validation table above and exits non-zero. The host should propagate this to the user in a single short acknowledgement (e.g., `Sidekick capture failed: <error>`) and continue the primary task — capture failure is never a reason to derail.

## Red Flags

These patterns indicate misuse of this skill; refuse or refactor:

- Capture invocation that includes a long-form analysis or planning content in the one-liner (the one-liner is for the *what*, not the *why*).
- Repeated invocations in the same conversation for the same one-liner (host should dedupe; see `shared/sidekick-capture.md`).
- Invocation with `--body` content longer than ~500 chars for routine captures (defeats the write-and-continue discipline).
- Invocation that produces a slug requiring `-10` or higher disambiguator (signals the host is capturing the same family of ideas repeatedly; the consilium would dedupe these later, but during heavy capture-flooding, suggest the user pause and reflect).

## References

- [`shared/cli-detection.md`](../shared/cli-detection.md) — the Required-CLI Artifact Creation policy and Creation-class exit-status response (`127` install-then-retry, `8` upgrade-then-retry, other non-zero surface) the seed-creation call follows.
- [`shared/sidekick-capture.md`](../shared/sidekick-capture.md) — when and why hosts invoke this skill.
- [`shared/destination-resolution.md`](../shared/destination-resolution.md) — the deliberation-prompt template invoked from "Destination resolution" step 2.
- [`shared/events.md`](../shared/events.md) — event envelope and emission transport.
- [`references/seed-template.md`](references/seed-template.md) — illustrative example only; `specscore sidekick new` is the authoritative source of seed structure.
- [Feature: `sidekick-capture`](../../spec/features/sidekick-capture/README.md) — the parent spec this skill implements.
- [Feature: `sidekick-capture/destination-resolution`](../../spec/features/sidekick-capture/destination-resolution/README.md) — the sub-Feature for the multi-repo destination-resolution flow.
