# Scenario: an unmapped AC does not cause non-zero exit when all mapped ACs pass

**Validates:** [verify#ac:unmapped-not-fail](../README.md#ac-unmapped-not-fail-verifies-requnmapped-detection-reqexit-code-semantics)

## Steps

GIVEN an approved Feature `example` with three ACs `a`, `b`, `c`
AND AC `c` has zero matching `Verifies:` trailers in branch history
AND the verifier subagent returns `pass` for AC `a` and AC `b`
WHEN the user runs `specstudio:verify example`
THEN the report's YAML summary lists AC `c` with `verdict: unmapped`
AND the report's YAML summary lists AC `a` with `verdict: pass`
AND the report's YAML summary lists AC `b` with `verdict: pass`
AND the skill exits zero

---
*This document follows the https://specscore.md/scenario-specification*
