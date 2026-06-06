---
format: https://specscore.md/scenario-specification
---

# Scenario: recap refuses when no verify report exists

**Validates:** [recap#ac:refuses-when-no-verify-report](../README.md#ac-refuses-when-no-verify-report-verifies-reqrequires-verify-report)

## Steps

GIVEN an Approved, committed Feature at `spec/features/example/README.md`
AND `spec/features/example/_verify/` either does not exist OR contains zero `<sha>.md` report files reachable at HEAD
AND the user runs `specstudio:recap example`
WHEN the skill reaches the pre-flight check
THEN the skill refuses to dispatch any subagent
AND the skill recommends running `specstudio:verify example` first
AND no report is written under `spec/features/example/_recap/`
AND the skill exits non-zero

---
*This document follows the https://specscore.md/scenario-specification*
