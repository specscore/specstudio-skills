---
type: sidekick-seed
slug: implement-skill-checklist-missing-plan-body-status-transition
captured_at: 2026-05-19T19:01:10Z
captured_by: user
captured_during: spec/features/skills/implement
trigger: explicit
status: queued
synchestra_task: null
---

# implement skill: per-batch checklist is missing the Plan body-metadata Status transition step (Approved→Implementing on first dispatch; Implementing→Completed on final task)

## Observed problem (dogfood finding #2)

During the first end-to-end dogfood of `specstudio:implement` (against `spec/plans/dogfood-test.md`), the agent missed the Plan body-metadata `**Status:**` transition `Approved → Implementing` that should have fired on first-task dispatch (batch 1, task 1). The Plan body-metadata Status stayed `Approved` through batch 1's commit and was only transitioned to `Completed` in batch 2 — skipping `Implementing` entirely.

## Root cause

`REQ:implement-status-transition` in `spec/features/skills/implement/README.md` specifies the transitions correctly:
> Update the Plan's body-metadata `**Status:**` from `Approved → Implementing` on first task dispatch (if not already there), and from `Implementing → Completed` when the final task lands.

But the operational checklist in `skills/implement/SKILL.md` step 4 says only:
> "Stage Status writes. As each subagent is dispatched, transition that task's `**Status:** pending → in-progress` on the Plan file."

The checklist mentions **task** Status writes but does NOT explicitly call out the **Plan body-metadata** Status transition as a discrete step. The contract is in the REQ but the where-in-the-flow operationalization is missing.

## Suggested fix

In `skills/implement/SKILL.md`:

1. Split step 4 into 4a (task Status writes) and 4b (Plan body-metadata Status: `Approved → Implementing` if and only if this is the first task dispatch in the current invocation AND the Plan is currently at `Approved`).
2. Add a new final-batch step after step 17 (loop back): "When all tasks reach `**Status:** done`, transition Plan body-metadata `**Status:** Implementing → Completed` and stage."
3. Tighten `REQ:implement-status-transition` in `spec/features/skills/implement/README.md` to reference the specific checklist steps that own each transition.

## Why this matters

The Plan body-metadata Status field is the at-a-glance signal for outside observers (Hub, future verify/review skills, humans browsing `spec/plans/`). Skipping intermediate states makes the Plan's history less debuggable and breaks the "every state-change is observable" discipline the rest of the SpecScore lifecycle maintains.

Captured during the dogfood of implement against `spec/plans/dogfood-test.md`; see commit messages `52c4a80` and `46612aa` for the missing and remedial transitions respectively.
