# Scenario: malformed subagent verdict is retried once, then recorded as error

**Validates:** [verify#ac:malformed-verdict-retried-once](../README.md#ac-malformed-verdict-retried-once-verifies-reqsubagent-verdict-contract)

## Steps

GIVEN an approved Feature `example` with AC `example#ac:a` and at least one matching commit
AND the verifier subagent is stubbed to return a malformed first response (verdict outside `{pass, fail, error}`, missing justification, or justification exceeding 400 characters)
WHEN the orchestrator parses the first response
THEN the orchestrator re-dispatches the subagent with a corrective prompt
AND the orchestrator dispatches the subagent for AC `a` exactly twice total

GIVEN the stubbed subagent also returns a malformed second response
WHEN the orchestrator parses the second response
THEN the orchestrator records AC `a` with `verdict: error`
AND the orchestrator does NOT dispatch the subagent a third time

---
*This document follows the https://specscore.md/scenario-specification*
