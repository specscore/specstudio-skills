---
format: https://specscore.md/scenario-specification
---

# Scenario: the BA lens raises a Blocker when requirements do not address the stated Problem

**Status:** pending
**Validates:** [reviewer-gates#ac:ba-lens-problem-traceability-blocker](../README.md#ac-ba-lens-problem-traceability-blocker)

## Steps

GIVEN a Feature whose requirements are internally consistent but do not demonstrably address its stated `## Problem`, reviewed by the default multi-role reviewer (a mocked reviewer configured to apply the BA problem-traceability lens)
WHEN the BA lens evaluates the artifact
THEN the reviewer emits a `Blocker` finding under the BA lens for problem→requirements traceability (taxonomy category 7)
AND the aggregated `Blocker` count is at least 1
AND the gate does not release at the default threshold `B` (verdict `Issues Found`)

---
*This document follows the https://specscore.md/scenario-specification*
