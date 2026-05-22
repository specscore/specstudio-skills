# Scenario: drift-narrator subagents are dispatched one at a time, in AC order

**Validates:** [recap#ac:subagent-serial-dispatch](../README.md#ac-subagent-serial-dispatch-verifies-reqsubagent-dispatch-serial)

## Steps

GIVEN an Approved Feature `example` with four ACs each having at least one matching `Verifies:` trailer commit
AND a resolvable verify report at HEAD
WHEN the user runs `specstudio:recap example`
THEN the skill dispatches exactly one drift-narrator subagent at a time
AND the subagents are dispatched in the Feature's AC order
AND at no point during the run is more than one narrator subagent concurrently in flight

---
*This document follows the https://specscore.md/scenario-specification*
