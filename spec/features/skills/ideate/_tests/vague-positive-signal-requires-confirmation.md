---
format: https://specscore.md/scenario-specification
---

# Scenario: Vague positive signals require explicit confirmation before status transition

**Validates:** [ideate#ac:approval-detection](../README.md#ac-approval-detection)

## Steps

GIVEN an Idea at `spec/ideas/my-idea.md` with `**Status:** Draft` body metadata
AND the skill has presented the lint-clean artifact for user review
WHEN the user responds with a vague positive signal (e.g., "looks good", "yeah", "nice", "ship it", "lgtm")
THEN the skill MUST NOT silently transition the status
AND the skill MUST ask one explicit confirmation question (e.g., "Treat that as approval?")
AND the skill MUST wait for the user's response

GIVEN the skill has asked the confirmation question
WHEN the user responds with an explicit approval phrase (`approve`, `approved`)
THEN the skill transitions the status from `Draft` to `Approved`
AND the skill emits `idea.approved`

GIVEN the skill has asked the confirmation question
WHEN the user declines or asks for changes
THEN the skill MUST NOT transition the status
AND the `**Status:**` body-metadata value remains `Draft`

---
*This document follows the https://specscore.md/scenario-specification*
