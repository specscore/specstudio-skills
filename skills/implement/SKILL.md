---
name: implement
description: |
  Turns an approved SpecScore Plan into focused, AC-traceable
  source-code changes by dispatching one subagent per task in parallel
  batches computed from the Plan's **Depends-On:** dependency graph.
  Hard-gates on per-batch user approval of the consolidated staged
  diff. Stages by default, with an explicit per-batch commit-on-behalf
  override; provides a Verifies: commit-message trailer template.
  Also accepts a Feature directly (no Plan) or an Idea directly
  (no Feature or Plan) for single-pass conversational implementation.
  Trigger: "implement", "/implement", "implement this plan",
  "specstudio:implement", or event `plan.approved`.
aliases: [implement]
---

# Implement

Turn an approved SpecScore Plan, Feature, or Idea into staged, AC-traceable source-code changes. Plan-sourced mode uses parallel subagent dispatch with per-batch user approval. Feature-sourced and Idea-sourced modes operate as a single-pass conversation. By default the user commits the staged set; with an explicit per-batch override, the skill may commit the approved staged set on the user's behalf.

## Hard Gate

<HARD-GATE>
Do NOT invoke `specstudio:verify`, `writing-plans`, `frontend-design`, `mcp-builder`, or ANY downstream skill until ALL FIVE conditions hold for **every batch** produced in the current invocation:
  1. Every subagent in the batch returned a terminal status (`DONE`, `DONE_WITH_CONCERNS`, or `BLOCKED` with user decision); no subagent is still `NEEDS_CONTEXT`.
  2. The consolidated staged diff for the batch is lint-clean (`specscore spec lint` exits zero against the project, including any Plan-file changes staged in stub mode).
  3. The conflict-detection check has passed (no line-overlap between sibling subagents' staged diffs) OR the user has explicitly approved a manual conflict-resolution path.
  4. The user has explicitly approved the batch's consolidated staged diff.
  5. The approved staged set has been committed, either by the user or by the skill after an explicit per-batch commit-on-behalf override. The skill MUST NOT advance to the next batch while the working tree still has the prior batch staged but uncommitted.

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
- The user asks the skill to commit before the consolidated diff is approved, lint-clean, and conflict-checked → refuse; the override is available only after the batch approval gate.

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

Existing behavior, unchanged. The skill resolves a Plan, parses its tasks and dependency graph, dispatches subagents in batches, and gates each batch on user approval. `Verifies:` trailers reference Feature AC IDs (e.g., `Verifies: <feature-slug>#ac:<ac-slug>`). Per-task Status writes track progress on the Plan file. The full Checklist below applies.

### Feature-sourced (single-pass)

No Plan exists. The skill resolves a Feature directly. Instead of batch dispatch, the skill operates as a **single-pass conversation**: the user describes the change, the skill implements and stages it. There are no subagents, no batch dispatch, no task-status writes, no stub/full posture distinction.

- **Pre-flight:** Step 1(b) resolves the Feature. Step 2 validates the Feature's Status. Steps 3–6 (Plan parsing, git-log cross-check, batch computation, catch-up) are skipped.
- **Implementation:** The skill implements the user's described change conversationally, staging via `git add`.
- **Verifies: trailer:** Uses Feature AC IDs: `Verifies: <feature-slug>#ac:<ac-slug>, ...` listing every AC addressed by the staged change.
- **Lint and self-review:** `specscore spec lint` still runs against staged changes.
- **User-approval gate:** The consolidated staged diff is presented for user approval before the staged set is committed by the user or by explicit commit-on-behalf override.
- **Promotion:** On completion, hand off to `specstudio:verify` (or hand-back if unshipped), same as Plan-sourced.

### Idea-sourced (single-pass)

No Feature or Plan exists. The skill resolves an Idea directly. Same single-pass conversation model as Feature-sourced, with two differences:

- **Source of truth:** The Idea's `## Recommended Direction` section (instead of Feature ACs).
- **Verifies: trailer:** Uses `Verifies: idea:<slug>` (instead of Feature AC IDs).

All other single-pass behavior (no subagents, no batch dispatch, no task-status writes, lint, user-approval gate, commit-on-behalf override) is identical to Feature-sourced.

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
12. **Present consolidated diff.** User-facing message contains: per-task status summary (including any `DONE_WITH_CONCERNS` concerns or `BLOCKED` reports), the staged diff (or per-file summary if very large), the proposed commit-message template with mandatory `Verifies:` trailer listing every AC ID covered by **successful** tasks (DONE / DONE_WITH_CONCERNS only; BLOCKED tasks' ACs NOT included), and an explicit approval + commit instruction. Also state that the user may explicitly request the skill to commit the approved staged set on their behalf.
13. **User-approval gate.** Wait for explicit approval phrase (`approve`, `approved`, `accept`, `accepted`, `lgtm`, or semantic equivalents in any language). On vague positive (`looks good`, `ship it`, `🚀`): ask one explicit confirmation question — never silently advance.
14. **Pre-commit detection / commit override.** Before dispatching the next batch, verify the batch's staged set has been committed. If staged-but-uncommitted and the user has not explicitly requested the commit-on-behalf override, refuse to advance and prompt the user to commit or request the override. If the user explicitly requests the override, run `git commit` using either the exact message the user provided or the proposed template, then verify the new `HEAD` contains the required `Verifies:` trailer. Do not amend, squash, sign, push, or include unstaged changes unless the user explicitly asks through a separate non-implement workflow.
15. **Emit `implement.batch-completed`.** Payload: Plan slug, batch number, task numbers, commit SHA (from `git rev-parse HEAD` after the user commit or approved override commit), `Verifies:` AC IDs covered.
16. **Emit `plan.updated`.** Payload's `changed_sections` lists every task slug whose Status or body changed in this batch. `change_summary` factual, ≤2 sentences.
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

## Conflict Detection and Rollback

**Detection: line-overlap only** (post-batch). Run `git diff --staged` after all subagents return terminal statuses. Two subagents are in conflict when their staged changes touch the same file at overlapping line ranges. Semantic conflicts (two subagents implementing the same Feature differently in different files) are explicitly out of MVP scope.

**Rollback: atomic.** On detected conflict:

1. Surface to user: offending task numbers, file path, overlapping line range.
2. Offer three resolutions: (a) rewrite Plan with explicit `**Depends-On:**` that serializes the conflicting tasks, (b) manual `git restore --staged <path>` + re-run, (c) abort and investigate.
3. On (a) or (c): unstage ALL batch changes (`git restore --staged` per touched file), revert each task's `**Status:** in-progress → pending` (or → `blocked` if conflict implies a missing dependency), stop.

**Mixed terminal statuses are NOT conflicts.** A batch where 3 subagents are DONE and 2 are BLOCKED is a partial success: present the 3 DONE subagents' diff for user approval and commit, mark the 2 BLOCKED tasks `**Status:** blocked` (with cited causes), advance. Atomic-rollback semantics apply only to *line-overlap conflicts*, not to mixed-terminal batches.

## Staging, Commit Override, and Commit-Message Template

The skill stages every change via `git add` by default. Unlike `ideate` / `specify` / `plan`, `implement` may run `git commit` only after the user explicitly requests a per-batch commit-on-behalf override for a staged set that already passed conflict detection, lint, self-review, and user approval. The override is not persistent across batches.

### Commit-on-behalf override

When the user explicitly asks the skill to commit an approved staged batch (for example, "commit on my behalf" or "please commit this batch"), the skill MAY run `git commit` using the proposed template unless the user supplies an exact commit message. The commit MUST include the required `Verifies:` trailer for the successful tasks in the batch. The skill MUST verify the commit succeeded and that `git rev-parse HEAD` changed before emitting `implement.batch-completed`.

The override MUST NOT:

- Bypass the consolidated-diff approval gate.
- Commit before conflict detection, lint, and inline self-review pass.
- Commit unrelated unstaged files.
- Amend, squash, sign, push, or otherwise rewrite history unless the user leaves `implement` and explicitly requests that separate git operation.
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
4. The user commits both atomically as one commit per the template, or explicitly asks the skill to commit the approved staged set via the commit-on-behalf override.

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

- The user-approval gate on the consolidated batch diff IS the quality gate `implement` enforces.
- Code-quality and architecture review are the responsibility of `specstudio:review` downstream.
- Spec-compliance review for *the Feature spec* is owned by `specstudio:specify`'s reviewer subagent, not `implement`.

This keeps `implement` focused on dispatch + staging + per-batch user approval, and avoids triple-gating inside the loop.

## Promotion Boundary

The next skill is `specstudio:verify`, and only `specstudio:verify`.

#### Transition

When all tasks reach `**Status:** done` (no more eligible batches AND no pending/blocked tasks):

1. Update Plan body-metadata `**Status:** Implementing → Completed` via `specscore feature change-status` (or the equivalent Plan-status CLI command).
2. Re-run lint.
3. Emit `plan.updated` with the status-transition `change_summary`.
4. Transition to `specstudio:verify`. If `verify` is unshipped, hand back to the user with:

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
- [ ] User explicitly approved each batch (explicit phrase OR vague + confirmation)
- [ ] Approved staged set was committed before the next batch dispatched, either by the user or by explicit per-batch commit-on-behalf override
- [ ] Plan `**Status:**` field updated correctly per the state machine (no fifth tokens; no auto-fix violating the canonical four-token set)
- [ ] In `stub` mode: every successful task's placeholder body replaced with a SHA-free 1–2 sentence summary; bundled with code in one staging set
- [ ] In `full` mode: NO task body modified by the skill (only Status writes)
- [ ] `specscore spec lint` passes after every batch (auto-recovery via `--fix` attempted at most once on initial failure)
- [ ] All emitted events (`implement.batch-started`, `implement.batch-completed`, `plan.updated`) carry `changed_sections`, `previous_revision`, and a factual `change_summary`
- [ ] On Plan completion: Status transitioned `Implementing → Completed`; transition to `specstudio:verify` (or hand-back); no other skill invoked
- [ ] In Feature-sourced mode: no subagent dispatch, no task-status writes, `Verifies:` trailer uses Feature AC IDs
- [ ] In Idea-sourced mode: no subagent dispatch, no task-status writes, `Verifies:` trailer uses `idea:<slug>`

## Red Flags

- Running `git commit` before explicit batch approval and explicit commit-on-behalf override
- Auto-advancing past a `BLOCKED` subagent without surfacing to the user
- Silently retrying a `BLOCKED` task without user resolution
- Introducing a fifth `**Status:**` token (anything outside `{pending, in-progress, done, blocked}`)
- Writeback body that references a commit SHA (the SHA doesn't exist at writeback time; linkage lives in the event payload)
- Body writeback applied to a `full`-mode Plan (writeback is stub-only)
- Atomic rollback applied to a mixed-terminal batch (rollback is for conflicts only)
- Dispatching the next batch while the prior batch's stage is uncommitted and no approved commit-on-behalf override has committed it
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
- [events.md](../shared/events.md) — event payloads emitted by this skill.
- [sidekick-capture.md](../shared/sidekick-capture.md) — sidekick-idea handling during the skill's flow.
- [PRINCIPLES.md](../../PRINCIPLES.md) — repo-level principles (user-attention economy, batched questions, parallel work while user is idle).
- `superpowers:subagent-driven-development` — adopted four-status protocol; two deliberate departures documented above (staged-by-default batches with explicit commit override; no code-quality reviewer subagent).
- `superpowers:dispatching-parallel-agents` — parallel-fanout pattern adapted to per-batch user gate.
