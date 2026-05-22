# Scenario: the recap report begins with a grep-friendly YAML summary block

**Validates:** [recap#ac:report-yaml-block-grep-target](../README.md#ac-report-yaml-block-grep-target-verifies-reqreport-yaml-summary-reqreport-body)

## Steps

GIVEN a completed recap run for Feature `example`
WHEN a downstream consumer reads the report file
THEN the report's first content is a fenced YAML block delimited by ` ```yaml ` and ` ``` `
AND the YAML block lists every AC with `ac`, `verdict`, and `narrative` fields in the Feature's AC order
AND the YAML block includes top-level `feature:`, `revision:`, and `verify_revision:` fields
AND below the YAML block, the report contains one `## AC: <ac-slug>` section per AC
AND each `## AC:` section carries the drift verdict, the full narrative, the verify verdict and justification for the same AC (verbatim from the verify report), the commit list, and evidence references

---
*This document follows the https://specscore.md/scenario-specification*
