# Scenario: the released grade is recorded in the event payload and on the artifact

**Status:** pending
**Validates:** [reviewer-gates#ac:grade-recorded-on-release](../README.md#ac-grade-recorded-on-release)

## Steps

GIVEN a Feature whose `specstudio:specify` gate releases with grade `B` (zero `Blocker`s, no within-band `A`)
WHEN the gate releases
THEN `specstudio:specify` emits a `feature.approved` event whose payload includes `grade: B`
AND the Feature's `README.md` contains a `**Grade:** B` body-metadata line immediately after `**Supersedes:**` (added if absent, updated in place if already present)
AND `specscore spec lint` passes with that line present

---
*This document follows the https://specscore.md/scenario-specification*
