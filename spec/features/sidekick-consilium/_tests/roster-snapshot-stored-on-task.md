---
type: rehearse-stub
stub_status: pending
ac: roster-snapshot-stored-on-task
feature: sidekick-consilium
format: https://specscore.md/scenario-specification
---

# Rehearse: roster-snapshot-stored-on-task

## Scenario (from AC)

**Given** a completed `consilium-review` task and a subsequent `specscore.yaml` change that excludes one of the originally-active roles
**When** the task payload is inspected after the config change
**Then** the task's `roster_snapshot` field still reflects the roster active at review time (the excluded role is still listed); the task is NOT invalidated by the config change.

## Verification approach

Run the pipeline once against a fixture queued task with the default roster; record the resulting `roster_snapshot`; then mutate `specscore.yaml` to exclude one of the previously-active roles. Re-read the completed task and assert `roster_snapshot` is byte-identical to the pre-mutation snapshot and the task's status remains `complete`.

---
*This document follows the https://specscore.md/scenario-specification*
