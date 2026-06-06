---
format: https://specscore.md/scenario-specification
---

# Scenario: After feature.approved, the only sanctioned transition is to writing-plans

**Validates:** [specify#ac:promotion-boundary-held](../README.md#ac-promotion-boundary-held)

## Steps

GIVEN the user has approved a Feature and `feature.approved` has fired
WHEN the user (or an upstream orchestrator) asks the skill to transition next
THEN the skill MUST transition to `writing-plans`
AND the skill MUST NOT invoke `frontend-design`, `mcp-builder`, or any other implementation skill directly

GIVEN the user explicitly requests `specstudio:specify` invoke `frontend-design` immediately after approval
WHEN the skill processes the request
THEN the skill refuses
AND the skill explains that `writing-plans` is the only sanctioned next step
AND the skill suggests running `writing-plans` first to produce a Plan, after which implementation skills become valid

GIVEN no `writing-plans` skill is available in the environment
WHEN the skill is asked to transition
THEN the skill MUST surface the missing-skill condition to the user
AND the skill MUST NOT silently substitute a different downstream skill

---
*This document follows the https://specscore.md/scenario-specification*
