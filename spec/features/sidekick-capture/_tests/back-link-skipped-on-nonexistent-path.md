---
type: rehearse-stub
status: pending
ac: back-link-skipped-on-nonexistent-path
feature: sidekick-capture
---

# Rehearse: back-link-skipped-on-nonexistent-path

## Scenario (from AC)

**Given** a `/sidekick` invocation with `captured_during: spec/features/nonexistent-feature`
**When** the skill is invoked
**Then** the skill writes the seed and emits the event normally; no back-link write is attempted; the skill's exit code is 0 (the absent source artifact is not an error condition).

## Verification approach

Invoke with `captured_during=spec/features/this-feature-does-not-exist`. Assert: seed exists, event emitted, exit code 0, no back-link write attempted, no error reported.

---
*This document follows the https://specscore.md/scenario-specification*
