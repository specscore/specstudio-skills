# Scenario: the Blocker union across panel reviewers drives the grade (worst-wins)

**Status:** pending
**Validates:** [reviewer-gates#ac:worst-wins-union-across-reviewers](../README.md#ac-worst-wins-union-across-reviewers)

## Steps

GIVEN a `gates.specify.reviewers` panel of two `type: ai` reviewers where reviewer A reports zero `Blocker`s and reviewer B reports one `Blocker`, gated at the default threshold `B`
WHEN the gate aggregates the verdicts (runner Step 2.8.1 forms the `Blocker` union across reviewers dispatched in the pass)
THEN the aggregated `Blocker` union is 1
AND the computed grade is `C`
AND the gate does not release at threshold `B` (verdict `Issues Found`)

---
*This document follows the https://specscore.md/scenario-specification*
