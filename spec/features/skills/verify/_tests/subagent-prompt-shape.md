# Scenario: dispatched subagent prompt contains AC text, commits, fetch-on-demand instruction, and verdict contract

**Validates:** [verify#ac:subagent-prompt-shape](../README.md#ac-subagent-prompt-shape-verifies-reqsubagent-prompt-reqsubagent-verdict-contract)

## Steps

GIVEN an approved Feature `example` with AC `example#ac:a` whose body contains a complete `Given / When / Then`
AND the branch contains two commits `<sha-1>` and `<sha-2>` whose messages include `Verifies: example#ac:a`
WHEN the orchestrator dispatches the verifier subagent for AC `a`
THEN the dispatched prompt contains the full `Given / When / Then` text of AC `a` with the AC ID `example#ac:a`
AND the dispatched prompt contains `<sha-1>` and `<sha-2>` paired with their commit messages
AND the dispatched prompt does NOT contain the diff of `<sha-1>` or the diff of `<sha-2>`
AND the dispatched prompt contains an explicit instruction to fetch diffs and read source files on demand via the subagent's own Bash tool
AND the dispatched prompt contains the verbatim verdict contract listing allowed values `{pass, fail, error}`, the required justification length, and the evidence-reference requirement

---
*This document follows the https://specscore.md/scenario-specification*
