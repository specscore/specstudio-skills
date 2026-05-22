# Feature: Sidekick Destination Resolution

> [SpecScore.**Studio**](https://specscore.studio): | [Explore](https://specscore.studio/app/p/github.com/synchestra-io/specstudio-skills/spec/features/sidekick-capture/destination-resolution?op=explore) | [Edit](https://specscore.studio/app/p/github.com/synchestra-io/specstudio-skills/spec/features/sidekick-capture/destination-resolution?op=edit) | [Ask question](https://specscore.studio/app/p/github.com/synchestra-io/specstudio-skills/spec/features/sidekick-capture/destination-resolution?op=ask) | [Request change](https://specscore.studio/app/p/github.com/synchestra-io/specstudio-skills/spec/features/sidekick-capture/destination-resolution?op=request-change) |

**Status:** Implementing
**Source Ideas:** idea-skills-destination-resolution
**Supersedes:** —

## Summary

Sub-Feature of [`sidekick-capture`](../README.md). Adds a pre-write destination-resolution step to `specstudio:sidekick` so that, in multi-repo SpecScore workspaces, the host AI agent deliberates about which repo the captured seed belongs in before the write happens. The agent's pick + one-line reason is surfaced as an inline confirmation prompt the human can accept (press enter) or override (type a different repo slug or path). Ships alongside a new `specstudio:relocate-idea` skill — a thin shell over the [`specscore idea relocate`](https://github.com/specscore/specscore-cli/blob/main/spec/features/cli/idea/relocate/README.md) CLI verb — that handles the recovery path when the picked destination turns out wrong, with opt-in local mismatch logging at `.synchestra/destination-resolution-log.jsonl` for future prompt-template tuning.

## Problem

When `specstudio:sidekick` is invoked from inside another skill running in a multi-repo workspace, the host agent is mid-flow on a different task and the sidekick skill currently writes the seed wherever cwd happens to be. Neither the user nor the agent stops to ask "wait, which repo?" — see the worked dogfood case in the source Idea where an artifact was misfiled into `specstudio-skills` (commit `c4114cb`) and relocated by hand the next day (~30 minutes of cleanup; commits `7e32851` / `160ae03`). For sidekick specifically the cognitive gap is structural — fire-and-forget capture is inherently low-attention. Deliberate-mode Idea-creation skills (`specstudio:ideate`, `specstudio:specify`) are explicitly out of scope per the source Idea — when those are invoked, the user is consciously choosing destination and the skill should trust that. The recovery half exists because some misroutes will still happen even with deliberation; the new relocate skill collapses the manual ritual to a single command.

## Behavior

The Feature ships four components: the shared deliberation-prompt helper, the `specstudio:sidekick` pre-write hook with its inline confirmation UX, the new `specstudio:relocate-idea` thin-shell skill, and the opt-in mismatch log.

### Multi-repo workspace detection

#### REQ: sibling-dir-scan

The sidekick skill, before any seed-file write, MUST scan the parent directory of the source project for sibling directories containing a `specscore.yaml` file (semantically equivalent to `find ../ -maxdepth 2 -name specscore.yaml`). Each result's parent directory — plus the source project itself — becomes a candidate destination repo. The discovery MUST happen in-process and complete in under 500ms on a workspace with ≤20 siblings. The skill MUST NOT follow symlinks out of the parent directory, and MUST skip any sibling whose name starts with `.` (hidden directories).

#### REQ: single-repo-bypass

When zero siblings with `specscore.yaml` are discovered (the source project is the only candidate), the sidekick skill MUST proceed directly to its write step without invoking the deliberation-prompt helper. This preserves today's behavior for single-repo workspaces and avoids surfacing a confirmation prompt when there is nothing to choose between.

### The shared deliberation-prompt helper

#### REQ: helper-location

The shared deliberation-prompt helper MUST live at `skills/shared/destination-resolution.md` (a sibling of the existing `skills/shared/sidekick-capture.md`). The file MUST be a self-contained reference that `specstudio:sidekick` links to from its SKILL.md without copying the body. Future skills needing the same destination-resolution pattern MAY link to it; this Feature does not enumerate such future consumers.

#### REQ: helper-contract

The helper file MUST document, in this order: (a) the explicit instruction to the host agent to deliberate about destination given the candidate repos + their identity signals + the seed content; (b) the candidate-repo identity contract — each candidate's `project.repo` value (from its `specscore.yaml`, or its directory basename as a fallback when `project.repo` is unset) and the names of its `spec/features/*/` directories (top-level dirs AND their immediate sub-directories where present, capped at 20 entries per candidate with `, …` truncation) MUST be presented to the agent as identity signals; (c) the output format contract — the agent's response MUST be exactly one line, ≤120 characters, of shape `<repo>; <reason>` where `<repo>` is one of the candidate slugs (per (b)'s identity contract); (d) the escape clause — the agent MAY respond with the literal token `UNCERTAIN` if it cannot confidently pick.

The sub-directory enrichment in (b) — adding immediate sub-dirs alongside top-level Feature dir names — is necessary because nested-Feature repos (notably `specstudio-skills`, where `spec/features/skills/{ideate,implement,init,plan,specify,relocate-idea}` enumerates individually-routable skills under one top-level `skills` directory) otherwise present only the umbrella name to the agent. Without sub-dir signals, a seed referencing the `implement` skill by name routes ambiguously because the candidates table shows only `skills`, not `skills/implement`. The 20-entry cap accommodates this enrichment without bloating the table for flat-Feature repos.

The `project.repo`-empty fallback in (b) — using the directory basename when the yaml field is unset — is necessary because the field is optional in `specscore.yaml` and a candidate with empty `project.repo` would otherwise have no name the agent could reference in section (c)'s `<repo>` slot. The fallback identifier is treated identically to a yaml-supplied `project.repo` for all downstream parsing per [REQ:parses-agent-response](#req-parses-agent-response).

#### REQ: helper-prompt-iteration

The helper's exact prompt wording is intentionally implementation-iterable. The Source Idea defers wording to "iterate via replay-testing against captured seeds." This Feature requires only the contract enumerated in [REQ: helper-contract](#req-helper-contract); the wording may evolve in `skills/shared/destination-resolution.md` without revising this Feature, provided every revision preserves the contract.

### Sidekick pre-write hook

#### REQ: invokes-helper-before-write

When sibling-dir scan returns ≥1 sibling repo (the multi-repo case), the sidekick skill MUST invoke the deliberation-prompt helper BEFORE any seed-file write. The skill MUST construct the prompt body per [REQ: helper-contract](#req-helper-contract) (substituting actual candidate identity signals) and pass it to the host agent for response. The skill MUST capture the response and proceed per the inline confirmation UX.

#### REQ: parses-agent-response

The sidekick skill MUST parse the agent's response per the helper's output contract:

- The response MUST be exactly one line, ≤120 characters total.
- The shape MUST be `<repo>; <reason>` where `<repo>` matches (case-insensitive, whitespace-trimmed) the `project.repo` of exactly one candidate (including the source project itself).
- A response containing the literal token `UNCERTAIN` (case-sensitive, whole-word) MUST be treated per [REQ: malformed-or-uncertain-response](#req-malformed-or-uncertain-response).
- Any deviation (over-length, missing `;` separator, repo not in candidate list, multiple lines) is malformed per the same REQ.

A well-formed response routes to the [shows-pick-with-reason](#req-shows-pick-with-reason) confirmation flow with the parsed values.

#### REQ: malformed-or-uncertain-response

When the agent's response is malformed (per [REQ: parses-agent-response](#req-parses-agent-response)) or contains the literal `UNCERTAIN` token, the sidekick skill MUST attempt exactly ONE retry by re-invoking the helper with an additional instruction line: `your previous response was unparseable; reply EXACTLY in <repo>; <reason> ≤120 chars, where <repo> is one of: <comma-separated-candidate-list>`. The retry MUST be made within the same host conversation context (no new agent session). If the retry response is well-formed, it routes to the shows-pick-with-reason flow. If the retry is also malformed or returns `UNCERTAIN`, the skill MUST proceed to the [shows-ask-without-pre-fill](#req-shows-ask-without-pre-fill) flow.

### Inline confirmation UX

#### REQ: shows-pick-with-reason

When the agent's response is well-formed (initial or after retry), the sidekick skill MUST display a single-line confirmation prompt to the user in the host conversation. The prompt's exact text MUST be (whitespace and punctuation literal):

```
Routing to <repo> because <reason> — press enter to accept, type other to override.
```

Where `<repo>` is the agent's pick and `<reason>` is the agent's one-line reasoning verbatim. The prompt MUST appear as the next message in the host conversation and MUST block further sidekick progress until user input arrives.

#### REQ: shows-ask-without-pre-fill

When the agent declined (`UNCERTAIN` after retry) or returned malformed output (after retry), the sidekick skill MUST display the ask-without-pre-fill prompt: a numbered list `1..N` of candidate repos (one per line, each showing the candidate's `project.repo`), followed by the line `Type a number, a repo slug, or a path to override — or press enter to abort the capture.`. The skill MUST block until user input arrives.

#### REQ: accepts-enter-as-route-to-agent-pick

In the shows-pick-with-reason flow, on user input that is empty (only whitespace, including pressing enter alone), the sidekick skill MUST route the seed write to the agent's picked candidate repo without further confirmation.

#### REQ: accepts-numbered-selection-in-ask-flow

In the shows-ask-without-pre-fill flow, on user input that is a numeric value `1..N` matching a candidate's list position, the sidekick skill MUST route to that candidate.

#### REQ: enter-aborts-in-ask-flow

In the shows-ask-without-pre-fill flow, empty input (only whitespace) MUST abort the sidekick capture — the seed file MUST NOT be written, and no `sidekick-idea.captured` event MUST be emitted. The skill MUST report the abort to the host conversation with a short line and exit cleanly (no error).

#### REQ: accepts-override-input

On user input that is non-empty and not a list-position number (numeric value outside `1..N` in the ask flow, or any non-empty text in either flow), the sidekick skill MUST interpret the input per the same form rules as the [`cli/idea/relocate --to-repo`](https://github.com/specscore/specscore-cli/blob/main/spec/features/cli/idea/relocate/README.md#req-target-repo-resolution) flag:

- Value containing no `/` is a repo slug. Resolution: match against the in-process sibling-dir scan result; unique match wins. Multiple matches or zero matches MUST display an error message and re-prompt with the same confirmation form.
- Value containing `/` is a path. Resolution: relative to the source project root, or absolute if starting with `/`. The path MUST be a directory containing a `specscore.yaml`; a non-conforming path MUST display an error message and re-prompt.

The override target becomes the seed's write destination.

### Successful write

#### REQ: post-write-success-line

After the seed write succeeds at the resolved destination, the sidekick skill MUST emit its standard success line in the form `Captured: <slug> at <path-relative-to-resolved-repo>`. The destination repo identity is already visible to the user from the confirmation UX displayed pre-write; this line confirms the write completed. The skill's existing event-emission ([REQ: emits-captured-event in sidekick-capture](../README.md#req-emits-captured-event)) fires unchanged — its payload's `path` field reflects the resolved destination repo's path.

### The `specstudio:relocate-idea` skill

#### REQ: relocate-skill-location

A new skill MUST exist at `skills/relocate-idea/SKILL.md`. The skill MUST declare itself with standard skill frontmatter (name, description, aliases) per existing repo conventions. The SKILL.md body MUST be ≤200 lines of markdown — the skill is intentionally a thin shell.

#### REQ: relocate-skill-triggers

The skill MUST respond to triggers `specstudio:relocate-idea`, `/relocate-idea`, `relocate this idea`, and `move this seed to another repo`. Invocations missing either the slug or the target repo MUST elicit a prompt to the host conversation for the missing argument(s) rather than aborting.

#### REQ: relocate-skill-invokes-cli

The skill's primary action MUST be to shell-exec `specscore idea relocate <slug> --to-repo=<target>`, passing through the user-supplied `--no-commit` flag if present. The skill MUST NOT replicate any of the CLI verb's logic (file copy, in-file rewrite, cross-repo link cleanup, commit semantics) — all relocation mechanics live in `specscore-cli`.

#### REQ: relocate-skill-surfaces-output

The skill MUST surface the CLI verb's stdout and stderr to the host conversation verbatim. On non-zero exit from the CLI, the skill MUST display the stderr message (which per the CLI's [REQ: stop-on-first-commit-failure](https://github.com/specscore/specscore-cli/blob/main/spec/features/cli/idea/relocate/README.md#req-stop-on-first-commit-failure) includes user-runnable rollback commands) without paraphrasing or adding inference. The skill MUST propagate the CLI's exit code as its own.

#### REQ: relocate-skill-writes-mismatch-log

On successful invocation of the CLI verb (exit `0`), the skill MUST append a single JSON line to `.synchestra/destination-resolution-log.jsonl` in **the user's cwd at the moment the relocate skill was invoked** (which is the repo containing the misfiled artifact before the relocate happens — not the target repo, and not necessarily the artifact's original write location if multiple relocates have been chained). The directory MUST be created lazily if absent. Log writing is best-effort: any failure to write the log line MUST NOT mask the relocate's success, MUST NOT modify the CLI's exit code, and MUST emit a single short warning line to the host conversation.

#### REQ: mismatch-log-record-schema

Each line of `.synchestra/destination-resolution-log.jsonl` MUST be a single-line JSON object containing at minimum these fields:

```json
{
  "ts": "<ISO-8601 UTC>",
  "kind": "idea" | "seed",
  "slug": "<artifact-slug>",
  "original_repo": "<source-repo-project.repo-value>",
  "correct_repo": "<target-repo-project.repo-value>"
}
```

Implementations MAY add additional fields (e.g., the agent's pick at original-write time, if that context is retrievable) without breaking schema conformance. Consumers MUST tolerate unknown fields. Schema evolution is intentionally permissive at this stage; tighten if a stable consumer emerges.

### Departures from the source Idea

The source Idea referred to this Feature by the slug `skills/sidekick/destination-resolution`. This Feature adopts the slug `sidekick-capture/destination-resolution` to match the repo's actual Feature layout — the sidekick skill's Feature lives at `spec/features/sidekick-capture/`, not under `spec/features/skills/`, and this Feature is its sub-Feature, mirroring how `cli/idea/relocate` sits under `cli/idea` in `specscore-cli`. The change is layout-only; no behavioral departure from the Idea.

The source Idea's MVP item 4 read "Success line: every sidekick write reports the chosen destination explicitly, so a misroute is visible immediately." This Feature implements that property via the *pre-write* inline confirmation (the user sees the destination before the write happens) rather than via a post-write success line. The post-write success line stays simple. Stronger property: misroutes are caught before they hit disk, not just visible after.

## Acceptance Criteria

### AC: sibling-scan-discovers-candidates
**Requirements:** [#req:sibling-dir-scan](#req-sibling-dir-scan)

Given a workspace at `~/projects/specscore/` containing `specstudio-skills`, `specscore`, `specscore-cli`, and `ai-plugin-specscore` — each with `specscore.yaml` — and `specstudio:sidekick` is invoked from inside `specstudio-skills`, the in-process scan returns the three siblings plus the source project (four candidates total). Hidden directories (e.g., `.git`, `.synchestra`) are skipped. Scan completes in under 500ms.

### AC: single-repo-bypass-writes-to-cwd
**Requirements:** [#req:single-repo-bypass](#req-single-repo-bypass)

Given a workspace with no sibling SpecScore repos (the source project is the only `specscore.yaml`-bearing directory in its parent), invoking sidekick capture proceeds directly to the write step without displaying any deliberation prompt or confirmation UX. The seed lands at `spec/ideas/seeds/<slug>.md` in the source project. Today's behavior is preserved.

### AC: helper-file-exists-and-conforms
**Requirements:** [#req:helper-location](#req-helper-location), [#req:helper-contract](#req-helper-contract)

Given the Feature is implemented, the file `skills/shared/destination-resolution.md` exists and contains four sections matching the contract: (a) instruction to deliberate, (b) candidate-identity contract describing `project.repo` (with dir-basename fallback when unset) + top-level Feature dir names + immediate sub-dir names (capped at 20 per candidate) as identity signals, (c) output format `<repo>; <reason>` ≤120 chars, (d) escape clause naming `UNCERTAIN`.

### AC: helper-prompt-iteration-no-feature-revision
**Requirements:** [#req:helper-prompt-iteration](#req-helper-prompt-iteration)

Given a change to the helper file's prompt wording that preserves the [helper-contract](#req-helper-contract), the change does NOT require a corresponding revision to this Feature spec. Lint and reviewer-subagent runs against this Feature pass before and after the helper-wording change.

### AC: invokes-helper-on-multi-repo
**Requirements:** [#req:invokes-helper-before-write](#req-invokes-helper-before-write), [#req:sibling-dir-scan](#req-sibling-dir-scan)

Given the sibling-dir scan finds ≥1 sibling with `specscore.yaml`, when sidekick is invoked, the deliberation-prompt helper is invoked BEFORE any file is written to disk. Verified by checking that no `spec/ideas/seeds/<slug>.md` exists in any candidate repo until after the helper has produced a response and the confirmation UX has displayed.

### AC: parses-well-formed-response
**Requirements:** [#req:parses-agent-response](#req-parses-agent-response)

Given the candidate list is `[specstudio-skills, specscore, specscore-cli]` and the host agent's response is exactly `specscore; the convention itself belongs alongside the spec text`, the sidekick skill parses `<repo>=specscore` and `<reason>=the convention itself belongs alongside the spec text` and proceeds to the shows-pick-with-reason confirmation flow with those values.

### AC: parses-rejects-unknown-repo
**Requirements:** [#req:parses-agent-response](#req-parses-agent-response)

Given the candidate list is `[specstudio-skills, specscore]` and the agent's response is `specscore-cli; this concerns CLI behavior`, the response is treated as malformed (repo not in candidate list) and the skill proceeds to the [malformed-or-uncertain-response](#req-malformed-or-uncertain-response) retry path.

### AC: parses-rejects-over-length
**Requirements:** [#req:parses-agent-response](#req-parses-agent-response)

Given the agent's response exceeds 120 characters, the response is treated as malformed and routed to the retry path. The character count is on the complete line including separator and reason.

### AC: malformed-retry-then-ask-fallback
**Requirements:** [#req:malformed-or-uncertain-response](#req-malformed-or-uncertain-response)

Given the agent's first response is malformed, the sidekick skill issues one retry with the corrective instruction. If the retry response is also malformed, the skill proceeds to the shows-ask-without-pre-fill flow. The user sees the candidate list without a pre-filled pick.

### AC: uncertain-token-triggers-retry
**Requirements:** [#req:malformed-or-uncertain-response](#req-malformed-or-uncertain-response)

Given the agent's first response is the literal token `UNCERTAIN` (case-sensitive), the skill issues one retry. If the retry is also `UNCERTAIN` (or malformed), the skill proceeds to the shows-ask-without-pre-fill flow.

### AC: shows-pick-with-reason-form
**Requirements:** [#req:shows-pick-with-reason](#req-shows-pick-with-reason)

Given a well-formed agent response `specscore; the convention belongs alongside the spec text`, the user sees exactly the prompt line `Routing to specscore because the convention belongs alongside the spec text — press enter to accept, type other to override.` in the host conversation. Whitespace and punctuation match verbatim.

### AC: enter-routes-to-agent-pick
**Requirements:** [#req:accepts-enter-as-route-to-agent-pick](#req-accepts-enter-as-route-to-agent-pick)

Given the user is at the shows-pick-with-reason prompt for agent pick `specscore`, and the user submits empty input (presses enter), the seed file is written to `<specscore-repo-root>/spec/ideas/seeds/<slug>.md`. No further confirmation prompt is displayed.

### AC: numeric-input-routes-by-position-in-ask-flow
**Requirements:** [#req:accepts-numbered-selection-in-ask-flow](#req-accepts-numbered-selection-in-ask-flow)

Given the ask-without-pre-fill flow displays candidates as `1. specstudio-skills, 2. specscore, 3. specscore-cli`, and the user submits the literal text `2`, the seed is written to the `specscore` repo. Input `4` (out of range) is treated as an override per [#req:accepts-override-input](#req-accepts-override-input).

### AC: enter-in-ask-flow-aborts
**Requirements:** [#req:enter-aborts-in-ask-flow](#req-enter-aborts-in-ask-flow)

Given the ask-without-pre-fill flow is displayed, and the user submits empty input (enter alone), the sidekick capture aborts. No seed file is written; no `sidekick-idea.captured` event is emitted. The host conversation shows a single line indicating the capture was aborted.

### AC: override-slug-resolves-via-scan
**Requirements:** [#req:accepts-override-input](#req-accepts-override-input)

Given the user submits the text `specscore-cli` (no `/`) in either confirmation flow, the input is treated as a slug. The in-process sibling-dir scan resolves it to the matching repo. The seed is written there.

### AC: override-path-bypasses-scan
**Requirements:** [#req:accepts-override-input](#req-accepts-override-input)

Given the user submits the text `../specscore` (contains `/`), the input is treated as a path. The path is resolved relative to the source project root. The directory MUST contain a `specscore.yaml`; if it does, the seed is written there.

### AC: override-invalid-reprompts
**Requirements:** [#req:accepts-override-input](#req-accepts-override-input)

Given the user submits a non-empty override that does not resolve (slug matching zero or multiple candidates, or path missing `specscore.yaml`), the skill displays a clear error message and re-displays the same confirmation prompt for another attempt. No seed file is written.

### AC: post-write-success-line
**Requirements:** [#req:post-write-success-line](#req-post-write-success-line)

After a successful seed write at the resolved destination repo, the host conversation receives exactly the line `Captured: <slug> at spec/ideas/seeds/<slug>.md` where `<slug>` is the seed's slug. The `sidekick-idea.captured` event fires with its standard payload, with the `path` field reflecting the seed's path within the resolved destination repo (per the existing [emits-captured-event](../README.md#req-emits-captured-event) REQ).

### AC: relocate-skill-file-exists
**Requirements:** [#req:relocate-skill-location](#req-relocate-skill-location)

The file `skills/relocate-idea/SKILL.md` exists with standard skill frontmatter (`name`, `description`, optional `aliases`). The body is ≤200 lines.

### AC: relocate-skill-trigger-without-args-prompts
**Requirements:** [#req:relocate-skill-triggers](#req-relocate-skill-triggers)

Given the user invokes `/relocate-idea` with no arguments, the skill prompts the host conversation for the missing slug and target repo rather than exiting with an error. Once both arguments are supplied, the skill proceeds to invocation.

### AC: relocate-skill-shells-out-to-cli
**Requirements:** [#req:relocate-skill-invokes-cli](#req-relocate-skill-invokes-cli)

Given the user invokes `/relocate-idea foo --to-repo=specscore`, the skill shell-execs the command `specscore idea relocate foo --to-repo=specscore` (with the `--no-commit` flag passed through if the user supplied it). The skill does not perform any file copy, in-file rewrite, or commit operation itself — those happen inside the CLI.

### AC: relocate-skill-surfaces-cli-output
**Requirements:** [#req:relocate-skill-surfaces-output](#req-relocate-skill-surfaces-output)

Given the CLI verb exits non-zero with stderr containing recovery commands (e.g., `git -C specstudio-skills reset HEAD~1 --hard`), the skill surfaces the stderr verbatim to the host conversation, propagates the CLI's exit code, and does NOT paraphrase or summarize the stderr.

### AC: relocate-skill-appends-log-on-success
**Requirements:** [#req:relocate-skill-writes-mismatch-log](#req-relocate-skill-writes-mismatch-log), [#req:mismatch-log-record-schema](#req-mismatch-log-record-schema)

Given the CLI verb exits `0`, the skill appends one JSON-formatted line to `.synchestra/destination-resolution-log.jsonl` in the source repo's working directory. The line is valid JSON containing at minimum `ts`, `kind`, `slug`, `original_repo`, `correct_repo`. The `.synchestra/` directory is created if absent.

### AC: relocate-skill-log-write-failure-non-blocking
**Requirements:** [#req:relocate-skill-writes-mismatch-log](#req-relocate-skill-writes-mismatch-log)

Given the CLI verb exits `0` but the log file is unwritable (e.g., disk full, `.synchestra/` is read-only), the skill displays a single warning line to the host conversation and exits with the CLI's exit code (still `0`). The relocate's success is not masked.

## Outstanding Questions

- **Non-interactive invocation.** The inline confirmation UX assumes a human is reachable via the host conversation. How should sidekick behave when invoked from a non-interactive context (CI, scripted sub-agent, headless automation)? A `--no-prompt` flag that auto-accepts the agent's pick is the obvious answer but the source Idea did not enumerate this case; defer until a concrete non-interactive use case appears.
- **Seed frontmatter deliberation-context capture.** Should the seed's frontmatter optionally record the deliberation context (agent's pick, agent's reason, user-accepted vs. overrode) for retrospective debugging beyond the mismatch log? Lean: no for v1 — frontmatter is for cross-tool query of stable artifact identity, not for session-specific decision context. The mismatch log already captures the actionable signal (the move-to-correct delta). Revisit if prompt-tuning needs richer ground truth than the log provides.
- **Helper-prompt wording iteration loop.** The Source Idea specifies replay-testing against captured seeds as the validation mechanism, but the corpus and iteration loop are not specified here. Implementer iterates the wording inside `skills/shared/destination-resolution.md` against the captured-seed corpus once the rest of the Feature is in place; the iteration loop itself is implementation-time work.
- **Cross-repo coordination sequence with `specscore-cli`.** This Feature depends on `cli/idea/relocate` shipping first in `specscore-cli` (per the Source Idea). The actual sequencing (experimental flag → skill ships here → promote CLI to GA) is implementation-time work, tracked in the Source Idea's Outstanding Questions; not re-enumerated here.

---
*This document follows the https://specscore.md/feature-specification*
