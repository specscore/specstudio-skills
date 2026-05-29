# Scenario: Approve threshold resolves per-stage first, then top-level, then default

**Status:** pending
**Validates:** [reviewer-gates#ac:threshold-resolution-order](../README.md#ac-threshold-resolution-order)

## Steps

GIVEN a `specscore.yaml` with top-level `grade.threshold: C` and `gates.specify.threshold: B`, plus a second stage that declares no per-stage `threshold`
WHEN the loader (Step 2.5) resolves the threshold for the `specify` stage, for the second stage, and for a repo with neither key present
THEN the `specify` stage resolves to `B` (per-stage overrides top-level)
AND the second stage resolves to `C` (top-level default applies)
AND the neither-key repo resolves to `B` (built-in default)

---
*This document follows the https://specscore.md/scenario-specification*
