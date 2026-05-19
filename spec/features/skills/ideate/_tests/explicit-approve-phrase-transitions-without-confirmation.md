# Scenario: Explicit "approve" phrase transitions status without confirmation prompt

**Validates:** [ideate#ac:approval-detection](../README.md#ac-approval-detection)

## Steps

GIVEN an Idea at `spec/ideas/my-idea.md` with `**Status:** Draft` body metadata
AND the skill has presented the lint-clean artifact for user review
WHEN the user responds with the standalone phrase `approve` (or `Approved`, `APPROVE`, `approve.`, etc.)
THEN the skill recognizes the response as an explicit approval
AND the skill does NOT ask for additional confirmation
AND the skill transitions the `**Status:**` body-metadata value from `Draft` to `Approved`
AND the skill re-runs `specscore spec lint`
AND the skill emits `idea.approved`

GIVEN the user response contains `approve` as the dominant content of a short message (e.g., "approve, looks great")
WHEN the skill processes the response
THEN the skill treats it as explicit approval and proceeds without confirmation

GIVEN the user response contains `approve` only incidentally inside a longer message (e.g., "I think we should approve this approach but I have one concern: ...")
WHEN the skill processes the response
THEN the skill MUST NOT treat this as explicit approval
AND the skill MUST address the user's concern before any approval flow

---
*This document follows the https://specscore.md/scenario-specification*
