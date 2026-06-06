---
type: rehearse-stub
status: pending
ac: event-emitted-only-on-successful-write
feature: sidekick-capture
format: https://specscore.md/scenario-specification
---

# Rehearse: event-emitted-only-on-successful-write

## Scenario (from AC)

**Given** a filesystem state where `spec/ideas/seeds/` cannot be created (e.g., read-only parent) or written to (e.g., disk full)
**When** the skill is invoked with a valid one-liner
**Then** the skill reports the write failure with a clear error; no `sidekick-idea.captured` event is emitted; the skill exits non-zero.

## Verification approach

Chmod `spec/ideas/` to read-only; invoke; assert write fails with a clear error; assert `.specscore/events.jsonl` line count is unchanged.

---
*This document follows the https://specscore.md/scenario-specification*
