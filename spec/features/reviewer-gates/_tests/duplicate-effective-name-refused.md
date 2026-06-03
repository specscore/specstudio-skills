# Scenario: two same-type entries that both omit `name:` collide and are refused

**Status:** pending
**Validates:** [reviewer-gates#ac:duplicate-effective-name-refused](../README.md#ac-duplicate-effective-name-refused)

## Steps

GIVEN a gate `reviewers:` list with two `type: ai` entries that both omit `name:`
AND both therefore default to the effective name `ai`
WHEN a consumer loads the gate
THEN the consumer refuses to run with an error citing `reviewer-entry-required-fields` (duplicate effective name)
AND it dispatches no reviewer and exits non-zero

---
*This document follows the https://specscore.md/scenario-specification*
