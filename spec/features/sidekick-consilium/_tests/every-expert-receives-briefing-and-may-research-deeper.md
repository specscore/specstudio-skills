---
type: rehearse-stub
stub_status: pending
ac: every-expert-receives-briefing-and-may-research-deeper
feature: sidekick-consilium
format: https://specscore.md/scenario-specification
---

# Rehearse: every-expert-receives-briefing-and-may-research-deeper

## Scenario (from AC)

**Given** a completed pipeline run
**When** the panel agents' invocation transcripts are inspected
**Then** every role agent's input includes the briefing pack; at least one role agent's transcript also shows tool calls (Read/Grep/Glob) beyond the briefing — confirming the briefing is a floor, not a ceiling.

## Verification approach

Run the pipeline against a fixture seed that has supporting code paths inviting follow-up Read/Grep; iterate over each per-role entry in `pipeline_transcript.panel` and assert `input_includes_briefing: true` for every role. Assert at least one role's `tool_calls` list is non-empty with a Read, Grep, or Glob invocation.

---
*This document follows the https://specscore.md/scenario-specification*
