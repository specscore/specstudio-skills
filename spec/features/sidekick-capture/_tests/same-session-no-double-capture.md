---
type: rehearse-stub
stub_status: pending
ac: same-session-no-double-capture
feature: sidekick-capture
format: https://specscore.md/scenario-specification
---

# Rehearse: same-session-no-double-capture

## Scenario (from AC)

**Given** a host skill that has already invoked `/sidekick` with one-liner L in the current conversation and received seed path P
**When** the host encounters a cue that would re-fire `/sidekick` with the same L
**Then** the host does not re-invoke `/sidekick`; it may reference the existing seed P in passing without re-writing.

## Verification approach

Drive a host session (`specstudio:specify`) through a transcript that contains two identical sideline cues; assert exactly one capture invocation in the transcript; assert the second occurrence is mentioned without re-invoking. (Multi-turn agent behavior; manual review of transcript is acceptable for Phase 0.)

---
*This document follows the https://specscore.md/scenario-specification*
