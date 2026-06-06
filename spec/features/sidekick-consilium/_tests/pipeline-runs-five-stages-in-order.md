---
type: rehearse-stub
stub_status: pending
ac: pipeline-runs-five-stages-in-order
feature: sidekick-consilium
format: https://specscore.md/scenario-specification
---

# Rehearse: pipeline-runs-five-stages-in-order

## Scenario (from AC)

**Given** one queued `consilium-review` task with a valid seed
**When** the skill processes the task
**Then** the task's transcript shows exactly: (1) CLI gather invoked, (2) one researcher Agent call, (3) N parallel role-agent calls (N = active roster size), (4) one `specscore consilium verdict` invocation, (5) one scribe Agent call — in that order; out-of-order or skipped stages are a regression.

## Verification approach

Seed a fixture project with one queued `consilium-review` task; run `/consilium`; load the completed task's `pipeline_transcript` field and assert the stage records appear in the exact order `gather, researcher, panel, arbiter, scribe`. Assert the `panel` stage's per-role entry count equals the active roster size.

---
*This document follows the https://specscore.md/scenario-specification*
