---
format: https://specscore.md/scenario-specification
---

# Scenario: Feature with zero matching trailers produces a report with all ACs unmapped and exits zero

**Validates:** [verify#ac:no-commits-still-reports](../README.md#ac-no-commits-still-reports-verifies-reqno-commits-edge-case)

## Steps

GIVEN an approved Feature `example` with three ACs `a`, `b`, `c`
AND no commit in the branch history contains a `Verifies:` trailer referencing any of `example#ac:a`, `example#ac:b`, `example#ac:c`
WHEN the user runs `specstudio:verify example`
THEN a Markdown report exists at `spec/features/example/_verify/<sha>.md`
AND the report's YAML summary lists every AC with `verdict: unmapped`
AND `git diff --cached --name-only` lists the report path
AND exactly one `verify.completed` event is appended to `.specscore/events.jsonl`
AND the event payload's count fields are `passed_count: 0`, `failed_count: 0`, `unmapped_count: 3`, `errored_count: 0`
AND the skill exits zero

---
*This document follows the https://specscore.md/scenario-specification*
