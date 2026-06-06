---
format: https://specscore.md/scenario-specification
---

# Scenario: Hard gate blocks writing-plans until lint, reviewer, and user all pass

**Validates:** [specify#ac:hard-gate-enforced](../README.md#ac-hard-gate-enforced)

## Steps

GIVEN the user has invoked `specstudio:specify` and the skill has written `spec/features/checkout-v2/README.md` plus `spec/features/checkout-v2/requirements/payment.md`
AND `specscore spec lint` exits non-zero (one AC is not in G/W/T form)
WHEN the skill or the user attempts to invoke `writing-plans` against the same Feature
THEN the skill refuses to invoke `writing-plans`
AND the skill surfaces the specific lint violation
AND no `feature.specified` event is emitted

GIVEN lint now passes for the same Feature
AND the reviewer subagent has NOT yet been dispatched
WHEN the skill or the user attempts to invoke `writing-plans`
THEN the skill still refuses
AND the skill dispatches the reviewer subagent first

GIVEN lint passes AND the reviewer subagent returned `Approved`
AND the user has NOT yet explicitly approved
WHEN the skill or the user attempts to invoke `writing-plans`
THEN the skill still refuses
AND the skill prompts the user for explicit approval

GIVEN all five hard-gate conditions hold (Feature exists with requirements, ACs are G/W/T, lint passes, reviewer approved, user approved)
WHEN the user invokes `writing-plans`
THEN the transition proceeds

---
*This document follows the https://specscore.md/scenario-specification*
