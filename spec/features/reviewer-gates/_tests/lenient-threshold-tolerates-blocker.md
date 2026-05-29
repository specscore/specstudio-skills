# Scenario: a lenient threshold C releases an artifact with grade C

**Status:** pending
**Validates:** [reviewer-gates#ac:lenient-threshold-tolerates-blocker](../README.md#ac-lenient-threshold-tolerates-blocker)

## Steps

GIVEN `gates.specify.threshold: C` and an artifact whose aggregated findings contain exactly one `Blocker` (grade `C`)
WHEN the gate runs (runner Step 2.7 does NOT early-halt because the threshold is lenient, and Step 2.8 derives the verdict)
THEN the gate releases with verdict `Approved` because `C ≥ C`
AND the same artifact under the default threshold `B` does not release (`C < B`, verdict `Issues Found`)

---
*This document follows the https://specscore.md/scenario-specification*
