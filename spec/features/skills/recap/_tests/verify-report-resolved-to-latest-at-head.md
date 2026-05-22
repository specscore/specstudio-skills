# Scenario: recap resolves the verify report to the one at HEAD (or most recent in branch history)

**Validates:** [recap#ac:verify-report-resolved-to-latest-at-head](../README.md#ac-verify-report-resolved-to-latest-at-head-verifies-reqverify-report-resolution)

## Steps

GIVEN an Approved Feature `example` whose `_verify/` directory contains three reports `<sha-A>.md`, `<sha-B>.md`, and `<sha-C>.md`
AND `<sha-C>` matches `git rev-parse --short HEAD`
WHEN the user runs `specstudio:recap example`
THEN the skill resolves the verify report to `<sha-C>.md`
AND the skill parses the report's top-of-file YAML summary block
AND the skill surfaces per-AC verify verdicts and justifications to the subagent prompts
AND the resulting recap report's YAML summary records `verify_revision: <sha-C>`

GIVEN an Approved Feature `example` whose `_verify/` directory contains reports for older SHAs but no report at HEAD exactly
WHEN the user runs `specstudio:recap example`
THEN the skill resolves to the verify report whose embedded `revision:` field is most recent in the branch's git history
AND the skill parses that report's YAML summary
AND the resulting recap report's YAML summary records that report's `revision` as `verify_revision:`

---
*This document follows the https://specscore.md/scenario-specification*
