---
format: https://specscore.md/scenario-specification
---

# Scenario: Hard gate blocks specify until lint and user approval

**Validates:** [ideate#ac:hard-gate-enforced](../README.md#ac-hard-gate-enforced)

## Steps

GIVEN the user has invoked `specstudio:ideate` and the skill has written `spec/ideas/my-idea.md`
AND `specscore spec lint` exits non-zero (an Idea-specific rule fails)
WHEN the skill or the user attempts to invoke `specstudio:specify` against the same Idea
THEN the skill refuses to invoke `specstudio:specify`
AND the skill surfaces the specific lint violation to the user
AND no `idea.approved` event is emitted

GIVEN `specscore spec lint` now passes for the same Idea
AND the user has NOT yet approved the Recommended Direction
WHEN the skill or the user attempts to invoke `specstudio:specify`
THEN the skill still refuses to invoke `specstudio:specify`
AND the skill prompts the user for explicit approval

GIVEN lint passes AND the user has explicitly approved the Recommended Direction
WHEN the user invokes `specstudio:specify`
THEN the hand-off proceeds

---
*This document follows the https://specscore.md/scenario-specification*
