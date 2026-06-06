---
type: rehearse-stub
status: pending
ac: verdict-task-payload-completeness
feature: sidekick-consilium
format: https://specscore.md/scenario-specification
---

# Rehearse: verdict-task-payload-completeness

## Scenario (from AC)

**Given** one completed `consilium-review` task
**When** the task payload is inspected
**Then** it contains: `roster_snapshot` (list of role slugs + groups), `votes` (one YAML vote per role in the snapshot), `briefing_pack` (the researcher's output), `rule_trace` (the arbiter's trace), `verdict` (one of three enum values), `content_hash` (matches the seed at review time), `tokens_total` (non-zero int), and `scribe_summary` (the prose paragraph, ≤500 chars).

## Verification approach

Run the pipeline against a fixture queued task; load the completed task payload; assert presence and basic shape of every required field: `roster_snapshot`, `votes` (one entry per roster member), `briefing_pack`, `rule_trace`, `verdict` (must be one of the three enum values), `content_hash` (matches the recomputed seed hash), `tokens_total` > 0, and `scribe_summary` length ≤ 500.

---
*This document follows the https://specscore.md/scenario-specification*
