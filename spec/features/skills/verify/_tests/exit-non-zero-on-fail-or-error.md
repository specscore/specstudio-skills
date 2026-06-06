---
format: https://specscore.md/scenario-specification
---

# Scenario: skill exits non-zero when any AC has verdict fail or error

**Validates:** [verify#ac:exit-non-zero-on-fail-or-error](../README.md#ac-exit-non-zero-on-fail-or-error-verifies-reqexit-code-semantics)

## Steps

GIVEN an approved Feature `example` with three ACs `a`, `b`, `c`
AND every AC has at least one matching `Verifies:` trailer
AND the verifier subagent is stubbed to return `pass` for AC `a`, `fail` for AC `b`, `pass` for AC `c`
WHEN the user runs `specstudio:verify example`
THEN the report is written at the canonical path
AND the report's YAML summary marks AC `b` with `verdict: fail`
AND the skill exits non-zero

GIVEN an approved Feature `example2` with two ACs `x`, `y` each having at least one matching `Verifies:` trailer
AND the verifier subagent is stubbed to return malformed responses for AC `y` on both the initial dispatch and the corrective re-dispatch (per `REQ:subagent-verdict-contract`)
AND the verifier subagent returns `pass` for AC `x`
WHEN the user runs `specstudio:verify example2`
THEN the report is written at the canonical path
AND the report's YAML summary marks AC `y` with `verdict: error`
AND the skill exits non-zero

---
*This document follows the https://specscore.md/scenario-specification*
