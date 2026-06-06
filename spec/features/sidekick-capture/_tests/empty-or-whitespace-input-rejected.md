---
type: rehearse-stub
stub_status: pending
ac: empty-or-whitespace-input-rejected
feature: sidekick-capture
format: https://specscore.md/scenario-specification
---

# Rehearse: empty-or-whitespace-input-rejected

## Scenario (from AC)

**Given** a Claude Code session
**When** the user invokes `/sidekick` with no argument, or with only whitespace
**Then** the skill exits with a clear error indicating an empty one-liner and the 1–500-character constraint; no seed file is created; no event is emitted.

## Verification approach

Invoke `specstudio:sidekick ""` and `specstudio:sidekick "   "`; assert non-zero exit; assert no file was created under `spec/ideas/seeds/`; assert `.specscore/events.jsonl` line count is unchanged.

---
*This document follows the https://specscore.md/scenario-specification*
