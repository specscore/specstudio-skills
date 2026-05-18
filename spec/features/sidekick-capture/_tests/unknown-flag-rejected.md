---
type: rehearse-stub
status: pending
ac: unknown-flag-rejected
feature: sidekick-capture
---

# Rehearse: unknown-flag-rejected

## Scenario (from AC)

**Given** a Claude Code session
**When** the user invokes `/sidekick --review the one-liner here`
**Then** the skill exits with `"unknown flag --review"` rather than treating `--review` as part of the one-liner; no seed file is created; no event is emitted.

## Verification approach

Invoke `specstudio:sidekick --review "x"`; assert non-zero exit with `Unknown flag: --review`; assert no file or event change.

---
*This document follows the https://specscore.md/scenario-specification*
