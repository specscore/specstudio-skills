# Idea: Per-Task Execution Mode for implement

**Status:** Approved
**Date:** 2026-06-04
**Owner:** alexander.trakhimenok
**Promotes To:** —
**Supersedes:** —
**Related Ideas:** —

## Problem Statement

How might we let a Plan declare, per task, whether implement runs it inline in the current session, as an isolated subagent, or auto — so tasks that need conversation context or tight steering run inline while the default stays isolated and reproducible?

## Context

implement's Plan-sourced mode dispatches every task as an isolated subagent (the Subagent Contract: MUST NOT inherit parent context). Feature-sourced and Idea-sourced modes already run single-pass in the current session with no subagents — so inline execution exists, but only at whole-run granularity and only for those entry modes. There is no way for a Plan author to say 'run this particular task inline, in the conversation, because it needs judgment, earlier decisions, or close steering' while keeping the rest isolated. Separately, the implement-workflow-execution-engine Idea established that width-1 (sequential) batches gain nothing from the Workflow harness and should skip it — raising the question of what runs them. This Idea answers that: inline or subagent, the plan's choice. Neighbors: specstudio-implement-skill (Subagent Contract, single-pass modes), specstudio-plan-skill (owns the Plan schema this extends), implement-workflow-execution-engine (consumes this at the width-1 boundary).

## Recommended Direction

Add a per-task execution-mode axis to the Plan, declared by the plan author and honored by implement. Three values: inline (the main session agent does the task directly, with full conversation context — no Agent dispatch); subagent (one isolated Agent dispatch per the existing Subagent Contract); auto (the skill decides — a width-2-or-wider batch runs as parallel subagents/Workflow, a width-1 batch defaults to subagent). The hard constraint: inline is valid ONLY for width-1 (sequential) batches, because the single-threaded main session cannot run a parallel batch itself — a parallel batch is always subagents. inline is the opt-in exception, not the default: it trades away context isolation and reproducibility (an inline task is coupled to whatever was in the conversation) and it spends the main session's context budget — the very thing the subagent architecture protects. So auto resolves a width-1 batch to subagent unless the plan explicitly asks for inline. Gates are unaffected: inline or subagent, the task completes and then the skill evaluates the gate exactly as today. This generalizes implement's existing single-pass (current-session) behavior from whole-run-only to per-task. The *model* a task runs on is a separate, orthogonal property owned by the cross-repo `plan-granularity-improvement` Idea (specscore); the two axes meet because model is required only for isolated execution — a subagent/workflow task carries a real model tier, while an inline task uses that Idea's `inherit` sentinel (it runs on the session model). So this Idea's execution mode is what *gates* whether a model tier is required.

## Alternatives Considered

- **Skill-heuristic only (no plan field).** Let implement infer inline vs subagent from task shape (size, file count, output noise). Rejected as the control surface: the plan author holds the intent ("this one needs the conversation") and a heuristic can't read intent. The heuristic survives — it's exactly what the `auto` value does — but it can't be the only option.
- **Runtime prompt per task ("run task 4 inline?").** Maximal control, but it derails the run with a question per sequential task. Rejected for MVP; plan-declared is quieter. Could layer on later as an override.
- **Overload the existing `**Mode:** stub|full` posture field.** Reuse the posture axis to also carry execution mode. Rejected — posture (how much the planner pre-wrote) and execution (who runs the task) are orthogonal; conflating them muddies both fields.

## MVP Scope

A new optional per-task field in the Plan schema (e.g. **Execution:** inline | subagent | auto, default auto) that the plan skill writes and lints, and that implement honors when dispatching. Prove on a 3-task sequential chain that a task marked inline runs in the current session with no subagent, a task marked subagent dispatches isolated, and the gate fires identically for both. Reject inline on any width-2-or-wider batch with a clear lint or skill error.

## Not Doing (and Why)

- Inline execution of parallel (width-2-or-wider) batches — impossible on a single-threaded main session; a parallel batch is always subagents
- Making inline the default — it forgoes isolation and reproducibility and spends main-session context; the default stays auto resolving to subagent
- Runtime or interactive per-task mode override — MVP is plan-declared; a per-run user override is deferred
- Re-specifying the Workflow harness — width-2-or-wider execution is owned by implement-workflow-execution-engine
- Defining the per-task model property — model tier and override are owned by the cross-repo `plan-granularity-improvement` Idea (specscore); this Idea only defines the execution-mode axis that *gates* model requiredness (subagent/workflow → required tier; inline → the `inherit` sentinel). Other dispatch knobs remain out of scope.

## Key Assumptions to Validate

| Tier | Assumption | How to validate |
|------|------------|-----------------|
| Must-be-true | implement can run a Plan task inline (the main agent executes it directly) and still produce the same staged diff and `Verifies:` trailer a subagent would. | Run one task inline; confirm the staged changes and the commit trailer match the subagent path. |
| Must-be-true | The inline-only-for-width-1 rule is enforceable — the skill or lint can reject `inline` on a parallel batch before dispatch. | Mark a task in a width-2 batch `inline`; confirm it errors with a clear message. |
| Should-be-true | A per-task `**Execution:**` field fits the Plan schema and lint without disrupting existing plans (absent field = `auto` = today's behavior). | Lint existing plans that have no field; confirm they resolve to `auto`/subagent unchanged. |
| Should-be-true | Inline tasks' context cost is acceptable when used sparingly (a few per plan). | Run a plan with 2–3 inline tasks; observe main-session context growth against the budget. |
| Might-be-true | `auto` should sometimes resolve a width-1 batch to inline on its own (e.g. tiny doc edits), rather than always defaulting to subagent unless explicitly inline. | Revisit at specify with real plans; compare default-subagent vs heuristic-inline. |


## SpecScore Integration

- **New Features this would create:** an `implement-per-task-execution-mode` Feature defining the `**Execution:**` field, its values (`inline` / `subagent` / `auto`), the `inline` ⟺ width-1 constraint, and how implement honors each.
- **Existing Features affected:** `specstudio-implement-skill` (dispatch logic gains an inline path; generalizes single-pass execution from whole-run to per-task); `specstudio-plan-skill` (the Plan schema and lint gain the per-task field); `implement-workflow-execution-engine` (its width-1 branch hands off to this Idea's inline/subagent choice).
- **Dependencies:** pairs with `implement-workflow-execution-engine` at the width-1 boundary — neither blocks the other; they meet there. Interacts with the cross-repo `plan-granularity-improvement` Idea (specscore): that Idea's per-task model tier is required for subagent/workflow execution and uses `inherit` for inline tasks — i.e., this Idea's execution mode determines model requiredness. Independent of `detached-background-implement` and `approval-autonomy`.

## Open Questions

- Field name and vocabulary: `**Execution:** inline | subagent | auto`, or different terms? Where exactly in the per-task block does it sit?
- Should `auto` ever pick `inline` autonomously (tiny tasks), or is `inline` strictly opt-in?
- Does execution mode interact with `commit_cadence` — e.g., inline tasks fit task-cadence commits naturally since the main agent already holds the diff?
- For Feature/Idea-sourced single-pass modes (already inline by nature), is `**Execution:**` a no-op, or could a single-pass run dispatch a heavy task to a subagent?
- Model coupling: should the `**Execution:**` field and `plan-granularity-improvement`'s model tier be validated together by one lint rule (inline ⟹ `inherit`; subagent/workflow ⟹ a concrete tier), even though they live in different repos?

---
*This document follows the https://specscore.md/idea-specification*
