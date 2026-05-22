# Scenario: `_recap/README.md` index is created on first run and appended on subsequent runs

**Validates:** [recap#ac:report-index-readme-created-and-updated](../README.md#ac-report-index-readme-created-and-updated-verifies-reqreport-index-readme-reqreport-staged)

## Steps

GIVEN an Approved Feature `example` whose `_recap/` directory does not yet exist
WHEN the user runs `specstudio:recap example` for the first time
THEN the skill creates `spec/features/example/_recap/README.md`
AND the README contains an H1
AND the README contains a one-paragraph description
AND the README contains a `## Contents` table with exactly one row referencing the just-written `<sha>.md`
AND the README contains an `## Open Questions` section set to `None at this time.`
AND the README ends with the `*This document follows the https://specscore.md/index-specification*` footer
AND the README is in the same staged set as the per-run report
AND `specscore spec lint` exits zero after the run

GIVEN an Approved Feature `example` whose `_recap/README.md` already exists from a prior run with N rows in its `## Contents` table
WHEN the user runs `specstudio:recap example` again
THEN the skill appends one row to the table for the current `<sha>.md`
AND the existing N rows are preserved in their order
AND the resulting table has N+1 rows
AND the README is staged alongside the new per-run report
AND `specscore spec lint` exits zero

---
*This document follows the https://specscore.md/scenario-specification*
