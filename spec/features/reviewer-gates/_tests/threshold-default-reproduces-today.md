# Scenario: at the default threshold B the grade gate reproduces today's binary behavior

**Status:** pending
**Validates:** [reviewer-gates#ac:threshold-default-reproduces-today](../README.md#ac-threshold-default-reproduces-today)

## Steps

GIVEN a repo with no `gates.<stage>.threshold` and no top-level `grade.threshold`, using the existing findings-only reviewer prompt
WHEN a gate runs on an artifact whose aggregated findings contain zero `Blocker`s, and separately on an artifact whose findings contain one or more `Blocker`s
THEN the loader resolves the threshold to the built-in default `B`
AND the zero-`Blocker` artifact releases with verdict `Approved` (grade `A` or `B` ≥ `B`)
AND the `Blocker`-bearing artifact does not release (grade ≤ `C` < `B`, verdict `Issues Found`) — identical to the pre-grade AND-composition outcome

---
*This document follows the https://specscore.md/scenario-specification*
