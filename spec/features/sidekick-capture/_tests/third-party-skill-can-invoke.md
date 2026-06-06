---
type: rehearse-stub
stub_status: pending
ac: third-party-skill-can-invoke
feature: sidekick-capture
format: https://specscore.md/scenario-specification
---

# Rehearse: third-party-skill-can-invoke

## Scenario (from AC)

**Given** a third-party host skill (e.g., a fictional `agent-skills:build`) that follows the adoption contract and invokes `specstudio:sidekick` with a valid one-liner and a `captured_by` of `"agent-skills:build"`
**When** the skill executes
**Then** the seed is written exactly as it would be for a first-party caller; `captured_by` in the frontmatter and event payload is `"agent-skills:build"` verbatim; no special handling distinguishes first-party from third-party callers in the on-disk artifact.

## Verification approach

From a fixture third-party skill (mocked), invoke `specstudio:sidekick` with `captured_by="agent-skills:build"`; assert the seed's frontmatter `captured_by` is verbatim `agent-skills:build`; assert the emitted event's `actor.id` reflects the same value.

---
*This document follows the https://specscore.md/scenario-specification*
