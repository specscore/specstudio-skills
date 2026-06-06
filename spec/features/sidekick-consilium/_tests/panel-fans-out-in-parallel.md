---
type: rehearse-stub
status: pending
ac: panel-fans-out-in-parallel
feature: sidekick-consilium
format: https://specscore.md/scenario-specification
---

# Rehearse: panel-fans-out-in-parallel

## Scenario (from AC)

**Given** a 9-role active roster
**When** the skill processes one queued task
**Then** the panel stage dispatches 9 Agent tool uses in a single message (parallel invocation); the wall-clock time for the panel stage is approximately max(role-agent-time), not sum(role-agent-time).

## Verification approach

Run the pipeline with the default 9-role roster against a fixture seed; inspect the orchestrator's recorded transcript and assert the panel stage emits 9 Agent tool uses in one assistant message (not 9 sequential messages). Compute panel-stage wall-clock from `pipeline_transcript.panel.started_at` and `ended_at` and assert it is closer to max(per-role-duration) than to sum(per-role-duration).

---
*This document follows the https://specscore.md/scenario-specification*
