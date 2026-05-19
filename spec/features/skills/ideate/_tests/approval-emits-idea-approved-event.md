# Scenario: User approval transitions status and emits idea.approved

**Validates:** [ideate#ac:lifecycle-events](../README.md#ac-lifecycle-events)

## Steps

GIVEN the skill has written `spec/ideas/my-idea.md` for the first time
AND `specscore spec lint` exits zero on that artifact
WHEN the first-write completes
THEN the skill emits `idea.drafted`
AND the artifact's `**Status:**` body metadata is `Draft`

GIVEN the skill has presented the lint-clean artifact to the user for review
WHEN the user explicitly approves the Recommended Direction
THEN the skill updates the `**Status:**` body metadata from `Draft` to `Approved`
AND the skill re-runs `specscore spec lint`
AND lint still exits zero
AND the skill emits `idea.approved`

GIVEN the user has NOT yet explicitly approved
WHEN the skill is asked to advance status
THEN the skill refuses
AND no `idea.approved` event is emitted
AND the `**Status:**` body-metadata value remains `Draft`

---
*This document follows the https://specscore.md/scenario-specification*
