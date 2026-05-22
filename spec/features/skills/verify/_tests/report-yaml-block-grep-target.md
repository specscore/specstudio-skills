# Scenario: report opens with a fenced YAML block listing per-AC verdicts and followed by per-AC body sections

**Validates:** [verify#ac:report-yaml-block-grep-target](../README.md#ac-report-yaml-block-grep-target-verifies-reqreport-yaml-summary-reqreport-body)

## Steps

GIVEN an approved Feature `example` with three ACs `a`, `b`, `c` in that order in the Feature body
AND a completed verify run produced a report at `spec/features/example/_verify/<sha>.md`
WHEN a downstream consumer reads the report file
THEN the first content block in the file is a fenced YAML block delimited by ` ```yaml ` and ` ``` `
AND the YAML block's `verdicts` list orders entries `a`, `b`, `c` matching the Feature's AC order
AND each `verdicts` entry contains `ac`, `verdict`, and `justification` fields
AND below the YAML block the report contains one `## AC: a`, `## AC: b`, and `## AC: c` section

---
*This document follows the https://specscore.md/scenario-specification*
