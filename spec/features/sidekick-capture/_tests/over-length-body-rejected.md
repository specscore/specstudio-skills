---
type: rehearse-stub
status: pending
ac: over-length-body-rejected
feature: sidekick-capture
---

# Rehearse: over-length-body-rejected

## Scenario (from AC)

**Given** a Claude Code session
**When** the user invokes `/sidekick` with a valid one-liner and an optional body that, combined with the H1 line, produces total body content of 2001 or more characters
**Then** the skill exits with an error indicating the 2000-character body limit; the over-length body is not silently truncated; no seed file is created; no event is emitted.

## Verification approach

Invoke with a valid one-liner and `--body` content that pushes the total body length to 2001 chars; assert non-zero exit with the 2000-char body message; assert no file or event change.

---
*This document follows the https://specscore.md/scenario-specification*
