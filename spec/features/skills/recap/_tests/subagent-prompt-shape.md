# Scenario: the drift-narrator subagent prompt contains the AC, verify verdict, commits, contract, and on-demand fetch instruction

**Validates:** [recap#ac:subagent-prompt-shape](../README.md#ac-subagent-prompt-shape-verifies-reqsubagent-prompt-reqsubagent-drift-contract)

## Steps

GIVEN AC `example#ac:a` with two matching `Verifies:` trailer commits
AND a resolved verify report listing AC `a` with `verdict: pass` and a one-line justification snippet
AND a `Given / When / Then` body for AC `a`
WHEN the orchestrator dispatches the drift-narrator subagent for AC `a`
THEN the dispatched prompt contains the AC's full Given / When / Then text
AND the prompt contains the AC's verify verdict carried over verbatim from the resolved verify report
AND the prompt contains the AC's verify justification snippet carried over verbatim from the resolved verify report
AND the prompt contains both commit SHAs paired with their commit messages
AND the prompt contains the verbatim drift verdict contract (allowed values, required narrative length, evidence-reference requirement)
AND the prompt contains an explicit instruction to fetch diffs and read source files on demand via the subagent's own Bash tool
AND the prompt does NOT contain pre-fetched commit diffs
AND the allowed-verdict set named to the subagent does NOT include `unmapped` or `error`

---
*This document follows the https://specscore.md/scenario-specification*
