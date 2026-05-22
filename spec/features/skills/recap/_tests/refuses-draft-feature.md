# Scenario: recap refuses a Draft Feature

**Validates:** [recap#ac:refuses-draft-feature](../README.md#ac-refuses-draft-feature-verifies-reqrequires-approved-feature)

## Steps

GIVEN a Feature exists at `spec/features/example/README.md` with body metadata `**Status:** Draft`
AND the user runs `specstudio:recap example`
WHEN the skill reaches the pre-flight check
THEN the skill refuses to dispatch any subagent
AND the skill prints the Feature's current Status
AND the skill recommends `specstudio:specify` to re-approve
AND no report is written under `spec/features/example/_recap/`
AND the skill exits non-zero

---
*This document follows the https://specscore.md/scenario-specification*
