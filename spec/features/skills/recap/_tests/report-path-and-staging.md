---
format: https://specscore.md/scenario-specification
---

# Scenario: the recap report is written to the canonical path and staged without committing

**Validates:** [recap#ac:report-path-and-staging](../README.md#ac-report-path-and-staging-verifies-reqreport-path-reqreport-staged)

## Steps

GIVEN an Approved Feature `example`
AND a resolvable verify report at HEAD
AND a recap run that completes
WHEN the run finishes
THEN a Markdown report exists at `spec/features/example/_recap/<sha>.md` where `<sha>` is the abbreviated git SHA of HEAD
AND the file is staged via `git add`
AND the skill has NOT invoked `git commit`

---
*This document follows the https://specscore.md/scenario-specification*
