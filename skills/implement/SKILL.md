---
name: implement
description: |
  Turns an approved SpecScore Plan into focused, AC-traceable
  source-code changes by dispatching one subagent per task in parallel
  batches computed from the Plan's **Depends-On:** dependency graph.
  Fires the `implementation.pre_commit` / `implementation.pre_push`
  reviewer gates at its checkpoints, so per-batch approval is
  gate-config-driven (a `type: human` reviewer is the human
  checkpoint; a `auto-approve` gate commits autonomously). Applies
  publication policy at approved implementation
  milestones; provides a Verifies: commit-message trailer template.
  Also accepts a Feature directly (no Plan) or an Idea directly
  (no Feature or Plan) for single-pass conversational implementation.
  Trigger: "implement", "/implement", "implement this plan",
  "specstudio:implement", or event `plan.approved`.
aliases: [implement]
---

# Implement

Turn an approved SpecScore Plan, Feature, or Idea into AC-traceable source-code changes. Plan-sourced mode uses parallel subagent dispatch; each commit is gated by the `implementation.pre_commit` reviewer gate and each push by `implementation.pre_push`, so per-batch approval is gate-config-driven rather than a hardcoded step (a `type: human` reviewer is the human checkpoint; a `auto-approve`/`deterministic`-only gate releases autonomously). Feature-sourced and Idea-sourced modes operate as a single-pass conversation. After the commit gate releases, the skill applies the shared publication policy at the implementation milestone; downstream verification still requires the relevant Feature or implementation commits to exist in git history.

## Hard Gate

<HARD-GATE>
Do NOT invoke `specstudio:verify`, `writing-plans`, `frontend-design`, `mcp-builder`, or ANY downstream skill until ALL FIVE conditions hold for **every batch** produced in the current invocation:
  1. Every subagent in the batch returned a terminal status (`DONE`, `DONE_WITH_CONCERNS`, or `BLOCKED` with user decision); no subagent is still `NEEDS_CONTEXT`.
  2. The consolidated staged diff for the batch is lint-clean (`specscore spec lint` exits zero against the project, including any Plan-file changes staged in stub mode).
  3. The conflict-detection check has passed (no line-overlap between sibling subagents' staged diffs) OR the user has explicitly approved a manual conflict-resolution path.
  4. The batch's `implementation.pre_commit` gate released (`Approved`) before the commit. Approval is **gate-config-driven**, not hardcoded: a `auto-approve`/`deterministic`-only gate releases autonomously (no human prompt); a gate with a `type: human` reviewer stops for that human before committing. On `Issues Found` the commit is blocked and the gate's findings are surfaced.
  5. Publication policy for the approved implementation milestone has been resolved, disclosed, and applied; and when a push/promote is attempted, its `implementation.pre_push` gate released (`Approved`) first. If the allowed actions did not create a commit, the user has committed the approved set manually before the next batch or downstream verification. The skill MUST NOT advance to the next batch while the working tree still has the prior batch staged but uncommitted.

The only skill invoked after `specstudio:implement` is `specstudio:verify` (or — while `verify` is unshipped — a hand-back to the user with that recommendation).
</HARD-GATE>

## When to Use

- **Plan-sourced:** An approved Plan at `spec/plans/<slug>.md` is ready for implementation (`**Status:**` is `Approved` or `Implementing`).
- **Plan-sourced:** The event `plan.approved` has fired and the user has confirmed they want to implement.
- **Plan-sourced:** The user wants to resume an in-flight Plan after a prior `implement` session (Plan Status: `Implementing`).
- **Feature-sourced:** A Feature at `spec/features/<slug>/README.md` has `**Status:** ∈ {Approved, Implementing, Stable}` and no Plan exists for it. The user wants to implement directly against the Feature's ACs without writing a Plan first.
- **Idea-sourced:** An Idea at `spec/ideas/<slug>.md` has `**Status:** Approved` and no Feature or Plan exists for it. The user wants to implement directly against the Idea's Recommended Direction without writing a Feature or Plan first.

**Refuse and redirect when:**

- The Plan's `**Status:**` is `Draft`, `Under Review`, or `Completed` → tell the user to run `specstudio:plan` (or that there's nothing to implement).
- The Plan's `**Source Feature:**` has regressed to `Draft` or `Under Review` → stop, surface the spec drift, recommend re-approving the Feature via `specstudio:specify` or reverting.
- The Feature's `**Status:**` is `Draft` or `Under Review` → tell the user to run `specstudio:specify` first.
- The Idea's `**Status:**` is `Draft` or `Under Review` → tell the user to run `specstudio:ideate` first.
- The user asks the skill to commit or push before the consolidated diff is lint-clean and conflict-checked and its gate has released → refuse; the commit is available only after the `implementation.pre_commit` gate releases, and the push only after the `implementation.pre_push` gate releases (and publication-policy branch-safety passes).

## Pre-Flight

1. **Input resolution.** Resolve the input to one of three entry modes, checked in priority order:
   - **(a) Plan-sourced.** `spec/plans/<slug>.md` with `**Status:** ∈ {Approved, Implementing}`. Proceed with full batch-dispatch workflow.
   - **(b) Feature-sourced.** `spec/features/<slug>/README.md` with `**Status:** ∈ {Approved, Implementing, Stable}` and no Plan exists for this Feature. Proceed in single-pass mode (see Entry Modes below).
   - **(c) Idea-sourced.** `spec/ideas/<slug>.md` with `**Status:** Approved` and no Feature or Plan exists for this Idea. Proceed in single-pass mode (see Entry Modes below).
   Refuse if no artifact matches or Status is outside the accepted set for its type.
2. **Source-Feature validity.** Read the Plan's `**Source Feature:**`. Confirm the referenced Feature is at `spec/features/<feature-slug>/README.md` with `**Status:** ∈ {Approved, Implementing, Stable}`. On regression to Draft/Under Review, stop and surface the drift. Additionally confirm the Feature exists at git HEAD via `git cat-file -e HEAD:spec/features/<feature-slug>/README.md`. If the Feature exists only in the working tree (uncommitted), refuse to dispatch and instruct the user to commit it first — the `Verifies:` trailer must reference a Feature that exists in git history.
3. **Parse the Plan.** Use `specscore` CLI's Plan parser (do not re-implement). Surface: per-task `**Verifies:**`, `**Status:**`, `**Depends-On:**`, body (prose for `full`, placeholder `<!-- implement: pending -->` for `stub`). Parse failures stop the skill with the CLI's lint-rule citation.
4. **Git-log cross-check.** Run `git log --grep='^Verifies:'` on the current branch. For each task: if Plan says `**Status:** done` but no commit references the task's ACs, surface the divergence as a warning. If Plan says `**Status:** pending` but a commit DOES reference its ACs, offer to update the Status (with user confirmation) before dispatching. **Git log is authoritative; Plan Status is the at-a-glance signal.**
5. **Compute next batch.** Topological reduction of the dependency graph: batch = tasks where all `**Depends-On:**` predecessors are `**Status:** done` AND own `**Status:** pending`. Exclude tasks in `in-progress`, `done`, or `blocked` status.
6. **Pre-existing-Plan catch-up.** If the Plan pre-dates the plan-Feature revision (no `**Status:**` fields), initialize: scan git log for `Verifies:` trailers; mark matched-AC tasks `done`, rest `pending`. Save these initializations as a Plan-file edit that will land in the first batch's staging.

## Entry Modes

### Plan-sourced (default)

The skill resolves a Plan, parses its tasks and dependency graph, dispatches subagents in batches, and gates each batch's commit on the `implementation.pre_commit` reviewer gate (and each push on `implementation.pre_push`) — so whether a human is asked per batch is gate config, not hardcoded. `Verifies:` trailers reference Feature AC IDs (e.g., `Verifies: <feature-slug>#ac:<ac-slug>`). Per-task Status writes track progress on the Plan file. The full Checklist below applies.

### Feature-sourced (single-pass)

No Plan exists. The skill resolves a Feature directly. Instead of batch dispatch, the skill operates as a **single-pass conversation**: the user describes the change, the skill implements and stages it. There are no subagents, no batch dispatch, no task-status writes, no stub/full posture distinction.

- **Pre-flight:** Step 1(b) resolves the Feature. Step 2 validates the Feature's Status. Steps 3–6 (Plan parsing, git-log cross-check, batch computation, catch-up) are skipped.
- **Implementation:** The skill implements the user's described change conversationally, staging via `git add`.
- **Verifies: trailer:** Uses Feature AC IDs: `Verifies: <feature-slug>#ac:<ac-slug>, ...` listing every AC addressed by the staged change.
- **Lint and self-review:** `specscore spec lint` still runs against staged changes.
- **Commit/push gates:** The consolidated staged diff is gated by `implementation.pre_commit` before commit and `implementation.pre_push` before push, evaluated via the reviewer-gates loader + runner (same as Plan-sourced steps 13–14). A `type: human` reviewer on the gate is the human-approval checkpoint; a `auto-approve`/`deterministic`-only gate releases autonomously.
- **Promotion:** On completion, hand off to `specstudio:verify` (or hand-back if unshipped), same as Plan-sourced.

### Idea-sourced (single-pass)

No Feature or Plan exists. The skill resolves an Idea directly. Same single-pass conversation model as Feature-sourced, with two differences:

- **Source of truth:** The Idea's `## Recommended Direction` section (instead of Feature ACs).
- **Verifies: trailer:** Uses `Verifies: idea:<slug>` (instead of Feature AC IDs).

All other single-pass behavior (no subagents, no batch dispatch, no task-status writes, lint, the `implementation.pre_commit`/`pre_push` gates, publication milestone) is identical to Feature-sourced.

## Checklist (per invocation)

Create a task for each and complete in order:

1. **Pre-flight** (steps above).
2. **If no executable batch** (all tasks done or blocked) → transition to `specstudio:verify` (or hand-back), update Plan `**Status:** Implementing → Completed`, stop.
3. **Dispatch the batch.** For each task in the next executable batch (cap at 5 concurrent — see Max-Parallel below), dispatch one subagent via the Agent tool with `subagent_type: general-purpose`. Construct an isolated prompt per posture (see Subagent Contract below). When the batch has > 5 tasks, queue the rest; dispatch each queued task as a slot frees.
4. **Stage Status writes.**
   - **4a. Task Status.** As each subagent is dispatched, transition that task's `**Status:** pending → in-progress` on the Plan file. Stage via `git add`. (In `full` mode this is the only Plan-file change; in `stub` mode it will be joined by the post-return writeback.)
   - **4b. Plan body-metadata Status (first dispatch only).** On the first task dispatched in this invocation, if the Plan's body-metadata `**Status:**` is `Approved`, transition it to `Implementing` and stage the edit. Idempotent — no-op if already `Implementing`. The counterpart `Implementing → Completed` transition is owned by step 18.
5. **Wait for terminal returns.** Each subagent returns one of `DONE` / `DONE_WITH_CONCERNS` / `NEEDS_CONTEXT` / `BLOCKED`. `NEEDS_CONTEXT` → re-dispatch that specific subagent with augmented context (sibling subagents unaffected). `BLOCKED` → surface the cited cause to the user, do NOT silently retry.
6. **Update Status fields.** `DONE` / `DONE_WITH_CONCERNS` → `**Status:** done`. `BLOCKED` (with user decision to defer) → `**Status:** blocked`. Stage all Plan-file edits.
7. **Stub-mode writeback** (only when `**Mode:** stub`). For each `DONE` / `DONE_WITH_CONCERNS` task, replace the placeholder body `<!-- implement: pending -->` with the subagent's SHA-free 1–2 sentence "what landed" summary. Stage via `git add` as part of the same staging set as the code changes.
8. **Conflict detection.** Run `git diff --staged`. Detect line-overlap between sibling subagents' changes on the same file. On conflict: surface to user with file paths and line ranges; offer three resolutions (rewrite Plan with explicit `**Depends-On:**`, manual `git restore --staged` + re-run, abort). On user choice of rewrite-Plan or abort: unstage all batch changes, revert Statuses, stop.
9. **Lint.** Run `specscore spec lint`. On failure (typically Plan-file edits the skill produced), run `specscore spec lint --fix` exactly once, re-lint. On persistent failure: unstage Plan-file changes (`git restore --staged spec/plans/<slug>.md`), surface violations with rule IDs, stop the batch.
10. **Inline self-review.** Scan staged Plan-file changes for: (a) Status transitions violating the state machine (e.g., `done → in-progress` without user action), (b) writeback bodies still containing placeholder tokens (`<!-- implement: pending -->`, `TBD`, `TODO`), (c) Status values outside the canonical four-token set. Findings stop the batch.
11. **Emit `implement.batch-started`** (already done on step 3 — confirm payload was emitted: Plan slug, batch number, task numbers, dispatched count).
12. **Present consolidated diff.** User-facing message contains: per-task status summary (including any `DONE_WITH_CONCERNS` concerns or `BLOCKED` reports), the staged diff (or per-file summary if very large), and the proposed commit-message template with mandatory `Verifies:` trailer listing every AC ID covered by **successful** tasks (DONE / DONE_WITH_CONCERNS only; BLOCKED tasks' ACs NOT included). This consolidated diff is the artifact the `implementation.pre_commit` gate (step 13) reviews; when that gate includes a `type: human` reviewer the message also carries the explicit approval instruction the human responds to (when the gate is `auto-approve`/`deterministic`-only, no approval prompt is needed). Also state that publication policy will be resolved at the checkpoint and may leave the change unstaged, stage it, commit it, or commit and push it.
13. **`implementation.pre_commit` gate (before the commit).** Approval is gate-config-driven — the skill carries no hardcoded per-batch user-approval step. Fire the `implementation.pre_commit` gate-point event ([events.md](../shared/events.md), multi-fire — once per commit) and evaluate `gates.implementation.pre_commit`: load + validate the gate's reviewer list via [reviewer-gates/loader.md](../shared/reviewer-gates/loader.md) (event key `implementation.pre_commit`), then run it via [reviewer-gates/runner.md](../shared/reviewer-gates/runner.md). The consolidated staged diff is the artifact under review; a `type: human` reviewer, if configured, reviews that diff and the runner uses this skill's existing approval-phrase recognizer (`approve`/`approved`/`accept`/`accepted`/`lgtm` or semantic equivalents → `Approved`; a vague positive like `looks good`/`ship it`/`🚀` → ask one explicit confirmation question, never silently advance; an explicit change request → `Issues Found`). Proceed to step 14 **only when the gate releases (`Approved`)**. On `Issues Found`: block the commit, surface the gate's `Blocker` findings to the user, and do not advance. With a `auto-approve`/`deterministic`-only gate (no `type: human`) the gate releases autonomously and the commit happens with no human prompt; with a `type: human` reviewer the skill stops for that human before committing. Because `implementation.pre_commit` is multi-fire, each commit is an independent gate evaluation (a fresh first-pass run per the runner's per-occurrence contract). The boundary at which commits are produced (per-task / per-batch / per-plan) is resolved per the `autonomy:` namespace — see [Commit Cadence and the `autonomy:` Namespace](#commit-cadence-and-the-autonomy-namespace) — not here.
14. **Publication checkpoint.** Build the approved manifest from subagent-touched code paths, Plan status/writeback paths, single-pass edits, and any CLI-reported touched paths. Resolve and disclose [publication-policy.md](../shared/publication-policy.md) for milestone `implement.batch-approved` in Plan-sourced mode or `implement.single-pass-approved` in Feature/Idea-sourced mode. If the allowed actions include `commit`, commit only after the unrelated-index check and verify the new `HEAD` contains the required `Verifies:` trailer. If actions do not include `commit`, refuse to advance until the user commits the approved set manually with the trailer. If actions include `push`: first fire the `implementation.pre_push` gate-point event ([events.md](../shared/events.md)) and evaluate `gates.implementation.pre_push` via [reviewer-gates/loader.md](../shared/reviewer-gates/loader.md) (event key `implementation.pre_push`) + [reviewer-gates/runner.md](../shared/reviewer-gates/runner.md); the push proceeds **only when that gate releases (`Approved`)**, and on `Issues Found` the push is blocked and the gate's findings surfaced. When that gate includes a `type: human` reviewer, present a **cumulative review** as the reviewer's context (see [Cumulative Review at the Push Gate](#cumulative-review-at-the-push-gate)) — the full set of commits accumulated during the run, not merely the final commit. The `pre_push` gate **complements** publication-policy push branch-safety — it does not replace it: branch-safety is a non-negotiable floor that runs on every push (see [Push Safety Floor](#push-safety-floor)). Do not amend, squash, sign, or include non-manifest changes unless the user explicitly broadens the manifest.
15. **Emit `implement.batch-completed`.** Payload: Plan slug, batch number, task numbers, commit SHA when one exists (from `git rev-parse HEAD` after the user commit or policy-created commit), `Verifies:` AC IDs covered, and `publication_result`.
16. **Emit `plan.updated`.** Apply publication policy for `plan.updated` only to any Plan-file changes not already included in the batch milestone, then emit with `publication_result`. Payload's `changed_sections` lists every task slug whose Status or body changed in this batch. `change_summary` factual, ≤2 sentences.
17. **Loop back to step 2.** Compute next batch; if none, transition.
18. **Final transition.** When all tasks `**Status:** done`: update Plan body-metadata `**Status:** Implementing → Completed` (the counterpart to step 4b — this is the second body-metadata Status transition the skill writes), re-run lint, emit `plan.updated`, hand off to `specstudio:verify` (or, if `verify` is unshipped, recommend the user run their project's test/Rehearse suite manually).
19. **Throughout** — watch for sidekick ideas. When an out-of-scope improvement surfaces (e.g., a Feature change, a refactoring opportunity), invoke `specstudio:sidekick` with a one-liner, acknowledge in one line, and return to the current checklist step. Do not derail.

## Subagent Contract

Each subagent is dispatched with an **isolated prompt** — it MUST NOT inherit the parent session's context. Construct the prompt freshly per posture.

### Full posture (`**Mode:** full`)

Subagent prompt contains, in this order:

1. **Task identification.** `### Task N: <task-name>`.
2. **AC list.** The task's `**Verifies:**` AC IDs.
3. **AC full text.** For each referenced AC, the complete `Given / When / Then` text quoted verbatim from the source Feature at `spec/features/<feature-slug>/README.md`.
4. **Authored task body.** The 1–3 sentence prose from the Plan task body, verbatim.
5. **Commit-message trailer convention.** `Verifies: <feature-slug>#ac:<ac-slug>, ...` listing every AC ID from this task's `**Verifies:**`.
6. **Discipline pointer.** For tasks involving behavior change: the TDD pointer — reference `agent-skills:test-driven-development` or `superpowers:test-driven-development` when available; otherwise an in-skill TDD instruction (write failing test → minimal fix → refactor). For tasks with no testable surface (pure-documentation edits, file renames, deletions, formatting-only changes): substitute the AC-verification adapter clause — "re-read the artifact after editing and confirm each predicate in the AC's `Then` clause directly." The adapter preserves the verification discipline while honoring the actual task shape.
7. **Return-shape contract.** One of `DONE` / `DONE_WITH_CONCERNS` / `NEEDS_CONTEXT` / `BLOCKED`, with required fields per status.
8. **Stage-only instruction.** "Stage your changes with `git add`. Do NOT run `git commit`. The parent skill aggregates the approved batch and handles the commit gate."

### Stub posture (`**Mode:** stub`)

Subagent prompt contains items 1, 2, 3, 5, 6, 7, 8 from full posture (item 4 — authored body — does NOT apply because there isn't one), plus:

a. **Plan-level approach.** The Plan's `## Approach` section verbatim (the planner's higher-level decomposition strategy).
b. **Predecessor summaries.** For each task in `**Depends-On:**`, a brief summary of what that predecessor delivered (extracted from the predecessor's `Verifies:` commit trailer + the predecessor task's now-journaled body, if available).
c. **Inference-and-summary instruction.** "Infer your implementation approach from the source Feature's ACs and the Plan's Approach. After staging your changes, return a SHA-free 1–2 sentence 'what landed' summary describing your implementation choices. This summary will become the canonical body of the task via the writeback step. **Do NOT reference a commit SHA** — no SHA exists yet at writeback time; SHA linkage lives in the `implement.batch-completed` event payload."

### Status protocol (both postures, adopted from SDD)

| Status | Meaning | Parent skill behavior |
|---|---|---|
| `DONE` | Task complete, changes staged, no concerns | Keep staged; mark Plan `**Status:** done`; include in batch commit |
| `DONE_WITH_CONCERNS` | Task complete + staged, but subagent flagged observations (e.g., "this file is getting large") | Keep staged; mark Plan `**Status:** done`; surface concerns to user in consolidated diff |
| `NEEDS_CONTEXT` | Subagent needs information not provided | Re-dispatch this subagent with augmented context; siblings unaffected; Plan Status stays `in-progress` |
| `BLOCKED` | Subagent cannot complete the task as specified (cites specific cause) | Surface to user with full report; offer revise-Feature / mark-blocked / abort; do NOT silently retry |

## Max Parallel

Cap concurrent subagents at **5 per batch** in MVP. When the next executable batch has > 5 tasks, dispatch the first 5 and queue the rest; dispatch each queued task as a concurrent slot frees. The cap may become configurable per project via `specscore.yaml` in a future revision — MVP hardcodes 5.

## Commit Cadence and the `autonomy:` Namespace

`implement` execution knobs live under a top-level `autonomy:` key in `specscore.yaml`, keyed by skill name (MVP: `autonomy.implement`). This is a concern distinct from `gates:` — `gates:` declares *who approves* each event; `autonomy:` declares *execution knobs*. Workflow-step names (e.g., `implement:`) MUST NOT appear as top-level config keys; the knobs are reached only via `autonomy.implement.*`.

### `commit_cadence`

`autonomy.implement.commit_cadence` selects the boundary at which the skill commits:

| Value | Boundary |
|---|---|
| `task` | one commit per task |
| `batch` | one commit per integrated batch *(default when unset)* |
| `plan` | one commit at run end |

Resolution follows the publication-policy **scope ladder** (run → session → project → user), narrower overriding broader. When unset at every scope, the cadence is `batch`. Example: `autonomy.implement.commit_cadence: task` at project scope with no narrower (run/session) override resolves to `task` → one commit per task.

`commit_cadence: plan` is allowed, but the skill MUST warn that it defers all commits to run end, weakening per-batch revert granularity (a failure late in the run cannot be reverted batch-by-batch). No additional guard mechanism is provided in MVP — the warning is the safeguard.

### Cadence drives `pre_commit` firing

The resolved cadence determines how many commits a run produces, and `implementation.pre_commit` fires **once per commit** (per task, per batch, or once for the plan) — each firing an independent gate evaluation per the runner's multi-fire semantics. So a `batch`-cadence run that produces three batch commits fires `implementation.pre_commit` three times, once before each commit. The gate-evaluation mechanics live in Checklist step 13; this section owns only *where the commit boundaries fall*.

## Conflict Detection and Rollback

**Detection: line-overlap only** (post-batch). Run `git diff --staged` after all subagents return terminal statuses. Two subagents are in conflict when their staged changes touch the same file at overlapping line ranges. Semantic conflicts (two subagents implementing the same Feature differently in different files) are explicitly out of MVP scope.

**Rollback: atomic.** On detected conflict:

1. Surface to user: offending task numbers, file path, overlapping line range.
2. Offer three resolutions: (a) rewrite Plan with explicit `**Depends-On:**` that serializes the conflicting tasks, (b) manual `git restore --staged <path>` + re-run, (c) abort and investigate.
3. On (a) or (c): unstage ALL batch changes (`git restore --staged` per touched file), revert each task's `**Status:** in-progress → pending` (or → `blocked` if conflict implies a missing dependency), stop.

**Mixed terminal statuses are NOT conflicts.** A batch where 3 subagents are DONE and 2 are BLOCKED is a partial success: present the 3 DONE subagents' diff, run it through the `implementation.pre_commit` gate, and commit on release, mark the 2 BLOCKED tasks `**Status:** blocked` (with cited causes), advance. Atomic-rollback semantics apply only to *line-overlap conflicts*, not to mixed-terminal batches. (A BLOCKED subagent is also an anomaly-halt trigger — see [Anomaly Halts and Re-arm](#anomaly-halts-and-re-arm).)

## Anomaly Halts and Re-arm

Some failures are **execution-state anomalies**, not gate verdicts — a reviewer does not "approve" a merge conflict or a BLOCKED task. Regardless of gate configuration — **including a fully autonomous (`auto-approve`) `pre_commit` gate** — `implement` MUST halt the run on any of:

- **(a)** a sibling integration/merge conflict (the line-overlap conflict of [Conflict Detection and Rollback](#conflict-detection-and-rollback));
- **(b)** a BLOCKED subagent;
- **(c)** a lint failure that `specscore spec lint --fix` did not resolve in its single pass;
- **(d)** source-Feature drift (the Plan's source Feature regressed below `Approved`).

These halts are **not subject to the gate verdict** — an autonomous gate does not suppress them.

On an anomaly-halt, `implement` MUST stop, name the **specific cause**, perform **no auto-resolution**, and **not advance**:

| Anomaly | Surfaced cause |
|---|---|
| sibling conflict | the conflicting task numbers, file paths, and overlapping line range |
| BLOCKED subagent | the BLOCKED task and its cited reason |
| unresolved lint | the remaining lint violations (with rule IDs) |
| source-Feature drift | the drifted Feature and its current status |

### Explicit re-arm

After the user addresses the cause, `implement` MUST NOT auto-resume when the anomaly clears. It resumes autonomous execution only after the user issues an **explicit re-arm signal** — the lowercase standalone token **`continue`** (recognized in the same style as the approval phrases).

A re-arm re-enables autonomy for the **remainder of the current `implement` run only**. A subsequent run starts from the resolved `autonomy:` / `gates:` config — never from a prior run's re-armed state.

## Detached / Background Execution

When `implement` is launched as a detached background session (per [Feature: Detached Background Plan Implementation](../../spec/features/detached-background-implement/README.md) — a `claude --bg` process started from the plan-approval checkpoint, running in its own worktree), it follows an **autonomous progress contract** that maximizes forward progress instead of stopping at the first obstacle:

- **Maximize progress.** Complete every task the run can. (`#ac:continues-past-a-blocker`)
- **Defer blocked tasks — do not abort.** A task the run cannot complete (needs a human decision, missing information, an unresolved test failure, or a permission it lacks) is marked `**Status:** blocked` and skipped; the run continues with other unblocked tasks. Only the blocked task's own dependents are blocked. In this mode a deferrable blocked task is **not** a run-level anomaly-halt — this overrides the `BLOCKED`-subagent halt of [Anomaly Halts and Re-arm](#anomaly-halts-and-re-arm) for background runs. (`#ac:continues-past-a-blocker`)
- **Approval-requiring actions last.** Schedule any action that will need human approval after all independently-completable work, so a pause does not stall work that could proceed. (`#ac:approval-work-deferred-last`)
- **Pause — never improvise — when only blockers remain.** When no unblocked task is left, the run pauses and waits for input. It MUST NOT abort and MUST NOT improvise a decision that requires a human. (`#ac:pause-on-remaining-blockers`)
- **Blocker surface (v1): the live session only.** Blockers are resolved by attaching to the paused session (`claude attach <id>`). The run is not required to produce a `BLOCKED.md` or any other durable blocker artifact. (`#ac:no-blocked-artifact-required`)

The **integrity** anomaly-halts still apply unchanged even in background mode — a sibling integration conflict, a lint failure unresolved after the single `--fix` pass, and source-Feature drift all still halt the whole run. Only the deferrable "unfinishable task" case is relaxed here.

## Staging, Publication Policy, and Commit-Message Template

Subagents stage their own changes so the parent can aggregate and review a consolidated diff. The parent skill applies [publication-policy.md](../shared/publication-policy.md) only after the consolidated diff has passed conflict detection, lint, self-review, and the `implementation.pre_commit` gate's release (step 13). Policy does not persist as an `implement`-specific override; durable preferences are saved only through `specscore publication set`.

### Policy-created commits

When the resolved policy allows `commit`, the skill MAY run `git commit` using the proposed template unless the user supplies an exact commit message. The commit MUST include the required `Verifies:` trailer for the successful tasks in the batch or single-pass change. Before committing, compare the approved manifest to the staged index and stop on unrelated staged paths per the shared protocol. The skill MUST verify the commit succeeded and that `git rev-parse HEAD` changed before emitting `implement.batch-completed`.

The override MUST NOT:

- Bypass the `implementation.pre_commit` gate (commit only after it releases `Approved`).
- Commit before conflict detection, lint, and inline self-review pass.
- Commit unrelated staged or unstaged files.
- Amend, squash, sign, or otherwise rewrite history unless the user leaves `implement` and explicitly requests that separate git operation.
- Push before the `implementation.pre_push` gate releases, or without branch-policy approval and an upstream branch.
- Allow subagents to commit; subagents always stage only.

### Commit-message template (provided every batch / single-pass)

**Plan-sourced and Feature-sourced:**

```
<short summary describing what was implemented>

<optional longer body the user may edit>

Verifies: <feature-slug>#ac:<ac-slug>, <feature-slug>#ac:<ac-slug>, ...
```

**Idea-sourced:**

```
<short summary describing what was implemented>

<optional longer body the user may edit>

Verifies: idea:<slug>
```

**Trailer rules:**

- Keyword is exactly `Verifies:` (case-sensitive, Conventional Commits trailer convention).
- Follows the body, separated by a blank line.
- **Plan-sourced:** Lists every AC ID covered by **successful** tasks in the batch (DONE / DONE_WITH_CONCERNS only). BLOCKED tasks' ACs are NOT in the trailer. AC IDs deduplicated, ordered by task number then AC slug.
- **Feature-sourced:** Lists every AC ID addressed by the staged change.
- **Idea-sourced:** Uses `idea:<slug>` referencing the source Idea.
- May be a single comma-separated line OR multiple `Verifies:` lines — both valid.

The skill MUST NOT enforce the user's actual commit message format (that's the user's call). But the *suggested* template always includes the trailer.

## Per-Task Status Writes

| When | Transition | Apply in postures |
|---|---|---|
| Subagent dispatched | `pending → in-progress` | both |
| Subagent returns `DONE` or `DONE_WITH_CONCERNS` | `in-progress → done` | both |
| Subagent returns `BLOCKED` (user defers) | `in-progress → blocked` | both |
| User manually resolves a `blocked` task | (user edit) `blocked → pending` | both |

`**Status:**` writes apply identically to `full` and `stub` Plans. The body-writeback exclusion in `full` mode is specifically about task bodies, not about Status.

## Stub-Posture Body Writeback (Bundled)

When `**Mode:** stub` and a subagent returns `DONE` / `DONE_WITH_CONCERNS`:

1. Replace the task's placeholder body `<!-- implement: pending -->` with the subagent's SHA-free 1–2 sentence "what landed" summary.
2. Stage the Plan-file change via `git add` as part of the **same staging set** as the subagent's code changes.
3. The user reviews one consolidated `git diff --staged` containing both code and Plan-file edits.
4. The approved publication checkpoint commits both atomically when policy allows `commit`; otherwise the user commits both atomically with the suggested template before the skill advances.

**No two-phase commit.** No placeholder SHAs to reconcile. No separate "approve the journal entry" step.

When `**Mode:** full`: NO body writeback. Task bodies were authored at plan time and remain unchanged by `implement`. Only `**Status:**` is written.

## Lint and Self-Review

After every batch's staging phase, before presenting the consolidated diff to the user:

1. **Lint.** Run `specscore spec lint`. On failure: run `specscore spec lint --fix` exactly **once**, re-lint. If still failing: unstage Plan-file changes (`git restore --staged spec/plans/<slug>.md`), surface remaining violations with rule IDs, stop the batch. The skill MUST NOT loop `--fix`.

2. **Inline self-review.** Scan staged Plan-file changes for:
   - State-machine-violating Status transitions (e.g., `done → in-progress` without explicit user action).
   - Writeback bodies still containing placeholder tokens (`<!-- implement: pending -->`, `TBD`, `TODO`).
   - Status values outside the canonical four-token set `{pending, in-progress, done, blocked}`.

Findings stop the batch and prompt the user — never auto-fix beyond the one `--fix` pass above.

## No Code-Review Subagent

**Deliberate departure from `superpowers:subagent-driven-development`.** That skill dispatches a spec-compliance reviewer AND a code-quality reviewer per task. `implement` does **neither** in MVP.

- The `implementation.pre_commit` / `implementation.pre_push` reviewer gates on the consolidated batch diff ARE the quality gates `implement` enforces. What sits in each gate (a `type: human` checkpoint, an automated `deterministic` check, or an auto-approving `auto-approve`) is project config, not hardcoded here.
- Code-quality and architecture review are the responsibility of `specstudio:review` downstream.
- Spec-compliance review for *the Feature spec* is owned by `specstudio:specify`'s reviewer subagent, not `implement`.

This keeps `implement` focused on dispatch + staging + firing the gate-point events, and avoids triple-gating inside the loop. The `implement`-specific reviewer subagents this section forgoes are distinct from the configured `implementation.pre_commit`/`pre_push` gate reviewers, which `implement` fires but does not itself author.

## Cumulative Review at the Push Gate

When `implement` fires `implementation.pre_push` and that gate includes a `type: human` reviewer, that human is the **single human checkpoint of an autonomous run**. `implement` MUST present a **cumulative review** as the reviewer's context — the full set of commits accumulated during the run, not merely the final commit:

- **Default (small change):** the run's commits plus their **consolidated diff**, when the change is within ~**150 changed lines** *and* ~**10 files**.
- **Large change:** when either bound is exceeded (more than ~150 changed lines OR more than ~10 files), switch to a **per-commit summary** — one entry per commit (subject + `Verifies:` AC IDs + file/line counts) — instead of the full consolidated diff.

This threshold keeps the human's review context bounded; the cumulative set (commits, not just the tip) is presented either way.

## Push Safety Floor

Every promote/push MUST route through [`change-publication-policy`](../shared/publication-policy.md) push branch-safety, which denies `main`/`master`/`release/*` by default. Autonomy MUST NOT weaken or bypass this floor: a publication-policy-denied branch is refused regardless of `autonomy:` / `gates:` settings. A `type: deterministic` branch-safety reviewer configured on `implementation.pre_push` **complements** the publication-policy check (it can add project-specific refusals) — it is **not** a substitute for it. Both run on every push.

## Promotion Boundary

The next skill is `specstudio:verify`, and only `specstudio:verify`.

#### Transition

When all tasks reach `**Status:** done` (no more eligible batches AND no pending/blocked tasks):

1. Update Plan body-metadata `**Status:** Implementing → Completed` via `specscore feature change-status` (or the equivalent Plan-status CLI command).
2. Add CLI-reported `touched_paths` to the final checkpoint manifest.
3. Re-run lint.
4. Apply publication policy for `plan.updated`, preserving manifest and branch safety.
5. Emit `plan.updated` with the status-transition `change_summary` and `publication_result`.
6. Transition to `specstudio:verify`. If `verify` is unshipped, hand back to the user with:

> "Plan implemented. The `specstudio:verify` skill is not yet shipped — run your project's test or Rehearse suite manually against the source Feature's ACs. Every commit references the satisfied AC IDs in its `Verifies:` trailer for traceability."

The skill MUST NOT invoke `ideate`, `specify`, `plan`, `frontend-design`, `mcp-builder`, or any other skill on transition.

## Posture Immutability

The skill respects the Plan's `**Mode:**` as user-declared at plan time and MUST NOT switch postures mid-flight. No `--switch-mode` flag, no automatic re-classification. A user who needs the other posture for an in-flight Plan creates a successor Plan with `**Supersedes:**` (the plan Feature's `REQ:revise-vs-supersede` handles this).

## Tone

The skill MUST NOT yes-machine weak Plans or silently retry blocked subagents. When the skill detects Status-vs-git-log divergence, a BLOCKED subagent return, a sibling-diff conflict, or source-Feature drift, it MUST say so with concrete evidence (commit SHA absence, AC slug, file:line, etc.) and propose the alternative. The acceptance bar is honest disagreement, not performative agreement.

## Verification

- [ ] Pre-flight checks passed: input resolved to Plan (Status ∈ {Approved, Implementing}), Feature (Status ∈ {Approved, Implementing, Stable}), or Idea (Status: Approved)
- [ ] Git-log cross-check ran; any Status-vs-git-log divergences were surfaced to the user before dispatch
- [ ] Every batch dispatched ≤5 parallel subagents; queued tasks were dispatched as slots freed
- [ ] Every subagent returned a terminal status (DONE / DONE_WITH_CONCERNS / BLOCKED) or was re-dispatched on NEEDS_CONTEXT
- [ ] Conflict detection ran post-batch; conflicts surfaced and resolved per the three resolution paths
- [ ] Consolidated batch diff presented with: per-task status summary, staged diff (or summary), `Verifies:` trailer template covering only successful tasks' AC IDs
- [ ] `implementation.pre_commit` fired before each commit and released (`Approved`) before committing; on `Issues Found` the commit was blocked and findings surfaced. Approval was gate-config-driven (a `auto-approve`/`deterministic`-only gate committed with no human prompt; a `type: human` reviewer stopped for that human) — no hardcoded per-batch user-approval step ran
- [ ] `implementation.pre_push` fired before any push/promote and released (`Approved`) before pushing; on `Issues Found` the push was blocked and findings surfaced
- [ ] Publication policy was resolved and disclosed at each approved implementation milestone, and push branch-safety still ran (the `pre_push` gate complements it, does not replace it)
- [ ] Commit cadence was resolved from `autonomy.implement.commit_cadence` (default `batch`) across the scope ladder; `pre_commit` fired once per commit the cadence produced; `commit_cadence: plan` emitted the revert-granularity warning
- [ ] Autonomy config lived under the top-level `autonomy:` key (no workflow-step name as a top-level config key)
- [ ] On any anomaly (sibling conflict / BLOCKED subagent / unresolved-after-`--fix` lint / source-Feature drift) the run halted regardless of gate config, surfaced the specific cause, and performed no auto-resolution
- [ ] After an anomaly-halt, autonomous execution resumed only on an explicit `continue` re-arm, scoped to the current run
- [ ] At a `pre_push` gate with a `type: human` reviewer, the cumulative set of run commits (consolidated diff, or per-commit summary when large) was presented — not just the final commit
- [ ] Approved staged set was committed before the next batch dispatched, either by the user or by policy-created commit after the `implementation.pre_commit` gate released
- [ ] Plan `**Status:**` field updated correctly per the state machine (no fifth tokens; no auto-fix violating the canonical four-token set)
- [ ] In `stub` mode: every successful task's placeholder body replaced with a SHA-free 1–2 sentence summary; bundled with code in one staging set
- [ ] In `full` mode: NO task body modified by the skill (only Status writes)
- [ ] `specscore spec lint` passes after every batch (auto-recovery via `--fix` attempted at most once on initial failure)
- [ ] All emitted events (`implement.batch-started`, `implement.batch-completed`, `plan.updated`) carry `changed_sections`, `previous_revision`, a factual `change_summary` where applicable, and `publication_result` after publication checkpoints
- [ ] On Plan completion: Status transitioned `Implementing → Completed`; transition to `specstudio:verify` (or hand-back); no other skill invoked
- [ ] In Feature-sourced mode: no subagent dispatch, no task-status writes, `Verifies:` trailer uses Feature AC IDs
- [ ] In Idea-sourced mode: no subagent dispatch, no task-status writes, `Verifies:` trailer uses `idea:<slug>`

## Red Flags

- Running `git commit` before the `implementation.pre_commit` gate releases (`Approved`) and an allowed publication action exists
- Running `git push` before the `implementation.pre_push` gate releases (`Approved`)
- Carrying a hardcoded, unconditional per-batch user-approval step (an always-required approval phrase) instead of letting `gates.implementation.pre_commit`'s reviewers decide — the per-batch human checkpoint exists iff a `type: human` reviewer is configured on that gate
- Prompting a human to approve a commit when `gates.implementation.pre_commit` has only `auto-approve`/`deterministic` reviewers (autonomous commit, no prompt)
- Running `git commit` or `git push` before publication policy is resolved and disclosed for the approved milestone
- Letting the `pre_push` gate substitute for publication-policy push branch-safety (they are complementary — branch-safety still runs)
- Letting publication policy bypass the `implementation.pre_commit` gate or commit unrelated staged paths
- Auto-resuming after an anomaly-halt without an explicit `continue` re-arm, or carrying a re-armed state into a later run
- Committing at a boundary other than the resolved `commit_cadence`, or omitting the `commit_cadence: plan` revert-granularity warning
- Presenting only the final commit (not the cumulative run set) to a `type: human` reviewer at the `pre_push` gate
- Placing autonomy knobs under a workflow-step top-level key (e.g., `implement:`) instead of `autonomy.implement.*`
- Auto-advancing past a `BLOCKED` subagent without surfacing to the user
- Silently retrying a `BLOCKED` task without user resolution
- Introducing a fifth `**Status:**` token (anything outside `{pending, in-progress, done, blocked}`)
- Writeback body that references a commit SHA (the SHA doesn't exist at writeback time; linkage lives in the event payload)
- Body writeback applied to a `full`-mode Plan (writeback is stub-only)
- Atomic rollback applied to a mixed-terminal batch (rollback is for conflicts only)
- Dispatching the next batch while the prior batch's stage is uncommitted and publication policy did not create the required commit
- Dispatching a code-quality reviewer subagent inside the loop (deferred to `specstudio:review`)
- Auto-switching the Plan's `**Mode:**` (posture is one-way; user creates a successor Plan)
- Looping `specscore spec lint --fix` more than once
- Speculating about user intent in `change_summary` ("User chose...", "Important changes...") instead of factual ("Tasks 3, 5 transitioned to done; task 5 body journaled.")
- Using batch dispatch or task-status writes in Feature-sourced or Idea-sourced mode (single-pass only)
- Using `Verifies: idea:<slug>` when a Feature exists (use Feature AC IDs instead)
- Using Feature AC IDs when only an Idea exists (use `Verifies: idea:<slug>` instead)

## References

- [Feature: Implement Skill](../../spec/features/skills/implement/README.md) — the SpecScore Feature this skill implements.
- [Feature: Plan Skill](../../spec/features/skills/plan/README.md) — the upstream Feature whose `**Depends-On:**`, `**Status:**`, `**Mode:**` schema this skill consumes.
- [philosophy.md](../shared/philosophy.md) — shared tenets.
- [path-conventions.md](../shared/path-conventions.md) — `spec/` vs `docs/` rules.
- [publication-policy.md](../shared/publication-policy.md) — checkpoint resolution, manifest safety, first-run preference prompt, and publication disclosure.
- [reviewer-gates/loader.md](../shared/reviewer-gates/loader.md) — load + validate the `gates.implementation.pre_commit` / `gates.implementation.pre_push` reviewer list from `specscore.yaml`.
- [reviewer-gates/runner.md](../shared/reviewer-gates/runner.md) — evaluate a loaded gate (serial dispatch, grade/verdict, `Approved` vs `Issues Found`, per-occurrence multi-fire for the gate-point events).
- [events.md](../shared/events.md) — event payloads emitted by this skill, including the `implementation.pre_commit` / `implementation.pre_push` gate-point events.
- [sidekick-capture.md](../shared/sidekick-capture.md) — sidekick-idea handling during the skill's flow.
- [PRINCIPLES.md](../../PRINCIPLES.md) — repo-level principles (user-attention economy, batched questions, parallel work while user is idle).
- `superpowers:subagent-driven-development` — adopted four-status protocol; two deliberate departures documented above (publication-policy checkpoints after batch approval; no code-quality reviewer subagent).
- `superpowers:dispatching-parallel-agents` — parallel-fanout pattern adapted to per-batch user gate.
