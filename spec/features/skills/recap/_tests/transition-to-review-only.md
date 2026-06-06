---
format: https://specscore.md/scenario-specification
---

# Scenario: after a successful run the skill transitions only to specstudio:review and to no other skill

**Validates:** [recap#ac:transition-to-review-only](../README.md#ac-transition-to-review-only-verifies-reqtransition-to-review-reqhard-gate)

## Steps

GIVEN an Approved Feature `example` with at least one AC whose drift-narrator subagent returns `no-drift`
AND the report has been written and staged
AND the `recap.completed` event has been emitted
WHEN the skill prepares to transition
THEN the only skill name the skill offers or invokes is `specstudio:review`
AND when `specstudio:review` is unshipped, the skill hands back to the user with the report path and a recommendation to inspect drift items manually
AND the skill does NOT invoke `specstudio:ideate`
AND the skill does NOT invoke `specstudio:specify`
AND the skill does NOT invoke `specstudio:plan`
AND the skill does NOT invoke `specstudio:implement`
AND the skill does NOT invoke `specstudio:verify`
AND the skill does NOT invoke `specstudio:ship`
AND the skill does NOT invoke `writing-plans`
AND the skill does NOT invoke `frontend-design`
AND the skill does NOT invoke `mcp-builder`

---
*This document follows the https://specscore.md/scenario-specification*
