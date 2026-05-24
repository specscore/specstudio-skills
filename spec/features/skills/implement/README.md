# Feature: Implement Skill

> [SpecScore.**Studio**](https://specscore.studio): | [Explore](https://specscore.studio/app/github.com/specscore/specstudio-skills/spec/features/skills/implement?op=explore) | [Edit](https://specscore.studio/app/github.com/specscore/specstudio-skills/spec/features/skills/implement?op=edit) | [Ask question](https://specscore.studio/app/github.com/specscore/specstudio-skills/spec/features/skills/implement?op=ask) | [Request change](https://specscore.studio/app/github.com/specscore/specstudio-skills/spec/features/skills/implement?op=request-change) |

**Status:** Implementing
**Source Ideas:** specstudio-implement-skill
**Supersedes:** —

## Summary

The `specstudio:implement` skill turns an approved SpecScore Plan into focused, AC-traceable source-code changes by dispatching one subagent per task in parallel batches computed from the Plan's `**Depends-On:**` dependency graph. The skill stages all changes via `git add` (mirroring the upstream `ideate` / `specify` / `plan` discipline), provides the user a structured `Verifies:` commit-message trailer template, and gates batch progression on explicit per-batch user approval of the consolidated staged diff. Implementation lives at [`skills/implement/`](../../../../skills/implement/).

**Provenance.** This Feature follows the architectural shape established by `specstudio:specify` and `specstudio:plan` (same `<HARD-GATE>` pattern, same body-metadata schema, same lint → reviewer → user gate sequence, same auto-stage-on-create discipline) and adapts the parallel-dispatch model proven by `superpowers:subagent-driven-development` and `superpowers:dispatching-parallel-agents` to the per-batch checkpointed cadence SpecStudio requires. The departures from those upstream skills — staging-only instead of commit-per-task, per-batch user gate instead of continuous execution — are deliberate and documented in the source Idea.

## Problem

`specstudio:plan` produces a Plan whose tasks reference specific AC IDs via `**Verifies:**` lines. The next step — actually writing the source code that makes those ACs pass — has no SpecStudio skill. Users fall back to general-purpose implementation skills (`agent-skills:test-driven-development`, `agent-skills:incremental-implementation`, `superpowers:test-driven-development`, `superpowers:executing-plans`) that have no awareness of the Plan's AC linkage. As a result, commits don't reference the ACs they satisfy, partial implementations are hard to detect, parallel-eligible work is serialized unnecessarily, and the spec↔code coherence loop breaks at the implementation step.

`specstudio:implement` exists to close that loop by:

1. Reading the dependency graph from the Plan's `**Depends-On:**` task fields and executing tasks in maximally-parallel batches.
2. Producing staged changes whose accompanying commit-message template carries a mandatory `Verifies: <feature-slug>#ac:<ac-slug>, …` trailer that traces every commit back to the ACs it satisfies.
3. Preserving the user's control over commit shape (branch, message, granularity, signing) by staging only — never committing on the user's behalf.
4. Keeping a human-in-the-loop at the batch boundary, not at the task boundary, so the per-batch checkpoint catches misimplementation without bureaucratic per-task confirmations.

The Feature defines exactly what that contract must look like so the skill is enforceable and downstream skills (`verify`, `recap`, `review`) can rely on it.

## Behavior

### Invocation and skip conditions

The skill is invoked when an approved Plan is ready for implementation.

#### REQ: invocation-triggers

The skill MUST respond to the triggers `implement`, `/implement`, `specstudio:implement`, "implement this plan", "implement this task", and the event `plan.approved`. It MAY respond to additional natural-language phrasings of the same intent.

#### REQ: requires-approved-plan

The skill MUST refuse to proceed unless its input is a Plan at `spec/plans/<slug>.md` with `**Status:**` in `{Approved, Implementing}`. When given a Plan with `Status ∈ {Draft, Under Review, Completed}`, the skill MUST tell the user (a) the Plan must be approved first via `specstudio:plan`, or (b) the Plan is already complete and there's nothing to implement, and stop without dispatching any subagent.

#### REQ: requires-approved-source-feature

The skill MUST also verify that the Plan's `**Source Feature:**` resolves to a Feature with `**Status:**` in `{Approved, Implementing, Stable}`. If the Feature has regressed to `Draft` or `Under Review` since the Plan was approved (spec drift), the skill MUST stop, surface the drift, and recommend either re-approving the Feature via `specstudio:specify` or reverting the Feature to its prior approved revision. The skill MUST additionally confirm the Source Feature exists at git HEAD via `git cat-file -e HEAD:spec/features/<feature-slug>/README.md`; a Feature that exists only in the working tree (uncommitted) MUST cause pre-flight to refuse and instruct the user to commit the Feature first, because the `Verifies:` commit-message trailer must reference a Feature that exists in git history.

### Hard gate

The skill enforces a hard gate downstream — once invoked, every batch must produce a complete, lint-clean, reviewer-approved, user-approved staging set before any downstream skill can run.

#### REQ: hard-gate

The skill MUST NOT invoke `specstudio:verify`, `writing-plans`, or ANY downstream skill (no `frontend-design`, `mcp-builder`, etc.) until ALL FIVE conditions hold for every batch produced in the current invocation:

1. Every subagent in the batch returned a terminal status (`DONE`, `DONE_WITH_CONCERNS`, or `BLOCKED` with user decision); no subagent is still `NEEDS_CONTEXT`.
2. The consolidated staged diff for the batch is lint-clean (`specscore spec lint` exits zero against the project, including any Plan-file changes the skill staged in stub mode).
3. The conflict-detection check has passed (no line-overlap between sibling subagents' staged diffs) OR the user has explicitly approved a manual conflict-resolution path.
4. The user has explicitly approved the batch's consolidated staged diff (per `REQ:user-approval-required`).
5. The user has committed the staged set (the skill MUST NOT advance to the next batch while the working tree still has the prior batch staged but uncommitted).

A skill invocation that bypasses any condition for any batch is a contract violation.

### Reading the Plan

The Plan is the contract `implement` consumes; the skill MUST parse it deterministically and cross-check the parse against the authoritative git log.

#### REQ: plan-parse

The skill MUST parse the Plan via the `specscore` CLI's Plan parser (delegated; do not re-implement). The parse MUST surface, per task: the task number, `**Verifies:**` AC IDs, `**Status:**`, `**Depends-On:**` predecessor task numbers, and the task body (prose for `full` posture, or the canonical placeholder `<!-- implement: pending -->` for `stub` posture). Parse failures (invalid `**Mode:**`, dangling `**Depends-On:**` references, malformed AC IDs) MUST stop the skill with the CLI's lint-rule citation surfaced to the user.

#### REQ: dependency-graph-computation

The skill MUST compute the next executable batch by topological reduction of the dependency graph: the batch is the set of all tasks whose `**Depends-On:**` predecessors are all in `**Status:** done` AND whose own `**Status:** pending`. Tasks with `**Status:** in-progress` are NOT eligible for re-dispatch (they're owned by an in-flight subagent in the current or a prior invocation); tasks with `**Status:** done` are skipped; tasks with `**Status:** blocked` are skipped with a surface to the user that names the blocker and instructs the user how to resolve.

The user-resolves-block workflow uses the canonical four-token Status set, not a synthetic fifth token: to resolve a blocked task, the user manually edits the Plan's `**Status:** blocked → pending` for that task (after fixing the cause of the block); the next `implement` invocation then sees the task as eligible. The skill MUST NOT introduce or recognize any Status token outside `{pending, in-progress, done, blocked}` — `REQ:inline-self-review` enforces this.

#### REQ: git-log-cross-check

Before computing the next batch, the skill MUST scan `git log --grep='^Verifies:'` for commits in the current branch's history and cross-check each task's `**Status:**` against the git-log evidence. When a task is marked `**Status:** done` but no commit references its `**Verifies:**` ACs in a `Verifies:` trailer, the skill MUST surface the divergence to the user as a warning before proceeding. When a task is marked `**Status:** pending` but a commit DOES reference its ACs, the skill MUST offer to update the task's Status to `done` (with user confirmation) before proceeding. The git log is **authoritative**; the Plan's `**Status:**` field is the at-a-glance signal.

### Subagent dispatch and parallelism

The execution engine dispatches one subagent per task in the next executable batch, in parallel up to a cap.

#### REQ: subagent-dispatch-type

The skill MUST dispatch subagents using the Claude Code `Agent` tool with `subagent_type: general-purpose`. The skill MUST NOT require a custom-named subagent type in MVP — `general-purpose` is sufficient and ubiquitous.

#### REQ: max-parallel-cap

The skill MUST cap concurrent subagents at **5** per batch in MVP. When the next executable batch has more than 5 tasks, the skill MUST dispatch the first 5 and queue the remainder; queued tasks are dispatched as concurrent slots free (after a parallel-dispatched subagent returns). The cap MAY be made configurable in a future revision via `specscore.yaml` (see Outstanding Questions); MVP hardcodes 5.

#### REQ: subagent-isolation

Each subagent dispatched by `implement` MUST receive an isolated prompt that contains only what is necessary for its task. Subagents MUST NOT inherit the parent session's full context. The skill constructs each subagent prompt freshly per `REQ:subagent-prompt-full` or `REQ:subagent-prompt-stub`.

### Subagent contract: full posture

When the Plan's `**Mode:** full`, the subagent receives the authored task body as part of its prompt.

#### REQ: subagent-prompt-full

For a Plan with `**Mode:** full`, each subagent's prompt MUST include:

1. The task's number and name (`### Task N: <name>`).
2. The task's `**Verifies:**` AC ID list.
3. For each referenced AC, the full `Given / When / Then` text quoted from the source Feature.
4. The task's authored body (1–3 sentences of prose describing what the task does).
5. The mandatory commit-message trailer convention (`Verifies: <feature-slug>#ac:<ac-slug>, …` listing every AC ID from the task's `**Verifies:**`).
6. The discipline pointer. For tasks involving behavior change, this is the TDD pointer (delegated to `agent-skills:test-driven-development` / `superpowers:test-driven-development` when those skills are available; otherwise a minimal direct in-skill TDD instruction). For tasks with no testable surface — pure-documentation edits, file renames, deletions, formatting-only changes — the prompt MUST substitute the AC-verification adapter clause: "re-read the artifact after editing and confirm each predicate in the AC's `Then` clause directly." The adapter preserves the verification discipline while honoring the actual task shape.
7. The expected return shape (one of the four SDD-style statuses; see `REQ:subagent-status-protocol`).
8. Explicit instruction to **stage** changes via `git add` and NOT to commit.

### Subagent contract: stub posture

When the Plan's `**Mode:** stub`, the task body is the canonical placeholder marker `<!-- implement: pending -->`; the subagent must generate its own implementation approach from the ACs.

#### REQ: subagent-prompt-stub

For a Plan with `**Mode:** stub`, each subagent's prompt MUST include items 1, 2, 3, 5, 6, 7, 8 from `REQ:subagent-prompt-full` AND in addition:

a. The Plan's `## Approach` section (the planner's higher-level decomposition strategy).
b. The task's `**Depends-On:**` list and a brief summary of what those predecessor tasks delivered (per their `Verifies:` commit trailers).
c. An explicit instruction that the subagent MUST infer its own implementation approach from the source Feature's ACs and the Plan's `## Approach`, then return a SHA-free 1–2 sentence "what landed" summary describing the implementation choices it made — this summary becomes the canonical body of the task in the writeback step (see `REQ:stub-writeback-bundle`). The summary is produced at subagent-return time, BEFORE the user commits the batch, so it MUST NOT reference any commit SHA (no SHA exists yet at writeback time). The commit SHA linkage is established post-commit via the `implement.batch-completed` event payload (`REQ:event-batch-completed`), NOT via the task body.

The full task body verbatim is NOT included (it's a placeholder) — the subagent must derive its work from the ACs and the Plan's `## Approach`, not from a pre-authored body.

### Subagent status protocol (SDD-style)

Each subagent returns one of four terminal statuses, derived from `superpowers:subagent-driven-development`.

#### REQ: subagent-status-protocol

Each subagent MUST return exactly one of the following four statuses:

1. **`DONE`** — the subagent completed its task, staged all changes via `git add`, and has no concerns.
2. **`DONE_WITH_CONCERNS`** — the subagent completed its task and staged its changes, but flagged observations the user should see (e.g., "this file is getting large", "I noticed an unrelated bug in adjacent code").
3. **`NEEDS_CONTEXT`** — the subagent was unable to proceed without information not provided. The subagent MUST identify what information is missing and what would unblock it. The parent skill re-dispatches that specific subagent with augmented context; sibling subagents in the batch are unaffected.
4. **`BLOCKED`** — the subagent cannot complete the task as specified. The subagent MUST cite the specific problem (e.g., "AC X is ambiguous in section Y", "the dependency graph implies task N before task M but task M's tests already cover task N's API"). The parent skill surfaces the block to the user with the subagent's full report.

The skill MUST NOT silently retry a `BLOCKED` task without changes — either the task is re-specified, the user resolves the block (e.g., revises the Feature via `specstudio:specify`), or the user explicitly chooses to defer the task (mark `**Status:** blocked` on the Plan, skip in this batch).

### Conflict detection and rollback

Independent tasks per the dependency graph SHOULD produce non-overlapping diffs. When they don't, the skill detects and rolls back atomically.

#### REQ: conflict-detection-line-overlap

After every subagent in a batch returns a terminal status, the skill MUST run `git diff --staged` and detect line-level overlaps between sibling subagents' staged changes. Two subagents are deemed in conflict when their staged diffs touch the same file at overlapping line ranges. The detection is post-batch (after all subagents return), not real-time during dispatch.

#### REQ: conflict-batch-rollback

On detected conflict, the skill MUST:

1. Surface the conflict to the user with the two (or more) offending tasks named, the conflicting file path, and the overlapping line ranges.
2. Offer the user three resolution paths: (a) accept the rollback and re-author the Plan with explicit `**Depends-On:**` declarations that serialize the conflicting tasks; (b) manually resolve by `git restore --staged <path>` and re-running affected tasks; (c) abort the batch and investigate further.
3. On (a) or (c), unstage all of the batch's changes (`git restore --staged` for each touched file), revert each task's `**Status:** in-progress` back to `pending` (or `blocked` if the conflict implies a missing dependency), and stop. The skill MUST NOT silently auto-merge or auto-resolve conflicts.

Semantic conflicts (two subagents implementing the same Feature differently in different files) are explicitly out of MVP scope; only line-overlap detection is enforced.

### Per-batch user-approval gate

The user — not the skill, not a reviewer subagent — owns final approval of each batch's staged diff before the next batch dispatches.

#### REQ: consolidated-diff-presentation

After all subagents in a batch return terminal statuses and conflict-detection passes, the skill MUST present to the user, in a single response:

1. A summary of the batch — task numbers, names, statuses (`DONE` / `DONE_WITH_CONCERNS` / `BLOCKED`), any concerns surfaced by `DONE_WITH_CONCERNS` subagents, any BLOCKED reports with their cited causes.
2. The consolidated staged diff (output of `git diff --staged`) or, if it's very large, a per-file summary with line-counts and a recommendation to inspect with `git diff --staged` directly. The staged diff includes only the successful subagents' work (DONE / DONE_WITH_CONCERNS); BLOCKED tasks have produced no stage.
3. The proposed commit-message template containing the mandatory `Verifies:` trailer listing every AC ID covered by the **successful** tasks in the batch (DONE / DONE_WITH_CONCERNS only); BLOCKED tasks' ACs are NOT in the trailer.
4. An explicit request for approval: e.g., "Approve this batch and commit? N successful, M blocked (listed above). On approval, run `git commit -m '<template>'` (or your preferred shape) and reply `approve` to advance to the next batch."

#### REQ: partial-batch-approval

When a batch contains a mix of successful (DONE / DONE_WITH_CONCERNS) and BLOCKED subagents, the skill MUST treat the batch as a **partial success**: present the successful subagents' consolidated diff for user approval and commit, mark the BLOCKED tasks' `**Status:** blocked` on the Plan with their cited causes, and proceed to the next batch's dependency-graph computation after the user commits the partial set. The next batch's eligibility check will simply exclude the BLOCKED tasks (they have `**Status:** blocked`, not `pending`); the user resolves blocked tasks separately per `REQ:dependency-graph-computation`. The skill MUST NOT roll back successful subagents' work because of unrelated BLOCKED siblings — successful tasks are independent contracts that landed cleanly. (Atomic-rollback semantics apply only to *conflict* cases per `REQ:conflict-batch-rollback`, not to mixed terminal-status batches.)

#### REQ: user-approval-required

The skill MUST NOT advance to the next batch — dispatching new subagents, updating `**Status:**` to `done`, emitting `plan.updated`, or running the stub-posture body writeback — without explicit user approval of the consolidated batch.

#### REQ: approval-explicit-phrase

The skill MUST recognize the same explicit-approval phrase set as `specstudio:ideate`, `specstudio:specify`, and `specstudio:plan`: English `approve`, `approved`, `accept`, `accepted`, `lgtm`, plus their direct semantic equivalents in any language the user is communicating in. On detection of any qualifying phrase as a standalone or dominant response, the skill MUST proceed to per-task `**Status:**` writes and (in stub posture) writeback.

#### REQ: approval-vague-confirmation

When the user's response signals positive sentiment but does not contain a recognized explicit phrase (e.g., `looks good`, `yeah`, `ship it`, `+1`, `🚀`, `yes`, `ok`), the skill MUST treat this as a soft signal and ask one explicit confirmation question before proceeding. The skill MUST NOT silently advance to the next batch on a vague signal.

#### REQ: pre-commit-batch-block

The skill MUST detect whether the user has actually committed the staged batch before dispatching the next batch. If the working tree still has the prior batch staged but uncommitted when the user attempts to advance, the skill MUST refuse and prompt the user to commit (or unstage + re-stage as desired) before continuing.

### Stage-only commit-message template

The skill never commits; it provides a structured commit message the user pastes.

#### REQ: stage-only

The skill MUST stage all subagent-produced changes via `git add` and MUST NOT run `git commit`. This matches the staging-only discipline of `ideate` / `specify` / `plan` and preserves the user's full control over commit shape (branch, message wording, granularity, signing, co-author trailers).

#### REQ: commit-message-template

The skill MUST output, with every consolidated-diff presentation, a draft commit message of the form:

```
<short summary describing what the batch implemented>

<optional longer body the user may edit>

Verifies: <feature-slug>#ac:<ac-slug>, <feature-slug>#ac:<ac-slug>, ...
```

The `Verifies:` trailer is **mandatory** and MUST list every AC ID covered by the batch's tasks (deduplicated, ordered by task number then AC slug). The short summary and body are best-effort drafts the user is expected to edit. The skill MUST NOT enforce the user's actual commit message format — that's the user's call — but the *suggested* template always includes the trailer.

#### REQ: trailer-format

The `Verifies:` trailer MUST follow the Conventional Commits trailer convention: footer block separated from the body by a blank line, `Key: value` shape, one trailer per line (so multi-AC tasks may emit `Verifies:` on multiple lines or as a comma-separated single line — either is valid). The keyword is exactly `Verifies:` (case-sensitive). AC IDs use the form `<feature-slug>#ac:<ac-slug>`.

### Per-task Status writes

The skill writes `**Status:**` on the Plan as subagents progress, in both postures.

#### REQ: status-write-on-dispatch

When the skill dispatches a subagent for a task, it MUST transition that task's `**Status:** pending → in-progress` and stage the Plan-file change in the same staging operation as any other Plan-file edits this batch produces (in stub posture; see `REQ:stub-writeback-bundle`). In full posture, the Plan-file edit is the only Plan-side change and is staged alongside the code changes.

#### REQ: status-write-on-return

When a subagent returns a terminal status, the skill MUST update the task's `**Status:**` accordingly:

- `DONE` → `**Status:** done`
- `DONE_WITH_CONCERNS` → `**Status:** done` (the concerns are surfaced to the user in the consolidated diff but don't change the done-ness of the task)
- `BLOCKED` → `**Status:** blocked` until the user resolves
- `NEEDS_CONTEXT` → keeps `**Status:** in-progress` while the subagent is re-dispatched

#### REQ: status-applies-to-both-postures

`**Status:**` writes apply identically in `full` and `stub` posture Plans. The body-writeback exclusion in full posture (see `REQ:full-no-body-writeback`) is specifically about task bodies, not about Status.

### Stub-posture body writeback bundled with staging

In stub posture, the skill writes back the post-hoc task body alongside the code changes.

#### REQ: stub-writeback-bundle

When the Plan's `**Mode:** stub` and a subagent returns `DONE` or `DONE_WITH_CONCERNS`, the skill MUST replace that task's placeholder body (`<!-- implement: pending -->`) with the subagent's 1–2 sentence "what landed" summary, and MUST stage the Plan-file change via `git add` as part of the same staging operation as the subagent's code changes. The Plan-file edit and the code edits land in one `git diff --staged` for the user's review; the user commits them atomically as one commit per `REQ:commit-message-template`. This minimizes user involvement: no separate "approve the journal entry" step, no placeholder SHAs to reconcile post-commit, no two-phase commit dance.

#### REQ: full-no-body-writeback

When the Plan's `**Mode:** full`, the skill MUST NOT modify any task's body text. Full-posture Plans have task bodies authored at plan time; only `**Status:**` is written by `implement`. (The user MAY manually edit a full Plan's task body at any time; `implement` simply does not author such edits itself.)

### Lint and self-review

Every staged batch passes through lint before user presentation.

#### REQ: lint-pass

After all subagents in a batch return and the skill has staged any Plan-file changes (Status updates in both postures; body writeback in stub), the skill MUST run `specscore spec lint` and confirm zero exit code before presenting the consolidated diff to the user. Lint failure on the Plan side (e.g., a malformed Status write) MUST stop the batch and surface the violation; the skill MUST NOT silently auto-fix Plan-file lint errors it caused.

#### REQ: lint-failure-recovery

On `specscore spec lint` failure that the skill itself triggered (Plan-file writes), the skill MUST:

1. Run `specscore spec lint --fix` exactly once.
2. Re-run `specscore spec lint` to verify.
3. If passing, continue and tell the user what was auto-fixed.
4. If still failing, unstage the Plan-file changes (`git restore --staged spec/plans/<slug>.md`), surface the remaining violations to the user with rule IDs, and stop the batch.

The skill MUST NOT loop `--fix` more than once.

#### REQ: inline-self-review

Before presenting the consolidated diff to the user, the skill MUST scan the staged Plan-file changes for: (a) `**Status:**` transitions that violate the state machine (e.g., `done → in-progress` without explicit user action), (b) body writebacks that include placeholder tokens (`<!-- implement: pending -->`, `TBD`, `TODO`), (c) `**Status:**` values not in the canonical four-token set. Findings MUST stop the batch and prompt the user.

### Reviewer subagent gate (no separate reviewer for code; relies on user-approval gate)

`implement` does NOT dispatch a code-quality reviewer subagent in MVP — the consolidated-diff presentation and the user's approval IS the review gate. This is a deliberate departure from `superpowers:subagent-driven-development`, which has both a spec-compliance reviewer and a code-quality reviewer per task. `specstudio:review` (a downstream skill) does multi-axis code review against the Feature; `implement` defers code quality entirely to that skill.

#### REQ: no-code-review-subagent

The skill MUST NOT dispatch a reviewer subagent against the staged code changes in MVP. The user's approval of the consolidated diff is the only quality gate; code-quality and architecture review are the responsibility of `specstudio:review` downstream. (The reviewer-subagent gate on the *Feature spec* — used by `specify` and `plan` — is unrelated; this REQ is specifically about not adding a reviewer step *inside* `implement`'s execution loop.)

### Auto-stage in git

Files the skill (or its subagents) create or modify are staged via `git add`.

#### REQ: auto-stage-on-create

When the skill or any dispatched subagent creates or modifies a file, that file MUST be `git add`-ed to the index. The skill MUST report the staged paths to the user in the consolidated-diff presentation. The skill MUST NOT commit on the user's behalf. If staging fails (no git repository, lock contention), the skill MUST surface the failure and stop the batch; staging is not optional for `implement` because the AC-traceability guarantee requires the changes be reviewable as a coherent set.

### Event emission

The skill participates in the event vocabulary.

#### REQ: event-batch-started

The skill MUST emit `implement.batch-started` when it dispatches the first subagent of a batch. Payload includes the Plan slug, the batch number, the task numbers in the batch, and the dispatched-subagent count. This is informational; downstream consumers (Hub, verify) MAY use it to display progress.

#### REQ: event-batch-completed

The skill MUST emit `implement.batch-completed` exactly once per batch, after the user approves the consolidated diff and commits the staged set. Payload includes the Plan slug, the batch number, the task numbers in the batch, the commit SHA (resolved by the skill from `git rev-parse HEAD` after detecting the user's commit), and the `Verifies:` AC IDs covered.

#### REQ: event-plan-updated

When the skill writes Status changes or stub-posture body writebacks to the Plan file, it MUST emit `plan.updated` per the Plan Feature's event contract (`REQ:event-updated` in the plan Feature). The payload's `changed_sections` MUST list every task slug whose `**Status:**` or body changed in the current batch. The `change_summary` MUST be factual and ≤2 sentences (e.g., "Batch N: tasks 3, 5, 7 transitioned to done; task 5 body journaled with commit <sha>.").

#### REQ: event-payload-change-context

All `implement.*` event payloads MUST carry the same change-context fields as Idea, Feature, and Plan events: `changed_sections`, `previous_revision` (git SHA at the start of the batch), and `change_summary` (factual, no speculation, no editorializing).

### Promotion boundary

The next skill is `specstudio:verify`, and only `specstudio:verify`.

#### REQ: transition-to-verify

After the final batch's `implement.batch-completed` event fires (the dependency graph has no more eligible tasks AND every task is `**Status:** done`), the skill MUST transition only to `specstudio:verify` (or, while `verify` is unshipped, hand back to the user with the recommendation to run their project's test/rehearse suite manually). The skill MUST NOT invoke `ideate`, `specify`, `plan`, `frontend-design`, `mcp-builder`, or any other skill on transition.

#### REQ: implement-status-transition

When the final task transitions to `**Status:** done`, the skill MUST update the Plan's body-metadata `**Status:**` from `Approved` → `Implementing` on first task dispatch (if not already there), and from `Implementing` → `Completed` when the final task lands. The transition Completed → (any further state) is user-driven (e.g., the user superseding the Plan with a follow-on). The `Approved → Implementing` transition is owned by checklist step 4b (first task dispatch); the `Implementing → Completed` transition is owned by step 18 (final-task hand-off).

### Posture immutability

The skill respects the Plan's posture as user-declared at plan time.

#### REQ: posture-respect

The skill MUST honor the Plan's `**Mode:**` (or absence-default of `full`) and MUST NOT auto-switch postures, recommend posture switches, or refuse to operate on a Plan because of its posture. Posture is the planner's call, declared at plan time; `implement` is a consumer.

#### REQ: posture-no-mid-flight-switch

The skill MUST NOT support mid-flight posture re-classification — there is no `--switch-mode` flag, no automatic switching, no protocol for handling already-written-back bodies on switch-to-full. A user who needs the other posture for an in-flight Plan MUST create a successor Plan with `**Supersedes:**` set (the plan Feature's `REQ:revise-vs-supersede` handles this).

### Tone

#### REQ: honest-pushback

The skill MUST NOT yes-machine weak Plans or silently retry blocked subagents. When a Plan has a `Status: done` task whose `Verifies:` ACs have no git-log evidence, when a subagent returns `BLOCKED` with a specific cause, when a conflict is detected, or when the source Feature has drifted since Plan approval — the skill MUST say so with specificity and propose the alternative. The acceptance bar is honest disagreement, not performative agreement.

## Interaction with Other Features

| Feature | Interaction |
|---|---|
| [Plan Skill](../plan/README.md) | `implement` is the downstream consumer of `plan`. It consumes the approved Plan, parses the dependency graph from `**Depends-On:**`, dispatches per the graph, and writes back `**Status:**` (both postures) and task bodies (stub only). The `plan.approved` event triggers `implement` (with user confirmation). |
| [Specify Skill](../specify/README.md) | `implement` reads the source Feature's `## Acceptance Criteria` section to construct subagent prompts. If the source Feature has drifted to `Draft` since Plan approval (`REQ:requires-approved-source-feature`), `implement` refuses to proceed and redirects to `specify`. |
| [Verify Skill](../verify/README.md) | `implement` is the upstream gate of `verify`. `verify` runs the source Feature's Rehearse scenarios against the implemented code; `implement` never invokes `verify` itself — `transition-to-verify` is the explicit handoff. |
| [Third-Party Integration](../../third-party-integration/README.md) | The `Verifies:` commit-message trailer is the lint-enforceable convention that `verify`, `recap`, and `review` all consume. Third-party tooling (commit hooks, dashboards) may also parse the trailer; the format is fixed by `REQ:trailer-format`. |
| Synchestra Events | Emits `implement.batch-started`, `implement.batch-completed`, and `plan.updated`, all with change-context payloads. Consumers (including Hub and downstream skills) observe these to advance their own state. |
| `specscore` CLI | The Plan parser (with `**Mode:**`, `**Status:**`, `**Depends-On:**` recognition) and lint rules `P-001..P-004` are owned by `specscore`. `implement` is a consumer of those parses and lints, not a definer. |
| `agent-skills:test-driven-development` / `superpowers:test-driven-development` | The TDD discipline applied inside each subagent. Each subagent's prompt points at one of these (when available) or a minimal in-skill TDD fallback. `implement` does not reimplement TDD. |
| `superpowers:subagent-driven-development` | The SDD-style four-status protocol (`DONE` / `DONE_WITH_CONCERNS` / `NEEDS_CONTEXT` / `BLOCKED`) is adopted from this skill verbatim. The departures — staging-only, per-batch user gate, no code-quality reviewer subagent inside the loop — are documented in `REQ:stage-only`, `REQ:user-approval-required`, and `REQ:no-code-review-subagent`. |

## Acceptance Criteria

### AC: refuse-unapproved-plan (verifies REQ:requires-approved-plan)

**Given** a Plan at `spec/plans/<slug>.md` with `**Status:** Draft`,
**When** the user invokes `specstudio:implement` against it,
**Then** the skill refuses to proceed, explains the Plan must be approved via `specstudio:plan` first, and exits without dispatching any subagent.

### AC: refuse-drifted-feature (verifies REQ:requires-approved-source-feature)

**Given** an approved Plan whose source Feature has regressed to `**Status:** Draft` (e.g., the user opened the Feature for revision after Plan approval),
**When** the skill is invoked,
**Then** the skill stops, surfaces the drift with the Feature path and status, and recommends either re-approving the Feature via `specstudio:specify` or reverting to the prior approved revision. No subagent is dispatched.

### AC: refuse-uncommitted-source-feature (verifies REQ:requires-approved-source-feature)

**Given** an approved Plan whose `**Source Feature:**` exists only in the working tree (not yet committed to git HEAD),
**When** the skill's pre-flight runs,
**Then** the skill refuses to dispatch, names the uncommitted Feature path, and instructs the user to commit the Feature first. No subagent is dispatched.

### AC: hard-gate-enforced (verifies REQ:hard-gate, REQ:user-approval-required, REQ:pre-commit-batch-block)

**Given** a batch of subagents has returned terminal statuses and the consolidated diff is staged,
**When** the user attempts to advance to the next batch without committing the prior batch first,
**Then** the skill refuses, detects the uncommitted-staged state, and prompts the user to commit (or unstage) before continuing.

### AC: dependency-graph-batching (verifies REQ:dependency-graph-computation, REQ:subagent-dispatch-type, REQ:max-parallel-cap)

**Given** an approved Plan with 7 tasks where tasks 1 and 2 have `**Depends-On:** —`, tasks 3 and 4 have `**Depends-On:** 1`, tasks 5 and 6 have `**Depends-On:** 2`, and task 7 has `**Depends-On:** 3, 5`,
**When** the skill computes the next executable batch on first invocation,
**Then** batch 1 contains tasks 1 and 2 only (dispatched as 2 parallel subagents); after both return DONE, batch 2 contains tasks 3, 4, 5, 6 (dispatched as 4 parallel subagents, well under the cap of 5); after all four return DONE, batch 3 contains task 7 only.

### AC: max-parallel-cap-enforced (verifies REQ:max-parallel-cap)

**Given** a batch eligible to contain 8 independent tasks,
**When** the skill dispatches the batch,
**Then** the skill dispatches the first 5 in parallel, queues the remaining 3, and dispatches each queued task as a concurrent slot frees (a parallel subagent returns).

### AC: subagent-prompt-full (verifies REQ:subagent-prompt-full, REQ:subagent-isolation)

**Given** a `**Mode:** full` Plan with task 3 (verifies `auth#ac:login-success`, `auth#ac:login-failure`),
**When** the skill dispatches task 3's subagent,
**Then** the subagent's prompt contains: the task name, the two AC slugs, the full `Given / When / Then` text of both ACs from the source Feature, the task's authored body verbatim, the `Verifies: auth#ac:login-success, auth#ac:login-failure` trailer instruction, the TDD discipline pointer, the four-status return-shape contract, and an explicit "stage via `git add`, do not commit" instruction. The prompt does NOT contain the parent session's prior context.

### AC: subagent-prompt-stub (verifies REQ:subagent-prompt-stub, REQ:subagent-isolation)

**Given** a `**Mode:** stub` Plan with task 4 whose body is the canonical placeholder `<!-- implement: pending -->`,
**When** the skill dispatches task 4's subagent,
**Then** the subagent's prompt contains the same items as the full case EXCEPT the authored body (because there isn't one) AND additionally includes the Plan's `## Approach` section, the predecessor tasks' brief summaries, and an explicit instruction to (a) infer implementation approach from the ACs and Plan Approach, and (b) return a SHA-free 1–2 sentence "what landed" summary at subagent-return time (BEFORE the user commits the batch) — this summary becomes the canonical body of the task via the writeback step. Commit-SHA linkage lives in the `implement.batch-completed` event payload, not in the task body.

### AC: subagent-prompt-ac-verification-adapter (verifies REQ:subagent-prompt-full)

**Given** a Plan task whose `**Verifies:**` ACs all have `Then` clauses observable purely by file state (e.g., a docs-only task: "file exists with the following H2 headings"),
**When** the skill constructs the subagent prompt,
**Then** the prompt's discipline-pointer slot contains the AC-verification adapter clause ("re-read the artifact post-edit and confirm each predicate in the AC's `Then` clause directly") instead of the TDD pointer. The other prompt items (task name, AC list, AC full text, trailer convention, status protocol, stage-only instruction) are unchanged.

### AC: subagent-status-done (verifies REQ:subagent-status-protocol, REQ:status-write-on-return)

**Given** a dispatched subagent that completes its task cleanly and stages all changes,
**When** the subagent returns `DONE`,
**Then** the skill records the subagent's stage set, transitions the task's `**Status:** in-progress → done` on the Plan, and counts the task toward the next batch's dependency satisfaction.

### AC: subagent-status-needs-context-redispatch (verifies REQ:subagent-status-protocol)

**Given** a dispatched subagent that returns `NEEDS_CONTEXT` citing a specific missing artifact (e.g., the source Feature's full Acceptance Criteria section),
**When** the parent skill processes the return,
**Then** only that specific subagent is re-dispatched with the augmented context; sibling subagents in the batch are unaffected and continue to their terminal statuses; the affected task's `**Status:**` remains `in-progress`.

### AC: subagent-status-blocked-surfaces (verifies REQ:subagent-status-protocol, REQ:honest-pushback)

**Given** a dispatched subagent that returns `BLOCKED` with a specific cause (e.g., "AC X is ambiguous: both interpretations A and B would satisfy the Then clause"),
**When** the parent skill processes the return,
**Then** the skill stops the batch, surfaces the subagent's full report to the user (with the AC slug and both interpretations quoted), and offers three resolutions: revise the Feature via `specstudio:specify`, mark the task `**Status:** blocked` and skip in this batch, or abort the invocation. The skill MUST NOT silently retry the subagent.

### AC: conflict-detected-and-rolled-back (verifies REQ:conflict-detection-line-overlap, REQ:conflict-batch-rollback)

**Given** two parallel subagents in a batch both stage changes that overlap on lines 40–55 of `src/auth.ts`,
**When** the parent skill runs `git diff --staged` post-batch,
**Then** the skill detects the line overlap, surfaces the conflict to the user with both task numbers, the file path, and the conflicting line range; offers three resolutions (rewrite Plan with explicit `**Depends-On:**`, manual `git restore --staged` + re-run, or abort); on user choice of rewrite-Plan or abort, unstages all of the batch's changes, reverts the two tasks' `**Status:**` from `in-progress` to `pending`, and stops.

### AC: consolidated-diff-and-approval (verifies REQ:consolidated-diff-presentation, REQ:user-approval-required, REQ:commit-message-template, REQ:trailer-format)

**Given** a batch where 3 subagents returned `DONE` and conflict-detection passed,
**When** the skill presents the consolidated diff to the user,
**Then** the response includes: a per-task status summary, the consolidated `git diff --staged` (or a per-file summary if very large), a draft commit message with a `Verifies:` trailer listing every AC ID covered by the 3 tasks in the form `<feature-slug>#ac:<slug>`, and an explicit approval request. The skill MUST NOT dispatch the next batch until the user replies with an approval phrase AND commits the staged set.

### AC: approval-detection (verifies REQ:approval-explicit-phrase, REQ:approval-vague-confirmation)

**Given** the consolidated-diff presentation,
**When** the user replies `approve` / `lgtm` / `承認` (or equivalent explicit phrase in their language),
**Then** the skill proceeds to status writes and (in stub posture) writeback.
**And When** the user replies with a vague positive (`looks good`, `ship it`, `🚀`),
**Then** the skill asks one explicit confirmation question and does not advance until a recognized explicit phrase is received.

### AC: stage-only-no-commit (verifies REQ:stage-only, REQ:auto-stage-on-create)

**Given** any batch of subagents completing with code changes,
**When** the parent skill processes the returns,
**Then** every changed file is `git add`-ed and NO `git commit` is run by the skill or any subagent. The user is responsible for the actual commit; the skill provides only the staged set and the commit-message template.

### AC: trailer-mandatory-and-deduplicated (verifies REQ:commit-message-template, REQ:trailer-format)

**Given** a batch of 3 tasks covering ACs `auth#ac:login-success`, `auth#ac:login-failure`, `auth#ac:session-expiry`, where task 1 verifies both `login-success` and `login-failure` and tasks 2 and 3 each verify a subset,
**When** the skill produces the commit-message template,
**Then** the `Verifies:` trailer lists each AC ID exactly once (deduplicated), ordered by task number then AC slug, in the form `Verifies: auth#ac:login-success, auth#ac:login-failure, auth#ac:session-expiry` (or as multiple `Verifies:` lines — both valid per Conventional Commits trailer convention).

### AC: status-writes-both-postures (verifies REQ:status-write-on-dispatch, REQ:status-write-on-return, REQ:status-applies-to-both-postures)

**Given** two Plans — one `**Mode:** full` and one `**Mode:** stub` — each with the same task 1 dispatched,
**When** the skill dispatches and the subagent returns DONE,
**Then** both Plans see identical `**Status:**` transitions: `pending → in-progress` on dispatch, `in-progress → done` on return. The body-writeback step (stub-only) is a separate operation; status writes are identical across postures.

### AC: stub-writeback-bundled (verifies REQ:stub-writeback-bundle)

**Given** a `**Mode:** stub` Plan with task 5's body as `<!-- implement: pending -->` and a subagent returning DONE with a "what landed" summary,
**When** the skill processes the return,
**Then** the placeholder body is replaced with the subagent's 1–2 sentence summary in the Plan file, the Plan file is `git add`-ed to the index along with the subagent's code changes, and the user sees one consolidated `git diff --staged` containing both the code and the Plan-file edit. The user commits both atomically as one commit.

### AC: full-no-body-writeback (verifies REQ:full-no-body-writeback)

**Given** a `**Mode:** full` Plan with task 2's body containing an authored 2-sentence description,
**When** the skill processes a DONE return for task 2,
**Then** only the task's `**Status:**` is written (`in-progress → done`); the task body remains exactly as authored at plan time. The skill MUST NOT modify the body.

### AC: git-log-cross-check (verifies REQ:git-log-cross-check)

**Given** a Plan re-loaded by `implement` where task 3 is marked `**Status:** done` but the git log contains no commit with `Verifies:` trailer referencing task 3's ACs,
**When** the skill computes the next batch,
**Then** the skill surfaces the divergence to the user with the task number, expected AC IDs, and absence-of-commit evidence, BEFORE dispatching any new subagents. The user decides whether to revert the Status or investigate the missing commit.

### AC: lint-stops-batch-on-failure (verifies REQ:lint-pass, REQ:lint-failure-recovery)

**Given** a Plan-file edit the skill produced that fails `specscore spec lint` (e.g., a `**Status:**` value the skill wrote was malformed by a bug),
**When** the skill enters the lint step,
**Then** the skill runs `specscore spec lint --fix` exactly once; if lint still fails, unstages the Plan-file changes via `git restore --staged spec/plans/<slug>.md`, surfaces the remaining violations with rule IDs, and stops the batch. The skill MUST NOT loop `--fix`.

### AC: no-code-review-subagent (verifies REQ:no-code-review-subagent)

**Given** a batch of subagents that all return DONE with staged code changes,
**When** the skill processes the batch,
**Then** the skill does NOT dispatch any reviewer subagent against the staged code. Code review is deferred to `specstudio:review` downstream; the user's batch-approval is the only quality gate `implement` enforces.

### AC: lifecycle-events (verifies REQ:event-batch-started, REQ:event-batch-completed, REQ:event-plan-updated, REQ:event-payload-change-context)

**Given** a batch dispatched, executed, approved by the user, and committed,
**When** each gate completes,
**Then** the skill emits `implement.batch-started` on dispatch (carrying batch number and task numbers), `implement.batch-completed` after user commit (carrying the commit SHA and covered AC IDs), and `plan.updated` for the Plan-file writes (carrying `changed_sections` listing every modified task slug, `previous_revision`, and a factual `change_summary`).

### AC: promotion-to-verify (verifies REQ:transition-to-verify, REQ:implement-status-transition)

**Given** every task in a Plan has reached `**Status:** done` via the skill's batches,
**When** the final task transitions to done,
**Then** the skill updates the Plan's body-metadata `**Status:**` from `Implementing → Completed`, offers transition only to `specstudio:verify` (or, if `verify` is unshipped, hands back to the user with the recommendation to run rehearse scenarios manually). The skill MUST NOT invoke `ideate`, `specify`, `plan`, or any other skill on transition.

### AC: posture-respect-no-switch (verifies REQ:posture-respect, REQ:posture-no-mid-flight-switch)

**Given** a `**Mode:** stub` Plan mid-implementation (some tasks done with journaled bodies, some still placeholder),
**When** the user asks the skill to switch the Plan to `**Mode:** full`,
**Then** the skill refuses, explains the one-way constraint, and refers the user to the plan Feature's `REQ:revise-vs-supersede` for the successor-Plan path. The skill does NOT modify `**Mode:**` itself.

### AC: partial-batch-mixed-terminals (verifies REQ:partial-batch-approval, REQ:consolidated-diff-presentation)

**Given** a 5-task batch where 3 subagents return DONE, 1 returns DONE_WITH_CONCERNS, and 1 returns BLOCKED with a specific cause,
**When** the skill presents the consolidated batch,
**Then** the staged diff contains only the 4 successful subagents' work, the commit-message template's `Verifies:` trailer lists only the 4 successful tasks' AC IDs (NOT the BLOCKED task's ACs), the BLOCKED task's report is surfaced with its cited cause, and after the user approves and commits, the BLOCKED task is marked `**Status:** blocked` on the Plan. The next batch's eligibility check excludes the blocked task; the user resolves it separately by editing the Plan's Status field back to `pending` after fixing the underlying cause.

### AC: inline-self-review-catches-violations (verifies REQ:inline-self-review)

**Given** a stub-posture batch where the skill is about to present the consolidated diff but the in-process Plan-file edits contain (a) a writeback body that still includes the placeholder token `<!-- implement: pending -->` because the subagent returned a degenerate summary, OR (b) a `**Status:**` value not in the canonical four-token set, OR (c) a state-machine-violating transition like `done → in-progress` without explicit user action,
**When** the skill's inline self-review scan runs,
**Then** the skill stops the batch BEFORE presenting the diff to the user, surfaces the specific violation with the offending task number and the kind of violation detected, and prompts the user to investigate (no auto-fix).

### AC: honest-pushback (verifies REQ:honest-pushback)

**Given** any case where the skill detects a Status-vs-git-log inconsistency, a BLOCKED subagent return with a specific cause, a conflict between sibling subagents, or a source-Feature drift,
**When** the skill processes the case,
**Then** the skill names the specific problem with concrete evidence (commit SHA absence, AC slug, file:line, etc.), proposes the alternative resolution, and does not silently retry or auto-fix. The acceptance bar is honest disagreement, not performative agreement.

## Open Questions

- **Configurable max-parallel cap.** MVP hardcodes the concurrent-subagent cap at 5. Should this be configurable per project via `specscore.yaml` (e.g., `implement: { max_parallel: 8 }`)? Defer until dogfooding produces a real reason to vary — likely cost/budget management for projects with expensive subagent contexts. The current hardcoded 5 is a balance between parallelism and context-window cost.
- **Active TDD-skill detection vs passive delegation.** Each subagent's prompt currently *points at* `agent-skills:test-driven-development` / `superpowers:test-driven-development` as a discipline pointer. Should the subagent's prompt actively invoke those skills via Skill-tool calls from inside its own context, or rely on the subagent's natural prompt-following behavior? Active invocation couples to those skills' public surfaces; passive is looser. Spec-time decision OR implementation-time observation.
- **Status-vs-git-log divergence resolution policy.** This Feature surfaces divergences and prompts the user (`REQ:git-log-cross-check`); the exact resolution workflow (auto-revert Status, surface-and-block, surface-and-continue) is not pinned. Recommendation: surface-and-block by default — the user must explicitly revert the Status OR investigate the missing commit before any new batch dispatches. Tighten at implementation time based on dogfooding.
- **Long-running plans across many sessions.** A Plan with 30+ tasks might span multiple `implement` invocations across multiple days. The git-log scan is the durable progress source, but the in-flight `NEEDS_CONTEXT` / `BLOCKED` state isn't durable across invocations (a partial-batch crash loses subagent state). Recommendation: when re-invoked, the skill re-derives Status from the git log first, then offers to re-dispatch any tasks marked `in-progress` or `blocked` on the prior invocation. The exact protocol for restoring partial-batch state is implementation-time work.
- **Subagent-failure observability.** When a dispatched subagent crashes (context window exhausted, tool-call failure, etc.) before returning a status, how does the parent skill detect this? Recommendation: treat any non-returning subagent after a timeout as implicit `BLOCKED` with reason "subagent did not return"; surface to user and offer re-dispatch. Timeout value and detection mechanism are implementation-time work.
- **Pre-existing Plan Status initialization on first invocation.** When `implement` is invoked against a Plan that pre-dates the plan-Feature revision (no `**Status:**` fields on tasks), the skill must initialize Status fields without misreading the git log. Recommendation: scan git log for `Verifies:` trailers referencing any of the Plan's ACs and set `**Status:** done` for matched tasks; leave the rest `pending`. The first invocation against a pre-existing Plan thus implicitly "catches up" the Status fields. Coordinate at implementation time.

## Sidekick Seeds Generated

- [implement-skill-checklist-missing-plan-body-status-transition](../../../ideas/seeds/implement-skill-checklist-missing-plan-body-status-transition.md) — captured 2026-05-19 by user
- [implement-preflight-should-require-source-feature-in-git-h](../../../ideas/seeds/implement-preflight-should-require-source-feature-in-git-h.md) — captured 2026-05-19 by user
- [implement-subagent-tdd-discipline-pointer-needs-adapter-clau](../../../ideas/seeds/implement-subagent-tdd-discipline-pointer-needs-adapter-clau.md) — captured 2026-05-19 by user
- *Shipped in sibling repo `ai-plugin-specscore`* as `skills/change-status/` (specced at `spec/features/lifecycle-status-skill/`; idea at `spec/ideas/lifecycle-status-skill.md`). Was: sidekick seed `specscore-plugin-should-expose-a-dedicated-skill-for`, captured 2026-05-19 by user. The shipped design is a single cross-kind skill (Feature + Idea) rather than the seed's original two-skill proposal.

---
*This document follows the https://specscore.md/feature-specification*
