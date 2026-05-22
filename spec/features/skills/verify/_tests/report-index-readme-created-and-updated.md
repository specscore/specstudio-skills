# Scenario: _verify/README.md index is created on first run and appended on subsequent runs

**Validates:** [verify#ac:report-index-readme-created-and-updated](../README.md#ac-report-index-readme-created-and-updated-verifies-reqreport-index-readme-reqreport-staged)

## Steps

GIVEN an approved Feature `example` whose `spec/features/example/_verify/` directory does NOT yet exist
AND git HEAD's abbreviated SHA is `<sha-1>`
WHEN the user runs `specstudio:verify example` and the run completes
THEN a file exists at `spec/features/example/_verify/README.md`
AND that README contains an H1 heading
AND that README contains a one-paragraph description
AND that README contains a `## Contents` table with columns `Report | Run revision | Verdict summary`
AND the `## Contents` table contains exactly one row whose `Report` cell links to `<sha-1>.md`
AND that README contains an `## Open Questions` section whose body is the literal `None at this time.`
AND that README ends with the line `*This document follows the https://specscore.md/index-specification*`
AND `git diff --cached --name-only` lists both `spec/features/example/_verify/README.md` and `spec/features/example/_verify/<sha-1>.md`
AND `specscore spec lint` exits zero

GIVEN the same Feature `example` whose `spec/features/example/_verify/README.md` now exists with N rows in its `## Contents` table from prior runs
AND git HEAD's abbreviated SHA is now `<sha-2>` where `<sha-2>` differs from every prior row's revision
WHEN the user runs `specstudio:verify example` again and the run completes
THEN the README's `## Contents` table contains N+1 rows
AND every prior row is preserved in its existing order
AND the new row links to `<sha-2>.md`
AND `git diff --cached --name-only` lists both `spec/features/example/_verify/README.md` and `spec/features/example/_verify/<sha-2>.md`
AND `specscore spec lint` exits zero

---
*This document follows the https://specscore.md/scenario-specification*
