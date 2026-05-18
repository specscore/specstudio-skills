---
type: rehearse-stub
status: pending
ac: default-roster-is-9-roles-in-three-groups
feature: sidekick-consilium
---

# Rehearse: default-roster-is-9-roles-in-three-groups

## Scenario (from AC)

**Given** a fresh project with no `consilium.roster` configuration in `specscore.yaml`
**When** the skill loads the active roster
**Then** the roster contains exactly 9 roles: `engineer, architect, qa` (builders); `pm, ux, marketing` (customers); `yagni-cop, skeptic, security-ops` (adversaries); each role's markdown file is at `skills/consilium/roles/<role>.md`.

## Verification approach

In a fixture project with a minimal `specscore.yaml` containing no `consilium.roster` block, invoke the skill's roster-load step; assert the resulting active roster has exactly 9 entries with the expected slugs and group assignments. Assert each role's markdown file resolves to `skills/consilium/roles/<role>.md`.

---
*This document follows the https://specscore.md/scenario-specification*
