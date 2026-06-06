---
type: rehearse-stub
stub_status: pending
ac: seed-mutation-blocks-review
feature: sidekick-consilium
format: https://specscore.md/scenario-specification
---

# Rehearse: seed-mutation-blocks-review

## Scenario (from AC)

**Given** a queued `consilium-review` task whose stored `content_hash` does not match the current seed file's normalized one-liner hash
**When** the skill attempts to claim the task
**Then** the task transitions `queued → failed` with reason `seed-mutated`; no review proceeds; no verdict is produced; the operator sees a single short error line referencing the seed path.

## Verification approach

Create a queued task with a stored `content_hash`, then mutate the seed file's H1 one-liner so its normalized hash diverges from the stored value; run `/consilium` and assert the task transitions to `failed` with reason `seed-mutated`. Assert no verdict is written to the task payload and the operator-visible stderr line names the seed path.

---
*This document follows the https://specscore.md/scenario-specification*
