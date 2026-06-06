---
format: https://specscore.md/scenario-specification
---

# Scenario: subagents are dispatched serially, never concurrently

**Validates:** [verify#ac:subagent-serial-dispatch](../README.md#ac-subagent-serial-dispatch-verifies-reqsubagent-dispatch-serial)

## Steps

GIVEN an approved Feature `example` with four ACs `a`, `b`, `c`, `d`
AND every AC has at least one matching `Verifies:` trailer
AND the orchestrator records the start and end timestamp of each verifier-subagent call
WHEN the user runs `specstudio:verify example`
THEN the orchestrator dispatches subagents in AC order `a`, `b`, `c`, `d`
AND for any two consecutive ACs `X` and `Y`, the end timestamp of `X` precedes the start timestamp of `Y`
AND at no observed moment during the run are two verifier subagents in flight simultaneously

---
*This document follows the https://specscore.md/scenario-specification*
