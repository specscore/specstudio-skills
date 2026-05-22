---
type: rehearse-stub
status: pending
ac: back-link-skipped-on-null-captured-during
feature: sidekick-capture
---

# Rehearse: back-link-skipped-on-null-captured-during

## Scenario (from AC)

**Given** a `/sidekick` invocation with `captured_during: null`
**When** the skill writes the seed successfully
**Then** no file other than the seed file is modified; the `sidekick-idea.captured` event still fires; no error or warning about a missing source artifact is reported.

## Verification approach

Invoke with `captured_during=null`. Assert: seed exists, `.specscore/events.jsonl` line added, NO other file modified, exit code 0, no warning about a missing source artifact.

---
*This document follows the https://specscore.md/scenario-specification*
