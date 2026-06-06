---
type: rehearse-stub
status: pending
ac: malformed-vote-fails-pipeline
feature: sidekick-consilium
format: https://specscore.md/scenario-specification
---

# Rehearse: malformed-vote-fails-pipeline

## Scenario (from AC)

**Given** a fixture role agent that returns a vote with an invalid `verdict` value (e.g., `verdict: maybe`)
**When** the pipeline reaches the arbiter
**Then** the arbiter returns non-zero exit; the task transitions to `failed` with reason `malformed-vote`; no verdict is written.

## Verification approach

Substitute one role agent with a fixture that returns `verdict: maybe`; run the pipeline against a queued task; assert `specscore consilium verdict` exits non-zero, the task's final state is `failed` with reason `malformed-vote`, and no `verdict` field is set on the task payload.

---
*This document follows the https://specscore.md/scenario-specification*
