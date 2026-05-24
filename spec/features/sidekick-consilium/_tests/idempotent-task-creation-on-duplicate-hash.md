---
type: rehearse-stub
status: pending
ac: idempotent-task-creation-on-duplicate-hash
feature: sidekick-consilium
---

# Rehearse: idempotent-task-creation-on-duplicate-hash

## Scenario (from AC)

**Given** a queued `consilium-review` task for seed slug `X` and a second `sidekick-idea.captured` event arriving with the same `content_hash`
**When** the second event is processed
**Then** no second task is created; the existing task remains in its current state; the event is acknowledged without duplication.

## Verification approach

Seed a queued `consilium-review` task for slug `X` with a known `content_hash`; emit a second `sidekick-idea.captured` event carrying the same hash; trigger event processing. Assert via `specscore:task` listing that the queued-task count for slug `X` is still 1 and its state is unchanged, and confirm the second event was acknowledged (no error surfaced).

---
*This document follows the https://specscore.md/scenario-specification*
