# Scenario: a `type: noop` reviewer always approves and dispatches nothing

**Status:** pending
**Validates:** [reviewer-gates#ac:noop-always-approves](../README.md#ac-noop-always-approves)

## Steps

GIVEN a gate `reviewers:` list containing a `type: noop` entry
WHEN the gate is evaluated (runner Step 2-noop processes the entry)
THEN the `noop` entry returns `Approved` with no findings
AND it dispatches nothing — no subagent, no command, no human prompt
AND it contributes no `Blocker` to the grade

---
*This document follows the https://specscore.md/scenario-specification*
