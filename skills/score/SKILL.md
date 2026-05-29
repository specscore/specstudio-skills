---
name: score
description: |
  Manually re-invokes the configured reviewer-gates dispatch pipeline against
  one or more SpecScore artifacts (Ideas, Features, Plans), outside the
  producer-skill exit context. Wrapper-only: carries no reviewer logic of its
  own; defers entirely to `gates.<stage>.reviewers` in `specscore.yaml`.
  Surfaces the grade + findings produced by the shared reviewer-gates layer;
  `Approved` means `grade ≥ threshold` (configurable, default `B`). Skips
  `type: human` entries (manual invocation cannot suspend for human approval).
  Supports single-artifact, multi-artifact, and recursive tree-wide invocation;
  `--against REF` controls the diff baseline supplied to AI reviewers (default
  `HEAD`). Ephemeral by default; `--save` persists a report and `--badge`
  injects an A–F badge (both opt-in).
  Triggers: "specstudio:score", "/score", "score this", "re-score this".
aliases: [score]
---

# Score

Manual surface for the [`reviewer-gates`](../../spec/features/reviewer-gates/README.md) dispatch pipeline. Implements the [Score Command Feature](../../spec/features/score-command/README.md) (Phase 1 of the [`manual-review-and-score-commands`](../../spec/ideas/manual-review-and-score-commands.md) Idea).

Producer category: **NOT a Producer.** This is a *signal skill* — it produces no canonical artifact under `spec/`. Read-only on `spec/`; writes nothing; emits no events. The Feature's `REQ: no-file-writes` and `REQ: no-new-config` are load-bearing — do not violate them under any circumstances.

## When to Use

- Mid-iteration self-review before re-running a producer skill (which would emit events and update status).
- Audit of an artifact the user did not author.
- Triage across the spec tree to find what needs attention next.
- Pre-PR sanity check on a draft the user paused on.

Do NOT invoke this skill in response to any producer-skill event (`feature.specified`, `idea.approved`, `plan.approved`, etc.). Manual invocation is the only entry point. (`REQ: invocation-triggers`.)

## Anti-Patterns

- **Carrying reviewer logic inside this skill.** Reviewers come exclusively from `gates.<stage>.reviewers` in `specscore.yaml`. No hardcoded baseline. No silent injection. No fallback. (`REQ: reuse-reviewer-gates-pipeline`.)
- **Writing any file under `spec/`, `.specscore/`, or any other repo path.** This skill is read-only. (`REQ: no-file-writes`.)
- **Emitting a SpecStudio event, or writing a file, on the default run.** The default `/score` run is ephemeral. Persistence happens only under the explicit opt-in flags `--save` (report) and `--badge` (badge injection), and manual runs propose the write and ask before applying it. (`REQ: no-file-writes`.)
- **Adding a new key to `specscore.yaml` or any new dotfile or environment variable.** All reviewer behavior is governed by the existing `gates:` block defined by `reviewer-gates`. (`REQ: no-new-config`.)
- **Suspending the user's session waiting for a `type: human` reviewer.** Skip those silently with a one-line note per artifact. (`REQ: human-reviewers-skipped`.)
- **Refusing the invocation when `gates:` is absent.** This skill is a signal skill, not a gate. Report every artifact as skipped, exit zero with the no-verdict warning. (`REQ: missing-gates-block-handling`.)

## Invocation Shape

```
specstudio:score [PATHS...] [-r|--recursive] [--against REF] [--verbose] [--yes]
```

| Argument / Flag | Semantics |
|---|---|
| `PATHS...` (positional, optional) | Files, directories, or glob patterns relative to repo root (or absolute). Empty list → defaults to `spec/`. |
| `-r`, `--recursive` | Descend into sub-artifacts beneath directory entries. Without this flag, sub-artifacts MUST NOT be included. |
| `--against REF` | Diff baseline supplied to `type: ai` reviewers. Default: `HEAD`. |
| `--verbose` | In multi-artifact mode, print per-artifact detail in addition to the summary table. No-op in single-artifact mode. |
| `--yes` | Skip the threshold confirmation prompt regardless of resolved-artifact count. Does NOT skip any other confirmation. |

Any unknown flag (anything starting with `-` that is not in the table above) MUST cause the skill to refuse to run with a usage error and exit non-zero. Do NOT silently ignore unknown flags. (`REQ: invocation-shape`.)

## Pipeline Overview

1. **Parse arguments** — fail fast on unknown flags.
2. **Resolve paths** — empty → `spec/`; directories → `README.md`; globs → shell expansion; `-r` → recursive descent; deduplicate; preserve first-appearance order.
3. **Map each artifact to a stage** — `spec/ideas/` → `ideate`; `spec/features/<slug>/README.md` → `specify`; `spec/plans/` → `plan`; anything else → skipped.
4. **Confirm-at-threshold** — if `-r` AND resolved count > 10 AND no `--yes`, prompt for explicit approval before dispatch.
5. **Per-artifact, in resolved order, serially**:
   1. Load `gates.<stage>.reviewers` via the shared loader.
   2. Filter out `type: human` entries; record a one-line skip note per artifact.
   3. Compute the unified diff `git diff <REF> -- <artifact-path>`.
   4. Dispatch surviving `type: ai` entries via the shared runner, passing the artifact body + diff to each.
   5. Collect the verdict (`Approved` | `Issues Found`).
6. **Emit output** — per-artifact detail (single-artifact) or summary table + footer (multi-artifact, with `--verbose` adding per-artifact detail).
7. **Exit** — zero if every reviewed artifact returned `Approved`, non-zero if any returned `Issues Found`, zero with no-verdict warning if every artifact was skipped.

The skill writes nothing in `spec/`; emits no events. The single user-facing side effect is the terminal output and the exit code (plus the report/badge only under `--save` / `--badge`).

> **Grade dependency.** The verdict this skill surfaces is whatever the shared reviewer-gates runner returns. The steps below describe the `Approved | Issues Found` shape that `reviewer-gates` returns today. Once `reviewer-gates` ships the grade as its verdict currency — the findings → A–F aggregation plus the configurable Approve threshold (default `B`) — this skill surfaces that grade and derives `Approved` as `grade ≥ threshold`, with no change to path resolution, dispatch, or the no-writes guarantee below.

## Step 1 — Parse Arguments

Tokenize the invocation. Distinguish positional `PATHS` from flags.

- Anything starting with `-` is a flag. The recognized flags are exactly `-r`, `--recursive`, `--against`, `--verbose`, `--yes`. Unknown flags MUST refuse the invocation. (`REQ: invocation-shape`.)
- `--against` MUST take a value (`--against REF` or `--against=REF`). A bare `--against` without a value MUST refuse.
- `-r` and `--recursive` are equivalent.
- `--verbose`, `--yes` are boolean flags.
- Everything else is a positional `PATHS` entry, in user-supplied order.

If parsing fails, print a one-line usage error and exit non-zero. Do not proceed to path resolution.

## Step 2 — Resolve Paths

The output of this step is `resolved_paths` — an ordered, deduplicated list of artifact paths, each tagged with whether it was skipped (and why) before any reviewer dispatch.

### 2a — Default empty PATHS

If `PATHS` is empty, treat it as the single entry `spec/`. (`REQ: empty-paths-default`.)

### 2b — Expand each PATHS entry, in user-supplied order

For each entry `p` in `PATHS`:

1. **If `p` contains a glob character (`*`, `?`, `**`)**: expand using shell-glob semantics against the repo working tree. If the expansion produces zero matches, refuse the entire invocation with `Error: glob pattern '<p>' matched no files in the working tree`; exit non-zero. (`REQ: glob-expansion`.) Add each expansion result (in lexicographic order) to the working list, marked as a "file" entry — globs do NOT recurse on their own.

2. **Else if `p` resolves to a directory**:
   - If `<p>/README.md` exists, add `<p>/README.md` to the working list, marked as a "directory-readme" entry.
   - If `<p>/README.md` does NOT exist, record the directory as **skipped** with reason `(no README.md in directory)` and continue with remaining entries. (`REQ: directory-resolves-to-readme`.)
   - If `-r` is supplied, ALSO add (after the README) every reviewable sub-artifact beneath `<p>` per the patterns in `artifact-to-stage-mapping` (Step 3), in lexicographic order. (`REQ: recursive-descent`.) Reviewable patterns are:
     - `<p>/ideas/*.md` (excluding `<p>/ideas/README.md`)
     - `<p>/ideas/seeds/*.md`
     - `<p>/features/*/README.md`
     - `<p>/plans/*.md` (excluding `<p>/plans/README.md`)

3. **Else if `p` resolves to a regular file**: add `p` to the working list, marked as a "file" entry. `-r` is a no-op for file entries (a file has nothing beneath to recurse into) — accept silently per the Feature's Not Doing list.

4. **Else** (`p` does not resolve): refuse the entire invocation with `Error: path '<p>' does not exist`; exit non-zero. (Globs are handled in step 1; reaching this branch means a literal path that does not resolve.)

### 2c — Deduplicate, preserving first-appearance order

Walk the working list. Build `resolved_paths` by appending each entry the first time it appears (key = absolute path of the artifact). Subsequent appearances of the same absolute path MUST be dropped. The order in `resolved_paths` is the order of FIRST appearance in the working list. (`REQ: deduplication-and-order`.)

## Step 3 — Map Each Artifact to a Stage

For each entry in `resolved_paths`, compute its stage using the canonical mapping (`REQ: artifact-to-stage-mapping`):

| Path pattern (relative to repo root) | Stage |
|---|---|
| `spec/ideas/<slug>.md` (not `spec/ideas/README.md`) | `ideate` |
| `spec/ideas/seeds/<slug>.md` (not `spec/ideas/seeds/README.md`) | `ideate` |
| `spec/features/<slug>/README.md` | `specify` |
| `spec/plans/<slug>.md` (not `spec/plans/README.md`) | `plan` |

Any other path — including index `README.md` files at `spec/README.md`, `spec/features/README.md`, `spec/ideas/README.md`, `spec/plans/README.md`, and anything under `spec/research/` — MUST be tagged **skipped** with reason `(no gate configured for this artifact type)`. Do NOT dispatch any reviewer for skipped artifacts. Continue with remaining entries.

## Step 4 — Confirm-at-Threshold

After Step 3, count the artifacts that are NOT yet tagged skipped — call this `reviewable_count`.

If ALL of the following are true:
- `-r` was supplied (`REQ: confirm-at-threshold`).
- `reviewable_count > 10`.
- `--yes` was NOT supplied (`REQ: yes-flag-skips-threshold`).

Then surface a confirmation prompt naming:
- The reviewable count (e.g., "23 artifacts to review").
- The list of distinct stages involved (e.g., "stages: ideate, specify").
- The approximate AI-reviewer dispatch count — sum of `gates.<stage>.reviewers` `type: ai` entry counts across involved stages × number of artifacts in each stage. (For loader errors, fall back to "unknown".)

Wait for the user's response. Use the same approval-phrase recognizer as `specstudio:ideate` and `specstudio:specify`:
- Explicit approval phrase (`approve` / `approved` / `accept` / `accepted` / `lgtm` plus direct semantic equivalents in the user's language) → proceed.
- Explicit decline or change request → abort the invocation with `Error: user declined the threshold confirmation`; exit non-zero.
- Vague positive signal → ask one explicit confirmation question and wait. Do NOT proceed on a vague signal alone.

If `reviewable_count <= 10`, OR `-r` was not supplied, OR `--yes` was supplied: do NOT prompt; proceed directly to Step 5.

## Step 5 — Per-Artifact Gate Dispatch (serial, in resolved order)

For each entry in `resolved_paths` in order:

### 5a — Skip-tagged artifacts

If the entry is tagged skipped (from Step 2 or Step 3), record its skip reason in the per-artifact result and continue to the next entry. Do NOT dispatch any reviewer.

### 5b — Load the gate config

Resolve the entry's stage `<S>` from Step 3. Invoke the shared loader at [`../shared/reviewer-gates/loader.md`](../shared/reviewer-gates/loader.md) with `<skill> = <S>` (e.g., `specify` for a Feature artifact). Follow that protocol verbatim.

Loader-outcome handling — diverges from `specstudio:specify` per `REQ: stage-without-gate` and `REQ: missing-gates-block-handling`:

- **Loader refuses with "missing or empty `gates.<S>.reviewers`"** (any of the three states: no `gates:`, no `gates.<S>`, empty `reviewers: []`): tag this artifact as **skipped** with reason:
  - `(no gates: block in specscore.yaml)` if there is no top-level `gates:` at all (`REQ: missing-gates-block-handling`).
  - `(no gates.<S> configured)` if `gates:` exists but `gates.<S>` does not (`REQ: stage-without-gate`).
  - `(empty gates.<S>.reviewers: [])` if the list is present but empty.
  - Continue to the next artifact. Do NOT refuse the whole invocation. The skill is a signal skill, not a gate.
- **Loader refuses for any other reason** (entry-shape violation, duplicate name, unknown type, malformed prompt path, missing taxonomy section, etc.): the configuration itself is broken. Surface the loader's error verbatim, tag this artifact as skipped with reason `(gate config error — see loader output)`, and continue. Do NOT abort the whole invocation — the user may be reviewing artifacts at multiple stages and only one is misconfigured.

### 5c — Filter `type: human` entries

From the validated reviewer list returned by the loader, partition into `ai_entries` and `human_entries`. (`REQ: human-reviewers-skipped`.)

- If `human_entries` is non-empty, record a per-artifact note: `(skipped <N> type: human entry — manual invocation cannot suspend for human approval)`. The plural is `entries` for N > 1.
- If `ai_entries` is empty (every entry was `type: human`), tag this artifact as **skipped** with reason `(every reviewer for stage <S> is type: human; manual invocation has no verdict to compute)`. Continue to the next artifact.
- Otherwise, proceed to Step 5d with `ai_entries` as the dispatch list.

### 5d — Compute the diff

Run `git diff <REF> -- <artifact-path>` where `<REF>` is the value of `--against` (default `HEAD`, per `REQ: against-default`). Capture stdout as `diff_text`.

- If git exits non-zero because the ref does not resolve (e.g., `--against does-not-exist`): record a per-artifact note `(ref '<REF>' could not be resolved — diff is empty)`. Set `diff_text` to empty string. Continue to dispatch (the AI reviewer still receives the artifact body). (`REQ: diff-context-supplied-to-ai`.)
- If git is unavailable or the repo is not a git repository: set `diff_text` to empty string with a note `(not a git working tree — diff is empty)`. Continue.

This skill MUST NOT add or read any `mode:` discriminator field on reviewer entries — that contract belongs to `reviewer-gates`. (`REQ: no-mode-discriminator`.)

### 5e — Dispatch surviving `type: ai` entries via the shared runner

Invoke the shared runner at [`../shared/reviewer-gates/runner.md`](../shared/reviewer-gates/runner.md) with the filtered `ai_entries` list. The runner enforces serial dispatch, AND-composition, and halt-after-first-`Issues Found` within a pass. (`REQ: reuse-reviewer-gates-pipeline`.)

The user-facing message passed to each `type: ai` subagent MUST be the concatenation of:

1. The artifact's absolute path (one-line preamble).
2. The artifact's current working-tree contents.
3. A horizontal rule (`---`).
4. The diff baseline label: `Diff against: <REF>` (or the empty-diff note from Step 5d).
5. `diff_text` (unified-diff format).

This composition is the manifestation of `REQ: diff-context-supplied-to-ai` — every AI reviewer receives BOTH the snapshot AND the diff. No `mode:` field selects one or the other.

Each AI subagent's system prompt is the entry's resolved `prompt_path` file contents (loaded by the runner). Wait for the verdict. The runner enforces the verdict shape (`Approved` | `Issues Found` with `Blocker` / `Advisory` severities).

Verdict composition for this artifact:
- **`Approved`** — every entry in `ai_entries` returned `Approved`.
- **`Issues Found`** — at least one entry in `ai_entries` returned `Issues Found`. The runner halts within this artifact after the first failure; subsequent entries are NOT dispatched in this pass (per [`reviewer-gates#req:and-composition`](../../spec/features/reviewer-gates/README.md#req-and-composition)).

### 5f — Record the per-artifact result

Append to the results list:

```
{
  path: <relative path>,
  stage: <S>,
  verdict: Approved | Issues Found | Skipped,
  skip_reason: <string or null>,
  ai_verdict_map: { <reviewer-name>: <Approved | Issues Found> },
  findings: [ { reviewer, severity, text }, ... ],
  notes: [ "(skipped 1 type: human entry — ...)", "(ref '...' could not be resolved — ...)", ... ],
}
```

### 5g — Serial-across-artifacts discipline

Do NOT start the next artifact's gate until the current artifact's runner has returned a verdict (or has been tagged skipped). At no point may two artifact gates be concurrently in flight. (`REQ: serial-dispatch-across-artifacts`.)

This is mandatory — not a performance preference. Parallel dispatch is out of MVP scope per the Feature's `## Not Doing`.

## Step 6 — Emit Output

Output goes to stdout (not to any file). The shape depends on `reviewable_count` and the `--verbose` flag.

### 6a — Single-artifact mode

When `resolved_paths.length == 1` (regardless of `--verbose`), print the per-artifact section (`REQ: output-mode-single-artifact`):

```
<relative path>
Stage: <S>
Verdict: <Approved | Issues Found | Skipped>

<one-line notes, one per line, if any>

<For Issues Found:>
Findings:
  Blocker — [<reviewer-name>] <finding text>
  Blocker — [<reviewer-name>] <finding text>
  Advisory — [<reviewer-name>] <finding text>

<For Skipped:>
Skipped: <skip reason>
```

Group findings by severity (`Blocker` first, then `Advisory`). Include the reviewer's `name:` per the Feature's `REQ: per-artifact-section`. Do NOT print a summary footer in single-artifact mode.

### 6b — Multi-artifact summary mode (default, no `--verbose`)

When `resolved_paths.length >= 2` and `--verbose` was NOT supplied, print a summary table and footer (`REQ: output-mode-multi-artifact`):

```
| Path                            | Stage   | Verdict       | Findings |
|---------------------------------|---------|---------------|----------|
| spec/ideas/foo.md               | ideate  | Approved      | —        |
| spec/features/bar/README.md     | specify | Issues Found  | 2B, 1A   |
| spec/research/notes.md          | —       | Skipped       | —        |

Reviewed: 2 (Approved: 1, Issues Found: 1)
Skipped: 1
```

- Path column: the relative path from the repo root.
- Stage column: the resolved stage, or `—` for skipped artifacts.
- Verdict column: `Approved`, `Issues Found`, or `Skipped`.
- Findings column: counts in form `<N>B, <M>A` (blockers and advisories), `—` if none, or `—` for skipped.
- Footer: total reviewed (artifacts that produced a verdict), broken into Approved and Issues Found, and a separate Skipped line. Skipped artifacts MUST NOT contribute to the Approved/Issues Found counts. (`REQ: output-mode-multi-artifact` + `REQ: skipped-artifacts-in-table`.)

### 6c — Multi-artifact verbose mode (`--verbose`)

When `resolved_paths.length >= 2` and `--verbose` was supplied, print the full per-artifact section (per 6a) for EVERY artifact, in resolved-paths order, AND the footer from 6b. The summary table MAY be present or omitted at implementation discretion — but the per-artifact detail is mandatory. (`REQ: output-mode-multi-artifact` + `multi-artifact-verbose-mode` AC.)

### 6d — All-skipped warning

If every artifact was tagged skipped (no verdict was computed), print:

```
Warning: no artifact produced a verdict. Nothing was reviewed. Check your paths and gate configuration.
```

This warning is mandatory whenever the reviewed count is zero — it prevents a CI integration from silently mistaking "nothing reviewed" for "everything passed". (`REQ: exit-code` + `exit-code-all-skipped` AC.)

## Step 7 — Exit Code

(`REQ: exit-code`.)

- Every reviewed (non-skipped) artifact returned `Approved` → exit `0`.
- At least one reviewed (non-skipped) artifact returned `Issues Found` → exit `1` (non-zero).
- Every artifact was skipped → exit `0` with the all-skipped warning from Step 6d.

Skipped artifacts MUST NOT affect the success/failure exit code — they affect only the all-skipped warning.

## Verification Checklist

Before declaring the implementation complete on any change to this skill, verify (against [the Score Command Feature](../../spec/features/score-command/README.md)):

- [ ] Path resolution preserves first-appearance order (`REQ: deduplication-and-order`, AC `deduplication-preserves-first-order`).
- [ ] Index `README.md` files (`spec/README.md`, `spec/features/README.md`, `spec/ideas/README.md`, `spec/plans/README.md`, `spec/ideas/seeds/README.md`) are always skipped (AC `index-readme-skipped`).
- [ ] `type: human` entries are skipped per-artifact with a one-line note (AC `human-reviewers-silently-omitted`).
- [ ] When every reviewer for a stage is `type: human`, the artifact is reported as skipped with a no-verdict note (AC `all-human-list-reports-no-verdict`).
- [ ] AI reviewers receive both the artifact body AND the diff against `--against REF` (default `HEAD`) — AC `diff-supplied-to-ai-with-default-ref`, AC `against-custom-ref-applied`.
- [ ] Invalid `--against` ref falls back to empty diff with a per-artifact note, never refuses (AC `invalid-ref-falls-back-to-empty-diff`).
- [ ] Threshold confirmation prompts at `reviewable_count > 10` AND `-r` AND no `--yes` (AC `confirm-at-threshold-prompts`); silent below 10 (AC `confirm-at-threshold-below-bound`); silent under `--yes` (AC `yes-flag-skips-confirm`).
- [ ] Artifact gates run serially (AC `serial-across-artifacts`).
- [ ] Single-artifact output is always detailed (AC `single-artifact-detailed-output`); multi-artifact default is summary mode (AC `multi-artifact-default-summary-mode`); `--verbose` adds per-artifact detail (AC `multi-artifact-verbose-mode`).
- [ ] Exit codes match the Feature contract (ACs `exit-code-success`, `exit-code-failure`, `exit-code-all-skipped`).
- [ ] No file is written under `spec/`, `.specscore/`, or anywhere else (AC `no-files-written`).
- [ ] No new key is added to `specscore.yaml`; no new dotfile; no new environment variable (AC `no-new-config-keys`).
- [ ] Missing `gates:` block is graceful — artifacts skipped, exit zero with no-verdict warning (AC `missing-gates-block-graceful`).
- [ ] Unknown flag refuses the invocation (AC `unknown-flag-refused`); unmatched glob refuses (AC `glob-unmatched-refused`).

## Red Flags

- Dispatching any reviewer not present in `gates.<stage>.reviewers`.
- Carrying a hardcoded baseline reviewer or reviewer-prompt inside this skill.
- Writing to any file under `spec/`, including the Feature's own `_tests/` directory.
- Refusing the invocation when `gates:` is missing (this skill is signal, not gate).
- Reading or writing a `mode:` field on reviewer entries.
- Running two artifact gates concurrently.
- Suspending the session waiting for a `type: human` reviewer.
- Adding a `--parallel`, `--max-concurrency`, or any other concurrency-control flag (out of MVP per `## Not Doing`).
- Computing the grade *inside this skill* — the grade + Approve threshold come from the shared reviewer-gates layer; this skill only surfaces them.
- Writing a persisted report or injecting a badge without the explicit `--save` / `--badge` flag (and, for manual runs, user approval).
- Emitting any SpecStudio event.

## References

- [Score Command Feature](../../spec/features/score-command/README.md) — the contract this skill implements; every REQ and AC traces from here.
- [Reviewer Gates Feature](../../spec/features/reviewer-gates/README.md) — the contract that owns `gates.<stage>.reviewers` schema, reviewer-entry shape, AND-composition, and rerun policy.
- [`../shared/reviewer-gates/loader.md`](../shared/reviewer-gates/loader.md) — load-and-validate protocol for `gates.<stage>.reviewers`. Invoked by Step 5b.
- [`../shared/reviewer-gates/runner.md`](../shared/reviewer-gates/runner.md) — dispatch and verdict-aggregation protocol. Invoked by Step 5e.
- [`manual-review-and-score-commands` Idea](../../spec/ideas/manual-review-and-score-commands.md) — the source Idea. One command (`/score`); the grade + Approve threshold are single-sourced in `reviewer-gates`, which this skill consumes. `--save` / `--badge` are opt-in flags on the same command.
