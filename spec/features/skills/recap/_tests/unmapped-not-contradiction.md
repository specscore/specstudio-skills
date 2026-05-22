# Scenario: unmapped ACs are informational, not contradictions

**Validates:** [recap#ac:unmapped-not-contradiction](../README.md#ac-unmapped-not-contradiction-verifies-requnmapped-detection-reqexit-code-semantics)

## Steps

GIVEN an Approved Feature `example` with three ACs `example#ac:a`, `example#ac:b`, `example#ac:c`
AND AC `c` has zero matching `Verifies:` trailers in the branch history
AND ACs `a` and `b` each return `no-drift` from their drift-narrator subagents
WHEN the user runs `specstudio:recap example`
THEN the recap report marks AC `c` with `verdict: unmapped`
AND the YAML summary's `drift` list includes AC `c` with `verdict: unmapped`
AND the skill exits zero

---
*This document follows the https://specscore.md/scenario-specification*
