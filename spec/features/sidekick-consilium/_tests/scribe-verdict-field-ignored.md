---
type: rehearse-stub
status: pending
ac: scribe-verdict-field-ignored
feature: sidekick-consilium
---

# Rehearse: scribe-verdict-field-ignored

## Scenario (from AC)

**Given** a fixture scribe agent that emits a `verdict:` field in its response in addition to the prose paragraph
**When** the pipeline completes
**Then** the task's stored verdict is the arbiter's value, not the scribe's; the scribe's `verdict:` field is silently ignored at parse time.

## Verification approach

Substitute a fixture scribe stub that appends a contradictory `verdict:` field to its response; run the pipeline with arbiter inputs known to produce `should-implement`; assert the completed task's `verdict` field equals the arbiter's value, not the scribe's contradictory value. Assert no error is logged for the extra field.

---
*This document follows the https://specscore.md/scenario-specification*
