---
format: https://specscore.md/idea-specification
status: Approved
---

# Idea: Workflow-Driven Batch Execution for implement

**Status:** Approved
**Date:** 2026-06-04
**Owner:** alexander.trakhimenok
**Promotes To:** —
**Supersedes:** —
**Related Ideas:** depends_on:implement-per-task-execution-mode

## Problem Statement

How might we run implement's per-batch execution as a single predefined, parameterized Claude Code Workflow — making fan-out, result collection, conflict-detection, and lint deterministic code with a live /workflows progress view — while keeping reviewer gates and human checkpoints in the conversational skill layer between batches?

## Context

Today implement's within-batch orchestration is model-driven: SKILL.md instructs the agent to topologically batch tasks, dispatch up to 5 subagents, collect terminal statuses, run line-overlap conflict detection, and lint — all as prose the model executes. Under long context the model can silently drop a batch, mis-order the topo sort, or wave off a conflict, and the only progress signal is a wall of text. Claude Code's Workflow tool offers deterministic JS orchestration (parallel/pipeline primitives, schema-validated agent returns) plus a live /workflows progress tree. Reviewer gates (implementation.pre_commit / pre_push), which may require a human, cannot live inside a background workflow. Neighbors: implement-execution-topology (Specified — branch roles and transitions; this Idea depends on it), detached-background-implement (Approved — in-session vs claude --bg run placement), approval-autonomy (Draft — commit/push cadence and gates), reviewer-gates.

## Recommended Direction

Make implement's per-batch execution a single, predefined, parameterized Workflow distributed with the plugin and invoked once per executable batch. The skill keeps everything that needs context or a human: it parses the Plan, computes the next topological batch, evaluates reviewer gates, decides commit/push, and loops. For each batch it launches the Workflow with that batch's tasks (name, AC text, body, Verifies IDs) as args. The Workflow owns the deterministic compute and visibility: it dispatches one schema-constrained agent per task in parallel (each following the implement subagent contract — isolated prompt, TDD discipline, structured DONE / DONE_WITH_CONCERNS / NEEDS_CONTEXT / BLOCKED return), collects results, runs line-overlap conflict detection and lint as code, and returns a consolidated batch result (per-task status, the changes, AC coverage, concerns). The Workflow performs NO git commits, NO merges, NO pushes and fires NO gates — all repo-state mutation and every gate stay in the skill, between Workflow calls. Gates are the seam: because a human gate cannot block a background workflow, each batch is its own Workflow invocation and the skill gates between them. The Workflow is reserved for batches that actually fan out: a width-1 (sequential) batch gains nothing from the harness — no parallelism, no sibling conflict to detect — so it skips the Workflow and is run directly by the skill, with the inline-vs-subagent choice for those single-task batches owned by the sibling Idea `implement-per-task-execution-mode`. Net effect: the fragile parts (batching, fan-out, conflict-detect, lint) become guaranteed-correct code with a live /workflows view, while the human-in-the-loop parts stay in the conversation where they belong.

## Alternatives Considered

- **Status quo — model-driven Agent dispatch from the skill.** The skill prose tells the model to batch, dispatch, collect, conflict-check, and lint. Zero new mechanism, but it is exactly what we are replacing: orchestration correctness is model-dependent (silent dropped batches, mis-ordered topo sort, waved-off conflicts) and the only progress signal is a wall of text. Rejected as the target; it remains the fallback when the Workflow tool is unavailable.
- **Workflow owns the whole plan run (phases = batches).** One Workflow for the entire Plan, each batch a phase, giving the richest single `/workflows` tree. Rejected for MVP: a `type: human` reviewer gate cannot pause a background workflow, so this only holds when every gate auto-approves. Worth revisiting as an optimization for fully-autonomous (auto-approve-only) runs.
- **Workflow owns the run including gates, degrading human gates to auto-approve.** Keeps everything in one harness but neuters the gate system — a human gate would either silently auto-approve or block invisibly in the background. Rejected outright: it defeats the purpose of reviewer gates.

## MVP Scope

In-session implement only: one predefined Workflow script, invoked once per executable batch **of width ≥ 2**, that fans out task-to-agent in parallel, collects schema-validated returns, runs line-overlap conflict detection and lint, and returns a consolidated batch result to the skill — which then evaluates the implementation.pre_commit gate, commits the approved set, and loops to the next batch. Width-1 batches skip the Workflow entirely (run directly by the skill — see `implement-per-task-execution-mode`). Prove on a 2-3 task batch that the /workflows tree shows live per-task status and that gates still fire in the skill between batches.

## Not Doing (and Why)

- Detached-background composition — MVP is in-session where a human watches /workflows; composing with claude --bg is deferred to detached-background-implement
- resumeFromRunId batch recovery — deferred; a failed batch re-runs from the skill loop, not via workflow resume
- Single whole-plan workflow with phases as batches — rejected for MVP because human gates cannot pause a background workflow; per-batch invocation keeps gates between calls
- Running reviewer gates inside the Workflow — gates may require a human and stay in the skill layer by design
- Re-specifying branch topology or commit/push cadence — owned by implement-execution-topology and approval-autonomy respectively

## Key Assumptions to Validate

| Tier | Assumption | How to validate |
|------|------------|-----------------|
| Must-be-true | A predefined Workflow script can be distributed via the plugin and invoked by the implement skill (either inlined into SKILL.md or shipped as a `.js` file the skill locates by `scriptPath`). | Ship a trivial workflow in the plugin and have a skill invoke it end-to-end; confirm path/version resolution works from the installed plugin cache. |
| Must-be-true | A Workflow `agent()` call can faithfully execute the implement subagent contract (isolated prompt, TDD discipline, schema-validated DONE / DONE_WITH_CONCERNS / NEEDS_CONTEXT / BLOCKED return). | Run a 1-task batch through the Workflow and confirm the agent returns the structured status and staged changes the skill expects. |
| Should-be-true | The skill→Workflow→skill handoff (skill computes batch + gates; Workflow computes + collects) maps onto the existing implement checklist without losing gate semantics. | Walk the current implement checklist steps and assign each to skill-side or Workflow-side; confirm every gate stays skill-side and no step is orphaned. |
| Should-be-true | Per-batch Workflow invocation keeps the human gate timing intact — the Workflow completes, then the skill gates, with no human blocking inside the background harness. | Run a 2-batch plan with a `type: human` pre_commit gate; confirm the human is prompted in-conversation between batches, never inside `/workflows`. |
| Might-be-true | Workflow agents should each run in `isolation: 'worktree'` (collecting patches the skill integrates post-gate) rather than racing on a shared host index. | Spike both: shared-index concurrent `git add` vs worktree-per-agent with patch collection; compare for index-lock races and integration complexity. |


## SpecScore Integration

- **New Features this would create:** a `workflow-driven-batch-execution` Feature defining the skill↔Workflow contract (what the skill passes as `args`, what the Workflow returns), the Workflow's side-effect-free-on-git guarantee, and the per-batch-invocation gate seam.
- **Existing Features affected:** `specstudio-implement-skill` (its within-batch dispatch/collect/conflict-detect/lint steps move from skill prose into the Workflow; the skill retains parsing, batching, gates, and the loop); `reviewer-gates` (the pre_commit/pre_push gates remain skill-side, evaluated between Workflow calls).
- **Dependencies:** **the cross-repo specscore Idea `plan-granularity-improvement`** (it makes per-task `depends_on` required) is a hard prerequisite — without a real dependency graph in plans, every batch is width-1 and the Workflow never fans out. `implement-execution-topology` (the Workflow's agent-isolation substrate — shared index vs worktree-per-agent — is realized over its branch roles/transitions). Pairs with `implement-per-task-execution-mode` at the width-1 boundary (this Idea skips the Workflow there; that Idea owns what runs instead). Interacts with `detached-background-implement` (deliberately not composed in MVP) and `approval-autonomy` (cadence/gates layered on top, unchanged).

## Open Questions

- **Distribution mechanism:** inline the Workflow script inside SKILL.md, or ship it as a `.js` file the skill locates by `scriptPath`? The latter is cleaner but the plugin cache path carries a version number — how does the skill discover it at runtime?
- **Change transport:** do Workflow agents stage into a shared host index the skill then reads, or each work in `isolation: 'worktree'` and return patches the skill integrates after the gate? (Couples to the topology dependency.)
- **Conflict-detection mapping:** line-overlap detection currently runs over `git diff --staged`; in a worktree-per-agent model it must run as code over collected per-agent diffs. Confirm the check is equivalent.
- **`/workflows` content:** what exactly should the live tree show per batch (phase = batch number, agent label = task name + AC IDs) so the visibility is a genuine improvement over today's text output?
- **Concurrency cap vs dependency graph:** the existing `≤5 concurrent` cap and the Workflow tool's own `min(16, cores-2)` cap both operate *within* an already-dependency-respecting batch — confirm the skill's topological batching still owns ordering and the Workflow only ever sees independently-runnable tasks.

---
*This document follows the https://specscore.md/idea-specification*
