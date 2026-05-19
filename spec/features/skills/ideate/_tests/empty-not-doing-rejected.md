# Scenario: Empty Not-Doing list is rejected

**Validates:** [ideate#req:not-doing-required](../README.md#req-not-doing-required)

## Steps

GIVEN the skill is in Phase 3 and is about to crystallize an Idea artifact
AND the user has not provided any explicit "Not Doing" exclusions
WHEN the skill writes `spec/ideas/my-idea.md`
THEN the artifact's `Not Doing (and Why)` section is non-empty
AND `specscore spec lint` does not report rule `I-002`

GIVEN the user manually edits the artifact and removes all entries from `Not Doing (and Why)`
WHEN the user re-runs `specscore spec lint`
THEN lint reports rule `I-002` as an error
AND lint exits non-zero
AND the `hard-gate` AC blocks any downstream `specstudio:specify` invocation

---
*This document follows the https://specscore.md/scenario-specification*
