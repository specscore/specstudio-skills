# Scenario: a threshold value outside {A,B,C,D,F} is refused at load time

**Status:** pending
**Validates:** [reviewer-gates#ac:invalid-threshold-refused](../README.md#ac-invalid-threshold-refused)

## Steps

GIVEN a `specscore.yaml` with `gates.specify.threshold: E` (a value outside the allowed set `{A, B, C, D, F}`)
WHEN `specstudio:specify` invokes the loader (Step 2.5)
THEN the loader refuses with an error citing `threshold-config` and pointing at the reviewer-gates Feature
AND no reviewer is dispatched
AND the skill exits non-zero
AND the same refusal occurs for other invalid values (e.g. `B+`, lowercase `b`, a number)

---
*This document follows the https://specscore.md/scenario-specification*
