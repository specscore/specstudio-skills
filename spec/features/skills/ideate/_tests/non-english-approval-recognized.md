---
format: https://specscore.md/scenario-specification
---

# Scenario: Direct semantic equivalents in non-English languages count as explicit approval

**Validates:** [ideate#req:approval-explicit-phrase](../README.md#req-approval-explicit-phrase)

## Steps

GIVEN an Idea at `spec/ideas/my-idea.md` with `**Status:** Draft` body metadata
AND the user is communicating in Spanish
AND the skill has presented the lint-clean artifact for user review
WHEN the user responds with the standalone phrase `aprobado`
THEN the skill recognizes the response as an explicit approval (direct semantic equivalent of `approved`)
AND the skill does NOT ask for additional confirmation
AND the skill transitions the `**Status:**` body-metadata value from `Draft` to `Approved`
AND the skill emits `idea.approved`

GIVEN the user is communicating in Japanese
WHEN the user responds with `承認` or `承認する`
THEN the skill treats the response as explicit approval and proceeds without confirmation

GIVEN the user is communicating in Russian
WHEN the user responds with `одобрено` or `принято`
THEN the skill treats the response as explicit approval and proceeds without confirmation

GIVEN the user is communicating in Spanish
WHEN the user responds with `sí` (a generic affirmative, not the explicit approval verb)
THEN the skill MUST NOT treat the response as explicit approval
AND the skill MUST ask one explicit confirmation question per `approval-vague-confirmation`

GIVEN the user is communicating in Japanese
WHEN the user responds with `はい` (a generic affirmative)
THEN the skill MUST NOT treat the response as explicit approval
AND the skill MUST ask one explicit confirmation question

---
*This document follows the https://specscore.md/scenario-specification*
