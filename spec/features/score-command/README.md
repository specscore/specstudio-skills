---
format: https://specscore.md/feature-specification
status: Implementing
---

# Feature: Score Command

> [SpecScore.**Studio**](https://specscore.studio): | [Explore](https://specscore.studio/app/github.com/specscore/specstudio-skills/spec/features/score-command?op=explore) | [Edit](https://specscore.studio/app/github.com/specscore/specstudio-skills/spec/features/score-command?op=edit) | [Ask question](https://specscore.studio/app/github.com/specscore/specstudio-skills/spec/features/score-command?op=ask) | [Request change](https://specscore.studio/app/github.com/specscore/specstudio-skills/spec/features/score-command?op=request-change) |

**Status:** Implementing
**Date:** 2026-05-28
**Owner:** alex
**Source Ideas:** manual-review-and-score-commands
**Supersedes:** —

## Summary

Defines `specstudio:score` — a manual, user-facing skill that re-invokes the configured [`reviewer-gates`](../reviewer-gates/README.md) dispatch pipeline against one or more SpecScore artifacts (Ideas, Features, Plans), outside the producer-skill exit context. The skill is a thin wrapper: it carries no reviewer logic of its own, defers entirely to `gates.<stage>.reviewers` in `specscore.yaml`, and surfaces per-artifact `Approved | Issues Found` verdicts plus inline findings. The default run is ephemeral — no file writes, no events. Grade output, report persistence (`--save`), and badge injection (`--badge`) are the **next increment of this same command** — there is no separate `/score` command — gated on the `reviewer-gates` grade work (the findings → A–F aggregation and the configurable Approve threshold, default `B`). When that lands, `Approved` becomes `grade ≥ threshold`.

The MVP closes a real gap: today, the only way to invoke reviewer-gates is to re-run a producer skill (`specstudio:ideate`, `specstudio:specify`, `specstudio:plan`), which has unwanted side effects (status transitions, event emission, index updates). Authors who want mid-iteration self-review, reviewers who want to audit someone else's draft, and triagers who want to sweep a tree all need a manual surface. This Feature is that surface.

## Problem

[The `reviewer-gates` Feature](../reviewer-gates/README.md) (Approved) ships the dispatch mechanism: per-stage reviewer lists scoped under `gates:` in `specscore.yaml`, AND-composed verdicts, serial dispatch, AI + human reviewer types. But its sole MVP consumer is `specstudio:specify`, which only runs the gate at its own producer-exit point. The author's day contains more review moments than that:

- **Mid-iteration self-review** before re-running the producer skill (which would emit events, update status, touch the index).
- **Audit** of an artifact the user did not author (someone else's draft on a colleague's branch).
- **Triage** across the spec tree to find what needs attention next.
- **Pre-PR sanity check** on a draft the user paused on.

None of these have a manual surface today. The mechanism exists; the trigger does not.

The reframed use case from the source Idea — *a BA or developer wants AI feedback on a draft before spending human review time on it* — is exactly the `type: ai` reviewer slot defined by `reviewer-gates`, but with no current way to invoke it manually. This Feature ships that trigger.

## Behavior

### Invocation and triggers

The skill is a user-facing manual surface, not an event consumer.

#### REQ: invocation-triggers

The skill MUST respond to the triggers `specstudio:score`, `/score`, and the natural-language phrasings `score this`, `score this artifact`, `re-score this`. It MUST NOT respond to any producer-skill event (`feature.specified`, `idea.approved`, `plan.approved`, etc.) — manual invocation is the only entry point. Event-driven re-invocation is explicitly out of MVP scope (see `## Not Doing`).

#### REQ: invocation-shape

The skill MUST accept the invocation shape `specstudio:score [PATHS...] [-r|--recursive] [--against REF] [--verbose] [--yes]`. `PATHS` is a positional list of files, directories, or glob patterns expressed relative to the repo root or as absolute paths. All flags are optional. Unknown flags MUST cause the skill to refuse to run with a usage error; the skill MUST NOT silently ignore unknown flags.

### Path resolution

`PATHS` resolves to a deterministic, ordered list of artifact paths before dispatch begins.

#### REQ: empty-paths-default

When `PATHS` is empty, the skill MUST default to the single path `spec/`. Combined with `-r`, this resolves to every reviewable artifact under `spec/` (a tree-wide audit). Without `-r`, an empty `PATHS` resolves to `spec/README.md` only (the spec-tree root index) — the same single-artifact behavior as if the user had typed `spec/` explicitly.

#### REQ: directory-resolves-to-readme

When a `PATHS` entry resolves to a directory, the skill MUST treat the directory's `README.md` (if present) as the artifact to review. When `README.md` is missing, the directory entry MUST be skipped with a one-line warning that names the directory and the reason; the skill MUST continue with remaining entries.

#### REQ: recursive-descent

When `-r` (or `--recursive`) is supplied, the skill MUST additionally include every sub-artifact reachable beneath each directory entry. Sub-artifacts are files matching the artifact-path conventions defined in `artifact-to-stage-mapping`. Without `-r`, sub-artifacts MUST NOT be included; only the directly named artifacts (and directory-`README.md` resolutions) are reviewed.

#### REQ: glob-expansion

When a `PATHS` entry contains a glob pattern (`*`, `?`, `**`), the skill MUST expand it using standard shell-glob semantics against the repo working tree. Entries that match zero files MUST cause the skill to refuse to run with an error naming the unmatched pattern; the skill MUST NOT silently drop unmatched globs.

#### REQ: deduplication-and-order

When path resolution produces duplicate entries (e.g., a file named explicitly AND reached via `-r` from its parent directory), the skill MUST deduplicate so each artifact is reviewed at most once. The resolved list MUST preserve the order of first appearance from the user's `PATHS` argument; recursive descent from a directory MUST contribute entries in lexicographic order beneath that directory.

### Artifact-to-stage mapping

The skill must decide which `gates.<stage>.reviewers` list to invoke for each artifact. The stage is determined by the artifact's path.

#### REQ: artifact-to-stage-mapping

For each resolved artifact path, the skill MUST resolve a stage name using this mapping:

| Path pattern | Stage |
|---|---|
| `spec/ideas/<slug>.md` | `ideate` |
| `spec/ideas/seeds/<slug>.md` | `ideate` |
| `spec/features/<slug>/README.md` (and recursively-discovered sub-features) | `specify` |
| `spec/plans/<slug>.md` | `plan` |

Artifacts whose path does not match any pattern in this table (e.g., `spec/research/*.md`, the root `spec/README.md`, `spec/features/README.md`, `spec/ideas/README.md`, `spec/plans/README.md`) MUST be skipped with a one-line note ("(no gate configured for this artifact type)"); the skill MUST continue with remaining entries. Index files (`README.md` files under `spec/`, `spec/features/`, `spec/ideas/`, `spec/plans/`) are explicitly index-only and never reviewable through this skill.

#### REQ: stage-without-gate

When a resolved stage has no corresponding `gates.<stage>` block in `specscore.yaml` (e.g., a project that has only configured `gates.specify` and the artifact resolves to stage `plan`), the artifact MUST be skipped with a one-line note ("(no `gates.<stage>` configured)"); the skill MUST continue with remaining entries. This is NOT a hard refusal — manual review of an unconfigured stage is a no-op for that artifact, not a fatal error for the whole invocation.

### Dispatch through reviewer-gates

The skill is a thin wrapper. It MUST NOT carry reviewer logic of its own.

#### REQ: reuse-reviewer-gates-pipeline

For each artifact whose stage maps to a non-empty `gates.<stage>.reviewers` list, the skill MUST dispatch reviewers by loading and running that list through the same loader and runner described by [`reviewer-gates`](../reviewer-gates/README.md) (the `loader.md` / `runner.md` protocols referenced by `specstudio:specify`). The skill MUST NOT instantiate its own reviewer dispatch path, MUST NOT carry a hardcoded baseline reviewer, and MUST NOT add or remove reviewer entries beyond what the loader returns.

#### REQ: human-reviewers-skipped

When the validated reviewer list for an artifact contains one or more `type: human` entries, the skill MUST skip every `type: human` entry for that artifact's gate run. The verdict aggregation for that artifact MUST be composed from the surviving `type: ai` entries alone, under the same AND-composition rule defined by `reviewer-gates`. The skill MUST surface a one-line note per artifact naming the count of skipped human entries (e.g., "(skipped 1 `type: human` entry — manual invocation cannot suspend for human approval)"). When ALL reviewer entries for an artifact's stage are `type: human` (no AI reviewers survive), the artifact MUST be reported as skipped with the same note plus an explanation that no verdict is computable; the artifact MUST NOT contribute to the aggregate footer's verdict count.

#### REQ: diff-context-supplied-to-ai

Every `type: ai` reviewer dispatched by this skill MUST receive, in addition to the artifact contents, a unified-diff representation of the artifact computed by `git diff <REF> -- <artifact-path>` where `<REF>` is the value of `--against` (default per `against-default`). When `--against` resolves to a ref that does not exist in the local repo (invalid ref, untracked artifact, deleted ref), the diff is empty; the skill MUST surface a one-line note per artifact and proceed with empty-diff dispatch (the reviewer still receives the artifact body).

#### REQ: against-default

When `--against` is not supplied, the diff baseline MUST default to `HEAD`. This means the diff supplied to each reviewer compares the working-tree contents of the artifact against the last commit — matching `git diff` with no arguments. The Idea's preferred mental model is "feedback on what I'm about to commit."

#### REQ: no-mode-discriminator

This Feature MUST NOT introduce a `mode:` field (or any other discriminator) on `gates.<stage>.reviewers` entries to distinguish diff-aware from snapshot prompts. Existing reviewer prompts are reused as-is. Per the source Idea: any change to the reviewer-entry shape — including adding a `mode:` discriminator — is owned by [`reviewer-gates`](../reviewer-gates/README.md), not by this skill.

#### REQ: verdict-parity-with-producer-exit

For any single artifact, the verdict returned by `specstudio:score <path>` MUST equal the verdict the producer-exit gate would return for the same artifact at the same `HEAD`, holding `gates.<stage>.reviewers` constant and excluding `type: human` entries (which are skipped per `human-reviewers-skipped`). Drift between the manual path and the producer-exit path is a contract violation, not an acceptable optimization. (See the source Idea's must-be-true assumption on this point.)

### Recursion safety

A tree-wide invocation can dispatch dozens of LLM calls. The skill ships with a cost-safety mechanism.

#### REQ: confirm-at-threshold

When `-r` is supplied AND the resolved-artifact count exceeds 10, the skill MUST surface a confirmation prompt naming the count, the list of stages involved, and the approximate AI-reviewer dispatch count (sum of `type: ai` entries across involved stages), then wait for explicit user confirmation before dispatching any reviewer. The skill MUST NOT proceed on a vague positive signal — only an explicit approval phrase (per the same recognizer used by `specstudio:specify` and `specstudio:ideate`) or the `--yes` flag suppresses the prompt.

#### REQ: yes-flag-skips-threshold

When `--yes` is supplied, the threshold confirmation prompt MUST be skipped regardless of resolved-artifact count. The flag MUST NOT change any other behavior (it is not a global "skip all confirmations" flag — it only governs the `confirm-at-threshold` prompt).

#### REQ: serial-dispatch-across-artifacts

The skill MUST dispatch artifacts serially in the resolved-list order computed by `deduplication-and-order`. Within a single artifact's gate run, reviewer dispatch is already serial per `reviewer-gates#dispatch-serial`. Parallel dispatch across artifacts is out of MVP scope (see `## Not Doing`). The next artifact's gate MUST NOT begin until the previous artifact's gate has resolved.

### Output

The skill produces inline terminal output. No files are written; no events are emitted.

#### REQ: per-artifact-section

For each artifact reviewed (i.e., not skipped per `artifact-to-stage-mapping`, `stage-without-gate`, or `human-reviewers-skipped`'s all-human edge case), the skill MUST print a per-artifact section containing: (a) the artifact's resolved path, (b) the resolved stage name, (c) the verdict (`Approved` or `Issues Found`), (d) for `Issues Found`, the structured findings list grouped by severity (`Blocker` first, then `Advisory`), each finding identifying the reviewer's `name:` and the finding text, (e) any one-line notes from `human-reviewers-skipped` or `diff-context-supplied-to-ai`.

#### REQ: output-mode-single-artifact

When the resolved-artifact count is exactly 1, the skill MUST print the per-artifact section in full (as defined by `per-artifact-section`) without a summary footer. No `--verbose` flag effect applies — single-artifact mode is always detailed.

#### REQ: output-mode-multi-artifact

When the resolved-artifact count is ≥ 2, the skill MUST default to a compact mode: a one-line-per-artifact summary table (path, stage, verdict, finding count), followed by a footer summarizing the total count, the `Approved` count, and the `Issues Found` count. When `--verbose` is supplied, the skill MUST print the full per-artifact section (as in single-artifact mode) for every artifact, in addition to the footer. Skipped artifacts (per `artifact-to-stage-mapping`, `stage-without-gate`, all-human edge case, or zero-glob-match) MUST appear in the table with verdict `Skipped` and the skip reason; they MUST NOT contribute to the `Approved` or `Issues Found` counts in the footer.

#### REQ: exit-code

The skill MUST exit zero when every reviewed artifact (i.e., not skipped) returned `Approved`. It MUST exit non-zero when any reviewed artifact returned `Issues Found`. Skipped artifacts MUST NOT affect the exit code. An invocation in which every artifact was skipped (zero artifacts contributed a verdict) MUST exit zero with a warning that no artifact produced a verdict.

#### REQ: no-file-writes

The skill MUST NOT write or modify any file in `spec/`, `.specscore/`, or any other repo path. It MUST NOT emit any SpecStudio event. Persistence and event emission are deferred to the grade increment of this Feature (gated on the `reviewer-gates` grade work) — not to a separate command. This REQ governs the verdict-only contract specified here; the `--save` / `--badge` flags that relax it are part of that later increment.

### Configuration boundary

This Feature defines no new configuration. It consumes only existing config defined by `reviewer-gates`.

#### REQ: no-new-config

The skill MUST NOT introduce any new top-level key in `specscore.yaml`, any new dotfile, or any new environment variable. All reviewer behavior is governed by the existing `gates:` block defined by [`reviewer-gates`](../reviewer-gates/README.md). Per-stage configuration changes (e.g., changing the reviewer list, swapping the AI prompt, adding/removing reviewers) MUST be made by editing `gates:` in `specscore.yaml` — never inside this skill's logic.

#### REQ: missing-gates-block-handling

When `specscore.yaml` has no top-level `gates:` key at all (the scenario `reviewer-gates#missing-gates-block-refuses` covers for `specstudio:specify`), this skill MUST report every resolved artifact as skipped with a single shared one-line note ("(no `gates:` block in `specscore.yaml`)") and exit zero with the no-verdict warning per `exit-code`. The skill MUST NOT refuse to run outright (in contrast to `specstudio:specify`), because manual review is not a gating skill — it is a signal skill, and the appropriate signal for an unconfigured repo is "nothing to review," not "refusal."

## Architecture

- **Producer category.** The skill is NOT a Producer — it produces no canonical artifact under `spec/`. It is a *signal skill*, mirroring the read-only character of `specscore feature list` / `specscore idea list`. The reviewer-gates Producer/Reviewer/Capability taxonomy classifies this skill as a Capability (consumes Reviewer dispatch; writes nothing).
- **Reviewer dispatch.** The skill never instantiates reviewer dispatch directly. It reuses the loader + runner protocols defined by [`reviewer-gates`](../reviewer-gates/README.md) (`shared/reviewer-gates/loader.md`, `shared/reviewer-gates/runner.md`) — the same code path `specstudio:specify` uses. This is load-bearing for `verdict-parity-with-producer-exit`.
- **No persistence layer (verdict-only contract).** Output is inline terminal text only. The grade increment of this Feature (gated on `reviewer-gates`) will introduce `--save` persistence at `spec/_score/<artifact-slug>-<sha>.md`; the contract specified here deliberately defers that.
- **No event emission.** The skill subscribes to no events and emits none. The reframed manual-review use case is "I, the user, am invoking this right now" — there is no asynchronous trigger to wire.
- **Skill implementation lives at.** `skills/score/SKILL.md` (parallels `skills/specify/SKILL.md`). The Feature artifact defines the contract; the skill file is its implementation.

## Interaction with Other Features

| Feature | Relationship |
|---|---|
| [Reviewer Gates](../reviewer-gates/README.md) | This Feature is the second consumer of `gates.<stage>.reviewers` (after `specstudio:specify`). It reuses the loader/runner protocols verbatim. Any change to the reviewer-entry shape — including a future `mode:` discriminator for diff-aware vs snapshot prompts — is owned by `reviewer-gates`, not by this Feature. |
| [Specify Skill](../skills/specify/README.md) | Co-consumer of `gates.specify.reviewers`. `verdict-parity-with-producer-exit` requires that, holding the reviewer list constant, both skills return the same verdict for the same artifact at the same `HEAD` (excluding `type: human` entries, which the manual skill skips). |
| [Archived Review Feature](../skills/review/README.md) | The old `review` pipeline-step Feature (archived per `reviewer-gates#review-feature-archival`) keeps the `specstudio:review` trigger and the `skills/review/` path. Naming this manual command `/score` (skill at `skills/score/`) **resolves** the naming collision the source Idea flagged — the two no longer share a trigger or directory. |
| Grade increment (this Feature) | Grade output, report persistence (`--save`), badge injection (`--badge`), and aggregate output are the next increment of **this** command — not a separate Feature or command. They are gated on the `reviewer-gates` grade work (findings → A–F aggregation + the configurable Approve threshold, default `B`). The verdict-only contract specified here MUST be implementable and useful on its own before that increment lands. |

## Acceptance Criteria

ACs are grouped here with explicit REQ back-references, mirroring sibling Features' style.

### AC: invocation-triggers-respond (verifies REQ:invocation-triggers)

**Given** the skill is installed and a `specstudio:score` command is registered in the platform's skill registry,
**When** the user issues any of the strings `specstudio:score`, `/score`, `score this`, or `re-score this` against an artifact path,
**Then** the skill MUST be invoked and MUST proceed to path resolution; the skill MUST NOT be invoked by any event payload (`feature.specified`, `idea.approved`, `plan.approved`).

### AC: unknown-flag-refused (verifies REQ:invocation-shape)

**Given** the user invokes `specstudio:score spec/ideas/some-idea.md --not-a-real-flag`,
**When** the skill parses arguments,
**Then** the skill MUST refuse to run, print a usage error naming the unknown flag, exit non-zero, and MUST NOT dispatch any reviewer.

### AC: empty-paths-defaults-to-spec (verifies REQ:empty-paths-default)

**Given** the user invokes `specstudio:score` with no `PATHS` argument and without `-r`,
**When** the skill resolves paths,
**Then** the resolved artifact list MUST be exactly `[spec/README.md]` (a single artifact, the spec-tree root index — which will be reported as skipped per `artifact-to-stage-mapping`'s index-files clause).

### AC: empty-paths-recursive-walks-tree (verifies REQ:empty-paths-default, REQ:recursive-descent)

**Given** the user invokes `specstudio:score -r` with no `PATHS` argument in a repo containing `spec/ideas/foo.md`, `spec/features/bar/README.md`, and `spec/plans/baz.md`,
**When** the skill resolves paths,
**Then** the resolved-artifact list MUST contain `spec/ideas/foo.md`, `spec/features/bar/README.md`, and `spec/plans/baz.md` (in lexicographic order beneath `spec/`), and MUST NOT contain any of the index `README.md` files under `spec/`.

### AC: directory-uses-readme (verifies REQ:directory-resolves-to-readme)

**Given** the user invokes `specstudio:score spec/features/bar` and the directory `spec/features/bar/` contains a `README.md`,
**When** the skill resolves paths,
**Then** the resolved-artifact list MUST contain `spec/features/bar/README.md` exactly once and MUST NOT contain any other file from inside that directory (because `-r` was not supplied).

### AC: directory-without-readme-skipped (verifies REQ:directory-resolves-to-readme)

**Given** the user invokes `specstudio:score spec/features/empty-dir` where the directory exists but contains no `README.md`,
**When** the skill resolves paths,
**Then** the directory entry MUST be skipped with a one-line warning naming the directory, the skill MUST continue with remaining `PATHS` entries (if any), and MUST exit per `exit-code` based on whatever other entries produced verdicts.

### AC: glob-unmatched-refused (verifies REQ:glob-expansion)

**Given** the user invokes `specstudio:score 'spec/features/nonexistent-*/README.md'` and zero files match the pattern,
**When** the skill resolves paths,
**Then** the skill MUST refuse to run, print an error naming the unmatched pattern, exit non-zero, and MUST NOT dispatch any reviewer.

### AC: deduplication-preserves-first-order (verifies REQ:deduplication-and-order)

**Given** the user invokes `specstudio:score spec/features/bar/README.md spec/features/ -r` and `spec/features/bar/README.md` is reachable both as the explicit first entry and via recursive descent from `spec/features/`,
**When** the skill resolves paths,
**Then** the resolved-artifact list MUST contain `spec/features/bar/README.md` exactly once, and MUST list it at its first-appearance position (before any other entries that come only from the recursive descent of `spec/features/`).

### AC: stage-mapping-resolves-correctly (verifies REQ:artifact-to-stage-mapping)

**Given** the user invokes `specstudio:score spec/ideas/foo.md spec/features/bar/README.md spec/plans/baz.md spec/research/notes.md`,
**When** the skill resolves stages,
**Then** the resolved stage for each artifact MUST be `ideate`, `specify`, `plan`, and `<skipped: no gate configured for this artifact type>` respectively; `spec/research/notes.md` MUST appear in the output as skipped with that note.

### AC: index-readme-skipped (verifies REQ:artifact-to-stage-mapping)

**Given** the user invokes `specstudio:score spec/features/README.md`,
**When** the skill resolves the stage,
**Then** the artifact MUST be reported as skipped with the note `(no gate configured for this artifact type)`, MUST NOT dispatch any reviewer for that artifact, and MUST contribute to the no-verdict-warning logic per `exit-code`.

### AC: stage-without-gate-skipped (verifies REQ:stage-without-gate)

**Given** `specscore.yaml` configures `gates.specify` but does NOT configure `gates.ideate`, and the user invokes `specstudio:score spec/ideas/foo.md`,
**When** the skill loads the gate for stage `ideate`,
**Then** the artifact MUST be reported as skipped with the note `(no gates.ideate configured)`, MUST NOT dispatch any reviewer, MUST NOT refuse the overall invocation, and the skill MUST exit zero per `exit-code` (no verdict produced).

### AC: human-reviewers-silently-omitted (verifies REQ:human-reviewers-skipped)

**Given** `gates.specify.reviewers` contains one `type: ai` entry followed by one `type: human` entry, and the user invokes `specstudio:score spec/features/bar/README.md`,
**When** the skill dispatches reviewers for that artifact,
**Then** only the `type: ai` entry MUST be dispatched, the artifact's verdict MUST be composed from the AI entry's verdict alone, the per-artifact section MUST contain a one-line note `(skipped 1 type: human entry — manual invocation cannot suspend for human approval)`, and the skill MUST NOT prompt the user for approval as part of this gate.

### AC: all-human-list-reports-no-verdict (verifies REQ:human-reviewers-skipped)

**Given** `gates.ideate.reviewers` contains only `type: human` entries (no AI reviewers), and the user invokes `specstudio:score spec/ideas/foo.md`,
**When** the skill dispatches reviewers for that artifact,
**Then** the artifact MUST be reported as skipped with a note indicating that no AI reviewer is configured for this stage; the artifact MUST NOT contribute to the `Approved` or `Issues Found` footer counts; and the skill's exit code MUST follow `exit-code`'s no-verdict-warning rule if this is the only artifact.

### AC: diff-supplied-to-ai-with-default-ref (verifies REQ:diff-context-supplied-to-ai, REQ:against-default)

**Given** the user invokes `specstudio:score spec/features/bar/README.md` without `--against`, and the artifact has uncommitted working-tree edits relative to `HEAD`,
**When** the skill dispatches the `type: ai` reviewers for that artifact,
**Then** each AI reviewer's input MUST include both the working-tree contents of the artifact AND the unified-diff output of `git diff HEAD -- spec/features/bar/README.md`, and the diff portion MUST be non-empty (matching the uncommitted edits).

### AC: against-custom-ref-applied (verifies REQ:diff-context-supplied-to-ai)

**Given** the user invokes `specstudio:score spec/features/bar/README.md --against main`,
**When** the skill dispatches the `type: ai` reviewers for that artifact,
**Then** each AI reviewer's input MUST include the unified-diff output of `git diff main -- spec/features/bar/README.md`, NOT the diff against `HEAD`.

### AC: invalid-ref-falls-back-to-empty-diff (verifies REQ:diff-context-supplied-to-ai)

**Given** the user invokes `specstudio:score spec/features/bar/README.md --against does-not-exist-ref`,
**When** the skill attempts to compute the diff,
**Then** the skill MUST surface a one-line note in the per-artifact section indicating the ref could not be resolved, MUST still dispatch every `type: ai` reviewer for that artifact with the artifact body and an empty diff, and MUST NOT refuse the invocation.

### AC: verdict-parity-with-specify (verifies REQ:verdict-parity-with-producer-exit, REQ:reuse-reviewer-gates-pipeline)

**Given** `gates.specify.reviewers` contains two `type: ai` entries and one `type: human` entry, the artifact `spec/features/bar/README.md` exists at a specific `HEAD`, and the user runs `specstudio:specify` to produce the producer-exit verdict (composed from both AI entries plus the human),
**When** the user immediately afterward runs `specstudio:score spec/features/bar/README.md` against the same `HEAD`,
**Then** for the AI portion of the verdict, both runs MUST produce identical results — the same per-reviewer verdicts in the same order, the same set of findings — and the manual run MUST additionally report the human entry as skipped per `human-reviewers-skipped`.

### AC: confirm-at-threshold-prompts (verifies REQ:confirm-at-threshold)

**Given** the resolved-artifact count after `specstudio:score spec/ -r` is 11, no `--yes` flag was supplied, and the user has not yet responded,
**When** the skill completes path resolution,
**Then** the skill MUST display a confirmation prompt naming the resolved count (11), the involved stages, and the approximate AI-reviewer dispatch count; the skill MUST NOT dispatch any reviewer until the user responds with an explicit approval phrase; a vague positive signal alone MUST NOT proceed.

### AC: confirm-at-threshold-below-bound (verifies REQ:confirm-at-threshold)

**Given** the resolved-artifact count after `specstudio:score spec/features/ -r` is 10 and no `--yes` flag was supplied,
**When** the skill completes path resolution,
**Then** the skill MUST NOT display a confirmation prompt and MUST proceed directly to dispatch.

### AC: yes-flag-skips-confirm (verifies REQ:yes-flag-skips-threshold)

**Given** the resolved-artifact count after `specstudio:score spec/ -r --yes` is 50,
**When** the skill completes path resolution,
**Then** the skill MUST NOT display the threshold confirmation prompt and MUST proceed directly to dispatch.

### AC: serial-across-artifacts (verifies REQ:serial-dispatch-across-artifacts)

**Given** three artifacts in the resolved list and instrumentation that records dispatch start/end timestamps per artifact gate,
**When** `specstudio:score` runs through them,
**Then** at no point during the run are two artifact gates concurrently in flight, and the recorded gate-start order matches the resolved-list order exactly.

### AC: single-artifact-detailed-output (verifies REQ:output-mode-single-artifact)

**Given** the user invokes `specstudio:score spec/features/bar/README.md` (resolved count exactly 1) without `--verbose`,
**When** the skill prints output,
**Then** the output MUST be exactly the per-artifact section as defined by `per-artifact-section`, MUST NOT contain a summary footer or summary table, and the `--verbose` flag (if also supplied) MUST NOT change anything about the output.

### AC: multi-artifact-default-summary-mode (verifies REQ:output-mode-multi-artifact)

**Given** the resolved-artifact count is 3 and `--verbose` was NOT supplied,
**When** the skill prints output,
**Then** the output MUST consist of (a) a 3-row summary table (one row per artifact with path, stage, verdict, finding count) followed by (b) a footer showing total/approved/issues-found counts; per-artifact detailed sections MUST NOT be present.

### AC: multi-artifact-verbose-mode (verifies REQ:output-mode-multi-artifact)

**Given** the resolved-artifact count is 3 and `--verbose` was supplied,
**When** the skill prints output,
**Then** the output MUST contain the full per-artifact section (per `per-artifact-section`) for every artifact AND the footer; the summary table MAY be present or omitted at implementation discretion (but the per-artifact detail is mandatory).

### AC: skipped-artifacts-in-table (verifies REQ:output-mode-multi-artifact)

**Given** the resolved-artifact count is 3 with one artifact skipped per `artifact-to-stage-mapping` (e.g., a `spec/research/*.md` file), one returning `Approved`, and one returning `Issues Found`,
**When** the skill prints the default summary table,
**Then** the table MUST contain all three rows including the skipped one with verdict `Skipped` and the skip reason, the footer MUST show `Approved: 1`, `Issues Found: 1`, the skipped artifact MUST NOT be counted in either bucket, and the total row MUST clearly distinguish reviewed (2) from skipped (1).

### AC: exit-code-success (verifies REQ:exit-code)

**Given** every reviewed (non-skipped) artifact in the resolved-artifact list returned `Approved`,
**When** the skill completes,
**Then** the skill MUST exit zero.

### AC: exit-code-failure (verifies REQ:exit-code)

**Given** at least one reviewed (non-skipped) artifact in the resolved-artifact list returned `Issues Found`,
**When** the skill completes,
**Then** the skill MUST exit non-zero.

### AC: exit-code-all-skipped (verifies REQ:exit-code)

**Given** every artifact in the resolved-artifact list was skipped (no verdict was produced),
**When** the skill completes,
**Then** the skill MUST exit zero AND MUST print a warning that no artifact produced a verdict (so a CI integration cannot silently mistake "nothing reviewed" for "everything passed").

### AC: no-files-written (verifies REQ:no-file-writes)

**Given** any invocation of `specstudio:score` with any combination of arguments,
**When** the skill completes (whether by success, failure, or refusal),
**Then** no file under `spec/`, `.specscore/`, or any other repo path MUST have been created or modified by the skill, and no SpecStudio event MUST have been emitted.

### AC: no-new-config-keys (verifies REQ:no-new-config)

**Given** the Feature is implemented,
**When** a downstream consumer inspects the canonical SpecScore Repo Config schema and the skill's documentation,
**Then** no new top-level key MUST have been added to `specscore.yaml`, no new dotfile convention MUST have been introduced, and no new environment variable MUST have been required by the skill.

### AC: missing-gates-block-graceful (verifies REQ:missing-gates-block-handling)

**Given** `specscore.yaml` has no top-level `gates:` key at all and the user invokes `specstudio:score spec/features/bar/README.md`,
**When** the skill attempts to resolve the gate,
**Then** the artifact MUST be reported as skipped with the note `(no gates: block in specscore.yaml)`, the skill MUST exit zero with the no-verdict warning per `exit-code-all-skipped`, and MUST NOT refuse the invocation outright (in contrast to `specstudio:specify`'s `missing-gates-block-refuses`).

## Rehearse Integration

Every AC above is testable via filesystem fixtures (mock `specscore.yaml`, mock reviewer prompts, mock subagent verdicts, controlled `git` working-tree state) plus skill-instrumentation (resolved-path list, dispatch order, exit code, captured stdout). Verdict-parity (`AC: verdict-parity-with-specify`) is testable by recording both `specstudio:specify`'s and `specstudio:score`'s reviewer-input payloads and asserting equality on the AI portion; this requires capturing the loader/runner's intermediate state via the same hooks `reviewer-gates`'s Rehearse stubs use.

The non-testable cases (the actual judgment quality of `type: ai` reviewers on a real artifact, the user's real-world ergonomic experience of the confirmation prompt) are validated at the assumption-validation layer of the source Idea, not as Rehearse scenarios.

Rehearse stubs for each AC are scaffolded at `_tests/<ac-slug>.md` with `**Status:** pending`; authoring scenario steps follows the implementation plan.

## Not Doing

Inherited from the source Idea's MVP scope and Not-Doing list, plus spec-level cuts:

- **Grade output, `--save`, `--badge`, aggregate output** — the next increment of this same command (not a separate command), gated on the `reviewer-gates` grade work: the findings → A–F aggregation, the configurable Approve threshold (default `B`), the report-file shape at `spec/_score/<artifact-slug>-<sha>.md`, and badge rendering. Not blocking the verdict-only contract specified here.
- **Badge injection into artifacts** (`--badge`) — part of the grade increment; depends on the `reviewer-gates` grade. Includes per-artifact badges, per-sub-artifact badges with `-r`, and root-index badges in `spec/README.md`.
- **Persistence of `/score` output** — no `spec/_review/...` files in MVP. Output is inline terminal text only. Persistence and event emission are explicit `/score` concerns.
- **Event emission** — the skill emits no event in the verdict-only contract. The grade increment may emit `score.completed`; that decision is deferred to it.
- **Parallel reviewer dispatch across artifacts** — serial mirrors `reviewer-gates#dispatch-serial`. A `--parallel` or `--max-concurrency` flag is deferred until dogfood signal on typical artifact counts and token-burst behavior arrives.
- **`specscore review` CLI verb** — skills are the shipping surface; CLI parity is a likely follow-up Feature, not part of MVP.
- **Cross-artifact reviewers** (e.g., "does this Plan's task citations match its Feature's ACs") — a separate Feature in the reviewer-gates Idea family; out of scope for the manual command surface.
- **Auto-running `/score` on every `specscore spec lint`** — performance and noise risk per the source Idea. The skill is signal, not gate; users invoke it explicitly.
- **`mode:` discriminator on `type: ai` reviewer entries** to distinguish diff-aware from snapshot prompts — owned by [`reviewer-gates`](../reviewer-gates/README.md), not by this skill. If a future reviewer prompt needs the discriminator, the change lands there as an additive revision and this skill picks it up via the existing loader.
- **Manual human-reviewer invocation outside producer flow** — `type: human` entries are silently skipped (with a per-artifact one-line note). Suspending the user's session waiting on the user is incoherent. Handing a draft to a real human reviewer is a hand-off (PR review, Slack, etc.), not a skill invocation.
- **Score-based CI gating** — `specstudio:verify` and `specstudio:recap` already serve as gates. The manual-review verdict is triage signal; not a CI gate.
- **Reviewer-prompt edits inside reviewer-gates entries** — existing prompts are reused as-is. Any prompt change is a separate edit to the prompt file; this skill makes no copy-changes there.
- **`-r` flag on path patterns that resolve to a single file** — `-r` is a no-op for single-file `PATHS` entries (there is nothing beneath a file to recurse into). The skill accepts the combination silently rather than refusing; refusal would be user-hostile.

## Open Questions

- **Threshold value (10) for `confirm-at-threshold`** — chosen as a low-friction default suitable for "small repo" use. Once Phase 1 is dogfooded on real repos (e.g., this `specstudio-skills` repo, `specscore-cli`), the value MAY be revised in a follow-on edit to this Feature based on observed wall-clock time and dispatch cost.
- **Approximate AI-reviewer dispatch count surfaced in the threshold prompt** — surfaced as a number per `confirm-at-threshold`. The exact wording of the prompt is implementation discretion.
- **Behavior when an artifact lints dirty at the time of `/score`** — the skill currently dispatches reviewers without first running `specscore spec lint`. Lint failures will surface inside reviewer output (the AI reviewer is expected to call them out), but a lint-clean precondition is not enforced. Whether to short-circuit on lint failure (refuse to dispatch reviewers on a lint-dirty artifact) is deferred to dogfood signal.
- **Surfacing of skipped artifacts in single-artifact mode** — when the user invokes `specstudio:score` on exactly one artifact and that artifact is skipped (e.g., it's an index file or `spec/research/notes.md`), the output is the per-artifact section with verdict `Skipped`. Whether the skill should additionally print a recommendation ("this artifact has no gate — did you mean a different path?") is left to implementation polish.

---
*This document follows the https://specscore.md/feature-specification*
