---
format: https://specscore.md/scenario-specification
---

# Scenario: exactly one verify.completed event is emitted per successful run with the documented payload

**Validates:** [verify#ac:verify-completed-event-emitted](../README.md#ac-verify-completed-event-emitted-verifies-reqverify-completed-event)

## Steps

GIVEN an approved Feature `example` with two ACs `a`, `b`
AND AC `a` returns `pass` from its verifier subagent
AND AC `b` has zero matching `Verifies:` trailers
AND git HEAD's SHA at run time is `<sha>`
WHEN the user runs `specstudio:verify example` and the run completes
THEN exactly one event with `event: verify.completed` is appended to `.specscore/events.jsonl`
AND the event payload includes `feature_slug: example`
AND the event payload includes `revision: <sha>`
AND the event payload includes `report_path: spec/features/example/_verify/<sha>.md`
AND the event payload's count fields are `passed_count: 1`, `failed_count: 0`, `unmapped_count: 1`, `errored_count: 0`
AND `passed_count + failed_count + unmapped_count + errored_count` equals 2 (the Feature's total AC count)
AND the event payload does NOT include per-AC verdict details

---
*This document follows the https://specscore.md/scenario-specification*
