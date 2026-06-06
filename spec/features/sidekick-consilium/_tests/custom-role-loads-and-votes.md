---
type: rehearse-stub
stub_status: pending
ac: custom-role-loads-and-votes
feature: sidekick-consilium
format: https://specscore.md/scenario-specification
---

# Rehearse: custom-role-loads-and-votes

## Scenario (from AC)

**Given** a `specscore.yaml` with `consilium.roster.custom` listing one valid custom role at `.specscore/roles/accessibility.md` (markdown file with `**Name:** accessibility`, `**Group:** customers`, `**Output Schema Version:** 1`, a `## Role Prompt` section, and a `## Example Vote` section)
**When** the skill loads the roster and processes a queued task
**Then** the active roster has 10 roles (9 defaults + 1 custom); the custom role's agent is dispatched alongside the others; its vote is included in the arbiter's denominator for the customers group.

## Verification approach

Author a fixture `.specscore/roles/accessibility.md` matching the contract and register it under `consilium.roster.custom`; run `/consilium` against a queued task. Assert `roster_snapshot` on the completed task has 10 entries including `accessibility` in the customers group, the panel transcript shows a dispatch for `accessibility`, and the arbiter's `denominators.customers` counts the new role.

---
*This document follows the https://specscore.md/scenario-specification*
