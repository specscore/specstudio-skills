---
type: rehearse-stub
status: pending
ac: over-length-input-rejected
feature: sidekick-capture
---

# Rehearse: over-length-input-rejected

## Scenario (from AC)

**Given** a Claude Code session
**When** the user invokes `/sidekick` with a one-liner of 501 or more characters (after trimming)
**Then** the skill exits with an error indicating the 500-character limit; the over-length text is not silently truncated; no seed file is created; no event is emitted.

## Verification approach

Invoke with a fixture one-liner of 501 chars; assert non-zero exit with the 500-char message; assert no file or event change.

---
*This document follows the https://specscore.md/scenario-specification*
