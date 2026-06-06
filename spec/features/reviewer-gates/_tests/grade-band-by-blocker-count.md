---
format: https://specscore.md/scenario-specification
---

# Scenario: the grade band is fixed deterministically by aggregated Blocker count

**Status:** pending
**Validates:** [reviewer-gates#ac:grade-band-by-blocker-count](../README.md#ac-grade-band-by-blocker-count)

## Steps

GIVEN four gate runs whose aggregated findings contain exactly 0, 1, 3, and 4 `Blocker` findings respectively (fixtures: a mocked reviewer returning a configurable findings list; threshold left at the default `B`)
WHEN the runner computes the grade for each run per `grade-band-mapping` Step 2.8
THEN the 0-Blocker run lands in the pass band (`A` or `B`), the 1-Blocker run is `C`, the 3-Blocker run is `D`, and the 4-Blocker run is `F`
AND repeating each run with the same findings yields the same band every time (the band depends only on `Blocker` count, never on reviewer judgment)

---
*This document follows the https://specscore.md/scenario-specification*
