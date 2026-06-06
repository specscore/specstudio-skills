---
type: rehearse-stub
status: pending
ac: pipeline-transcript-payload-shape
feature: sidekick-consilium
format: https://specscore.md/scenario-specification
---

# Rehearse: pipeline-transcript-payload-shape

## Scenario (from AC)

**Given** one completed `consilium-review` task
**When** the task's `pipeline_transcript` field is inspected
**Then** it contains exactly five stage records in the order `gather, researcher, panel, arbiter, scribe`; each stage record has `started_at`, `ended_at`, and `outcome: ok`; the `panel` stage's record contains one per-role entry per active roster role; each per-role entry has `role`, `input_includes_briefing: true`, `tool_calls` (a list, possibly empty), and a `vote` parsed as YAML matching REQ `vote-schema`.

## Verification approach

Run the pipeline against a fixture queued task; load the completed task's `pipeline_transcript`; assert the five stage records appear in the specified order with the required envelope fields. Iterate the `panel` stage's per-role entries and assert each has `role`, `input_includes_briefing: true`, a list-valued `tool_calls`, and a `vote` that parses as YAML conforming to the vote schema (verdict, confidence, rationale).

---
*This document follows the https://specscore.md/scenario-specification*
