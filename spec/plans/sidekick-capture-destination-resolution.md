# Plan: Sidekick Capture — Destination Resolution

**Status:** Approved
**Source Feature:** sidekick-capture/destination-resolution
**Date:** 2026-05-21
**Owner:** alexandertrakhimenok
**Supersedes:** —

## Summary

Implements [`sidekick-capture/destination-resolution`](../features/sidekick-capture/destination-resolution/README.md) — the sidekick-side half of the multi-repo Idea-destination resolution two-Feature change. Decomposes the Feature's 24 acceptance criteria into nine ordered tasks: detection + helper foundation, the sidekick pre-write hook with parsing + retry, the two inline-confirmation UX flows + override handling + post-write success line, then the new `specstudio:relocate-idea` thin-shell skill and the opt-in mismatch log.

## Approach

Linear task ordering matches the runtime flow: discover candidates and the helper file → invoke the helper from sidekick's pre-write step → parse the agent's response → display the confirmation prompt → accept user input → write the seed at the resolved destination → ship the recovery skill that wraps the CLI verb. All 24 source-Feature ACs are covered by at least one task; no ACs are deferred.

The sidekick-side work (Tasks 1-7) is independent of the CLI verb in `specscore-cli`. The `specstudio:relocate-idea` skill (Tasks 8-9) **depends on `specscore idea relocate` being available** — per the source Idea's cross-repo sequencing, the CLI verb (specified in `specscore-cli`'s [`cli/idea/relocate`](https://github.com/specscore/specscore-cli/blob/main/spec/features/cli/idea/relocate/README.md) Feature, planned in [`idea-relocate-implementation`](https://github.com/specscore/specscore-cli/blob/main/spec/plans/idea-relocate-implementation.md), commit `98b99b0`) lands first; Tasks 8-9 here consume it.

## Tasks

### Task 1: Multi-repo workspace detection + deliberation-prompt helper file

**Verifies:** sidekick-capture/destination-resolution#ac:sibling-scan-discovers-candidates, sidekick-capture/destination-resolution#ac:single-repo-bypass-writes-to-cwd, sidekick-capture/destination-resolution#ac:helper-file-exists-and-conforms, sidekick-capture/destination-resolution#ac:helper-prompt-iteration-no-feature-revision

Create `skills/shared/destination-resolution.md` containing the four required contract sections (deliberation instruction, candidate-identity contract, output-format contract, UNCERTAIN escape clause). Implement the sibling-dir scan in the sidekick skill: walk `../` for siblings with `specscore.yaml`, skip hidden dirs and symlinks-out, complete in <500ms for ≤20 siblings. The scan returns the source project + matched siblings as candidates. When zero siblings match, the sidekick proceeds to its existing write path without surfacing any new UX.

### Task 2: Sidekick invokes helper + parses agent response

**Verifies:** sidekick-capture/destination-resolution#ac:invokes-helper-on-multi-repo, sidekick-capture/destination-resolution#ac:parses-well-formed-response, sidekick-capture/destination-resolution#ac:parses-rejects-unknown-repo, sidekick-capture/destination-resolution#ac:parses-rejects-over-length

When the scan returns ≥1 sibling, sidekick MUST invoke the helper before any seed-file write. Construct the prompt body by substituting actual candidate identity signals (`project.repo` + top-level `spec/features/*/` dir names) into the helper template. Parse the host agent's response: enforce exactly one line, ≤120 characters, shape `<repo>; <reason>` where `<repo>` matches (case-insensitive, trimmed) exactly one candidate's `project.repo`. Unknown repo, over-length, or shape deviations all route to the malformed handler (Task 3).

### Task 3: Retry + uncertain handling

**Verifies:** sidekick-capture/destination-resolution#ac:malformed-retry-then-ask-fallback, sidekick-capture/destination-resolution#ac:uncertain-token-triggers-retry

On malformed or `UNCERTAIN` (case-sensitive literal) response, re-invoke the helper exactly once with the corrective instruction `your previous response was unparseable; reply EXACTLY in <repo>; <reason> ≤120 chars, where <repo> is one of: <candidate-list>`. On well-formed retry, route to the pick-with-reason flow (Task 4). On a second malformed or `UNCERTAIN` response, route to the ask-without-pre-fill flow (Task 5). No further retries.

### Task 4: Inline confirmation — pick-with-reason flow

**Verifies:** sidekick-capture/destination-resolution#ac:shows-pick-with-reason-form, sidekick-capture/destination-resolution#ac:enter-routes-to-agent-pick

Display the literal prompt `Routing to <repo> because <reason> — press enter to accept, type other to override.` in the host conversation. Whitespace and punctuation must match the spec verbatim. On empty input (whitespace-only, including bare enter), route to the agent's picked candidate. The override path is implemented in Task 6 and applies to both this flow and the ask-without-pre-fill flow (Task 5).

### Task 5: Inline confirmation — ask-without-pre-fill flow

**Verifies:** sidekick-capture/destination-resolution#ac:numeric-input-routes-by-position-in-ask-flow, sidekick-capture/destination-resolution#ac:enter-in-ask-flow-aborts

Display the candidate list `1..N` (one repo per line, showing each candidate's `project.repo`) followed by the line `Type a number, a repo slug, or a path to override — or press enter to abort the capture.`. On numeric input `1..N` matching a candidate's list position, route to that candidate. On empty input (whitespace-only), abort the capture entirely: no seed file is written, no `sidekick-idea.captured` event is emitted, and a single short abort-confirmation line is printed to the host conversation. The override path (numeric out of range, slug, path) is implemented in Task 6.

### Task 6: Override input handling (cross-flow)

**Verifies:** sidekick-capture/destination-resolution#ac:override-slug-resolves-via-scan, sidekick-capture/destination-resolution#ac:override-path-bypasses-scan, sidekick-capture/destination-resolution#ac:override-invalid-reprompts

On user input that is non-empty and not a list-position number (works in both confirmation flows), interpret the input per the same form rules as the [`cli/idea/relocate --to-repo`](https://github.com/specscore/specscore-cli/blob/main/spec/features/cli/idea/relocate/README.md#req-target-repo-resolution) flag: value containing no `/` is a repo slug resolved via the in-process sibling-dir scan (single-match wins; zero or multiple matches re-prompts); value containing `/` is a relative or absolute path that must point at a directory containing a `specscore.yaml`. On any resolution failure, display a clear error message and re-display the same confirmation prompt for another attempt — no seed file is written on invalid override.

### Task 7: Post-write success line

**Verifies:** sidekick-capture/destination-resolution#ac:post-write-success-line

After the seed write completes at the resolved destination (per Tasks 4, 5, or 6), emit the standard sidekick success line `Captured: <slug> at <path-relative-to-resolved-repo>` in the host conversation. The existing `sidekick-idea.captured` event continues to fire per its existing REQ in [sidekick-capture](../features/sidekick-capture/README.md#req-emits-captured-event); the event payload's `path` field now reflects the resolved destination repo's path rather than always-cwd.

### Task 8: `specstudio:relocate-idea` skill — scaffold + CLI shell-out + output surfacing

**Verifies:** sidekick-capture/destination-resolution#ac:relocate-skill-file-exists, sidekick-capture/destination-resolution#ac:relocate-skill-trigger-without-args-prompts, sidekick-capture/destination-resolution#ac:relocate-skill-shells-out-to-cli, sidekick-capture/destination-resolution#ac:relocate-skill-surfaces-cli-output

Author a new skill file at `skills/relocate-idea/SKILL.md` with standard frontmatter (name, description, optional aliases) and a ≤200-line body. Register the triggers `specstudio:relocate-idea`, `/relocate-idea`, `relocate this idea`, `move this seed to another repo`. On invocation missing the slug or `--to-repo` argument, prompt the host conversation for the missing value(s). On full invocation, shell-exec `specscore idea relocate <slug> --to-repo=<target>` (with `--no-commit` passed through if present), capture stdout/stderr, surface both to the host conversation verbatim (no paraphrasing), and propagate the CLI's exit code as the skill's exit code. **Depends on** `specscore idea relocate` being available — [`idea-relocate-implementation`](https://github.com/specscore/specscore-cli/blob/main/spec/plans/idea-relocate-implementation.md) Plan Tasks 1-7 must ship first.

### Task 9: Mismatch log — write on success + failure non-blocking

**Verifies:** sidekick-capture/destination-resolution#ac:relocate-skill-appends-log-on-success, sidekick-capture/destination-resolution#ac:relocate-skill-log-write-failure-non-blocking

When the CLI verb exits `0`, append exactly one JSON line to `.synchestra/destination-resolution-log.jsonl` in the user's cwd at relocate-skill invocation time (the misfiled-artifact repo, not the target repo). The directory is created lazily if absent. The JSON line carries at minimum `ts`, `kind`, `slug`, `original_repo`, `correct_repo`; consumers tolerate unknown fields. On log-write failure (read-only path, disk full, etc.), display a single warning line to the host conversation, do NOT mask the relocate's success, and propagate the CLI's exit code (still `0`) without modification.

## Outstanding Questions

- Should the helper-prompt replay-test corpus be assembled as part of this Plan, or is it implementation-time / dogfooding work? Recommended: implementation-time. The Plan's responsibility ends at "helper file exists and conforms to contract"; iterating the wording against real captures is dogfooding once Task 7 ships. Resolve in Task 1.
- The `specstudio:relocate-idea` skill's `--no-commit` pass-through: does the skill detect `--no-commit` in its trigger arguments, or only when the user explicitly types the flag in the same form the CLI verb expects? Recommended: the skill accepts `--no-commit` verbatim and forwards it; no synonym handling in v1. Resolve in Task 8.
- The mismatch log's location (`.synchestra/destination-resolution-log.jsonl` in cwd) creates a new file in any repo where the relocate skill is invoked. Should the file be added to the repo's `.gitignore` automatically, or is that the user's responsibility? Recommended: user's responsibility — automatic .gitignore mutation feels like overreach. Resolve in Task 9.
- Cross-repo sequencing with `specscore-cli`: Tasks 8-9 cannot ship until [`idea-relocate-implementation`](https://github.com/specscore/specscore-cli/blob/main/spec/plans/idea-relocate-implementation.md) Tasks 1-7 ship the CLI verb. Recommended sequence: ship Tasks 1-7 of this Plan in parallel with Tasks 1-7 of the CLI plan; gate Tasks 8-9 here on the CLI verb's GA. Detail at implementation time.

---
*This document follows the https://specscore.md/plan-specification*
