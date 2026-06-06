---
format: https://specscore.md/scenario-specification
---

# Scenario: a `type: auto-approve` reviewer always approves and dispatches nothing

**Status:** pending
**Validates:** [reviewer-gates#ac:auto-approve-always-approves](../README.md#ac-auto-approve-always-approves)

## Steps

GIVEN a gate `reviewers:` list containing a `type: auto-approve` entry
WHEN the gate is evaluated (runner Step 2-auto-approve processes the entry)
THEN the `auto-approve` entry returns `Approved` with no findings
AND it dispatches nothing — no subagent, no command, no human prompt
AND it contributes no `Blocker` to the grade

---
*This document follows the https://specscore.md/scenario-specification*
