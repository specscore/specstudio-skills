---
type: rehearse-stub
status: pending
ac: roster-violating-group-floor-rejected
feature: sidekick-consilium
---

# Rehearse: roster-violating-group-floor-rejected

## Scenario (from AC)

**Given** a `specscore.yaml` with `consilium.roster.exclude: [yagni-cop, skeptic, security-ops]` (excluding all 3 default adversaries) and no custom adversary added
**When** the skill is invoked
**Then** roster validation fails with error `"adversaries group has 0 members; ≥1 required"`; no task is claimed; the skill exits non-zero.

## Verification approach

Configure a fixture `specscore.yaml` excluding all three default adversaries with no custom adversary defined; run `/consilium` against a fixture project with a queued task. Assert non-zero exit code, the stderr contains the exact error string `"adversaries group has 0 members; ≥1 required"`, and the queued task remains unclaimed.

---
*This document follows the https://specscore.md/scenario-specification*
