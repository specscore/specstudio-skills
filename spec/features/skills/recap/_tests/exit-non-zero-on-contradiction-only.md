---
format: https://specscore.md/scenario-specification
---

# Scenario: recap exits non-zero only on `contradiction` or `error` drift verdicts

**Validates:** [recap#ac:exit-non-zero-on-contradiction-only](../README.md#ac-exit-non-zero-on-contradiction-only-verifies-reqexit-code-semantics)

## Steps

GIVEN an Approved Feature `example` where the drift-narrator subagent returns `contradiction` for at least one AC
WHEN the run completes and the report is written
THEN the skill exits non-zero
AND the YAML summary marks the affected AC(s) with `verdict: contradiction`

GIVEN an Approved Feature `example` where every AC returns one of `{no-drift, spec-tighter-than-code, code-tighter-than-spec}`
AND at least one AC is `unmapped`
WHEN the run completes and the report is written
THEN the skill exits zero
AND the exit code is independent of how many ACs carry `spec-tighter-than-code`, `code-tighter-than-spec`, or `unmapped`

GIVEN an Approved Feature `example` where at least one AC ends with drift verdict `error` (e.g. because the subagent returned malformed responses on both attempts)
WHEN the run completes and the report is written
THEN the skill exits non-zero
AND the YAML summary marks the affected AC(s) with `verdict: error`

---
*This document follows the https://specscore.md/scenario-specification*
