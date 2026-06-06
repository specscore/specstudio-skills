---
type: rehearse-stub
stub_status: pending
ac: invocation-rejects-per-slug-argument
feature: sidekick-consilium
format: https://specscore.md/scenario-specification
---

# Rehearse: invocation-rejects-per-slug-argument

## Scenario (from AC)

**Given** a project with multiple queued tasks
**When** the user invokes `/consilium <some-slug>`
**Then** the skill exits non-zero with the error message `"per-slug invocation not supported in Phase 1; /consilium drains all queued tasks"`; no task is claimed.

## Verification approach

Run `/consilium some-slug` against a fixture project with at least one queued task; assert non-zero exit code and exact error string on stderr. Confirm via `specscore:task` listing that the queued task's state is unchanged (no claim transition occurred).

---
*This document follows the https://specscore.md/scenario-specification*
