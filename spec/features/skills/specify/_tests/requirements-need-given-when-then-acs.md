---
format: https://specscore.md/scenario-specification
---

# Scenario: Every requirement has at least one Given/When/Then acceptance criterion

**Validates:** [specify#req:ac-format](../README.md#req-ac-format)

## Steps

GIVEN the skill is authoring a Feature with one requirement file `spec/features/foo/requirements/bar.md`
AND the requirement file body contains zero AC blocks
WHEN `specscore spec lint` runs
THEN lint reports rule `F-003` (requirement missing AC) as an error
AND the skill MUST NOT proceed past the hard gate

GIVEN a requirement contains an AC written as prose ("The system should reject invalid tokens")
AND the AC is NOT in `Given / When / Then` form
WHEN `specscore spec lint` runs
THEN lint reports rule `F-004` (AC not in G/W/T) as an error
AND the skill surfaces the violation with the offending AC text

GIVEN a requirement contains an AC where the `Then` clause is unobservable ("Then the user feels confident")
WHEN the skill's inline self-review runs
THEN the skill surfaces the AC as ambiguous
AND the skill prompts the user to sharpen it to an observable outcome
AND the skill MUST NOT advance to the reviewer subagent until the AC is observable

GIVEN a requirement has multiple ACs all in proper G/W/T form
WHEN `specscore spec lint` runs
THEN lint exits zero for that requirement
AND each AC's `Then` clause names a specific, checkable observation

---
*This document follows the https://specscore.md/scenario-specification*
