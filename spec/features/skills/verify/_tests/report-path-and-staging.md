---
format: https://specscore.md/scenario-specification
---

# Scenario: report is written to the canonical path and staged but not committed

**Validates:** [verify#ac:report-path-and-staging](../README.md#ac-report-path-and-staging-verifies-reqreport-path-reqreport-staged)

## Steps

GIVEN an approved Feature `example` and a clean working tree
AND git HEAD's abbreviated SHA is `<sha>`
WHEN the user runs `specstudio:verify example` and the run completes
THEN a Markdown file exists at `spec/features/example/_verify/<sha>.md`
AND `git diff --cached --name-only` lists `spec/features/example/_verify/<sha>.md`
AND the skill did NOT invoke `git commit`
AND the working tree contains no other files modified outside `spec/features/example/_verify/`

---
*This document follows the https://specscore.md/scenario-specification*
