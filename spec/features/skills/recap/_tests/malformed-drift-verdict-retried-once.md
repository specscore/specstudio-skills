---
format: https://specscore.md/scenario-specification
---

# Scenario: a malformed drift verdict is retried exactly once

**Validates:** [recap#ac:malformed-drift-verdict-retried-once](../README.md#ac-malformed-drift-verdict-retried-once-verifies-reqsubagent-drift-contract)

## Steps

GIVEN a drift-narrator subagent that returns a malformed response on its first call (e.g. a verdict outside `{no-drift, spec-tighter-than-code, code-tighter-than-spec, contradiction}`, OR a missing narrative, OR a narrative exceeding 500 characters)
WHEN the orchestrator parses the response
THEN the orchestrator re-dispatches the same subagent with a corrective prompt exactly once
AND if the second response is also malformed, the orchestrator records the AC's drift verdict as `error`
AND the orchestrator does NOT call the subagent a third time

---
*This document follows the https://specscore.md/scenario-specification*
