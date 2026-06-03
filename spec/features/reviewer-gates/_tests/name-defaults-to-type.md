# Scenario: an entry with no `name:` defaults its effective name to `type:`

**Status:** pending
**Validates:** [reviewer-gates#ac:name-defaults-to-type](../README.md#ac-name-defaults-to-type)

## Steps

GIVEN a `gates.feature.approved.reviewers` entry declaring `type: auto-approve` and omitting `name:`
WHEN a consumer loads the gate
THEN the entry's effective name is `auto-approve` (its `type:` value)
AND the gate loads without error

---
*This document follows the https://specscore.md/scenario-specification*
