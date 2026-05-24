---
type: rehearse-stub
status: pending
ac: token-usage-recorded-on-task
feature: sidekick-consilium
---

# Rehearse: token-usage-recorded-on-task

## Scenario (from AC)

**Given** a completed `consilium-review` task
**When** the task payload is inspected
**Then** the payload contains `tokens_total: <int>` reflecting the actual sum of tokens consumed across the researcher, panel, and scribe calls; the value is non-zero.

## Verification approach

Run `/consilium` against a fixture project with one queued task; after completion, load the task payload via `specscore:task` and assert `tokens_total` is present, is a non-negative integer, and is non-zero. Cross-check the value matches the sum of per-stage token counts captured in `pipeline_transcript`.

---
*This document follows the https://specscore.md/scenario-specification*
