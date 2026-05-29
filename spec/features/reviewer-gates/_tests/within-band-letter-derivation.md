# Scenario: the within-band letter is lowest-wins across reviewers, defaulting to B

**Status:** pending
**Validates:** [reviewer-gates#ac:within-band-letter-derivation](../README.md#ac-within-band-letter-derivation)

## Steps

GIVEN three zero-`Blocker` pass-band gate runs — (a) a single reviewer supplies within-band letter `A`; (b) one reviewer supplies `A` and another supplies `B`; (c) a findings-only reviewer supplies no within-band letter
WHEN the runner computes the grade (Step 2.8.3 lowest-wins, default `B` when no letter is supplied) and derives the verdict (Step 2.8.4)
THEN the grade is `A` in (a), `B` in (b), and `B` in (c)
AND a configured threshold of `A` releases only in case (a)
AND the default threshold `B` releases in all three cases

---
*This document follows the https://specscore.md/scenario-specification*
