---
type: sidekick-seed
slug: implement-skill-checklist-missing-plan-body-status-transition
captured_at: 2026-05-19T19:01:10Z
captured_by: user
captured_during: spec/features/skills/implement
trigger: explicit
status: completed
synchestra_task: null
---

# implement skill: per-batch checklist is missing the Plan body-metadata Status transition step (Approved→Implementing on first dispatch; Implementing→Completed on final task)

## Observed problem (dogfood finding #2)

First end-to-end dogfood of `specstudio:implement` (against `spec/plans/dogfood-test.md`): the agent missed the Plan body-metadata `**Status:** Approved → Implementing` transition that should fire on first-task dispatch. The Status stayed `Approved` through batch 1 and was only flipped to `Completed` in batch 2 — skipping `Implementing` entirely.

## Root cause

`REQ:implement-status-transition` specifies both transitions correctly. But `skills/implement/SKILL.md` step 4 says only "Stage Status writes. As each subagent is dispatched, transition that task's `**Status:** pending → in-progress`." — it covers **task** Status writes but NOT the **Plan body-metadata** Status transition. The contract is in the REQ; the where-in-the-flow step is missing from the operational checklist.

## Suggested fix

In `skills/implement/SKILL.md`:

1. Split step 4 into 4a (task Status) and 4b (Plan body Status: `Approved → Implementing` iff first task dispatch AND Plan currently `Approved`).
2. Add a final-batch step: "When all tasks reach `**Status:** done`, transition Plan body `**Status:** Implementing → Completed`."
3. Tighten `REQ:implement-status-transition` to reference the checklist steps owning each transition.

## Why this matters

Plan body Status is the at-a-glance signal for outside observers (Hub, future verify/review). Skipping intermediates makes history less debuggable.

See commits `52c4a80` (missed) and `46612aa` (remedial).
