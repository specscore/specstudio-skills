# Scenario: Post-approval edits emit idea.updated, never re-emit idea.drafted or idea.approved

**Validates:** [ideate#ac:lifecycle-events](../README.md#ac-lifecycle-events)

## Steps

GIVEN an Idea at `spec/ideas/my-idea.md` with `**Status:** Approved` body metadata
AND `idea.approved` was previously emitted exactly once
WHEN the user edits the artifact (e.g., refines the Recommended Direction)
AND `specscore spec lint` passes after the edit
THEN the skill emits `idea.updated`
AND the skill does NOT emit `idea.drafted`
AND the skill does NOT re-emit `idea.approved`
AND the `**Status:**` body-metadata value remains `Approved`

GIVEN the same Approved Idea is edited multiple times in succession, each edit followed by a passing lint
WHEN the skill processes each edit
THEN the skill emits `idea.updated` exactly once per successful lint pass
AND the count of `idea.approved` events remains exactly 1 across all edits

---
*This document follows the https://specscore.md/scenario-specification*
