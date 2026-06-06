---
format: https://specscore.md/scenario-specification
---

# Scenario: after a successful run the skill transitions only to specstudio:recap and to no other skill

**Validates:** [verify#ac:transition-to-recap-only](../README.md#ac-transition-to-recap-only-verifies-reqtransition-to-recap-reqhard-gate)

## Steps

GIVEN an approved Feature `example` with at least one AC whose verifier returns `pass`
AND the report has been written and staged
AND the `verify.completed` event has been emitted
WHEN the skill prepares to transition
THEN the only skill name the skill offers or invokes is `specstudio:recap`
AND when `specstudio:recap` is unshipped the skill hands back to the user with the report path and a recommendation to review verdicts manually
AND the skill does NOT invoke `specstudio:ideate`
AND the skill does NOT invoke `specstudio:specify`
AND the skill does NOT invoke `specstudio:plan`
AND the skill does NOT invoke `specstudio:implement`
AND the skill does NOT invoke `specstudio:review`
AND the skill does NOT invoke `specstudio:ship`
AND the skill does NOT invoke `writing-plans`
AND the skill does NOT invoke `frontend-design`
AND the skill does NOT invoke `mcp-builder`

---
*This document follows the https://specscore.md/scenario-specification*
