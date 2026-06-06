---
format: https://specscore.md/scenario-specification
---

# Scenario: a `recap.completed` event is emitted exactly once per successful run

**Validates:** [recap#ac:recap-completed-event-emitted](../README.md#ac-recap-completed-event-emitted-verifies-reqrecap-completed-event)

## Steps

GIVEN a recap run that completes (report written, regardless of drift outcomes) on Feature `example` with N total ACs
WHEN the orchestrator finishes the report write
THEN exactly one `recap.completed` event is appended to `.specscore/events.jsonl` (or emitted via `specscore event emit` when the CLI is available)
AND the payload contains the field `feature_slug`
AND the payload contains the field `revision`
AND the payload contains the field `report_path`
AND the payload contains the field `verify_report_path`
AND the payload contains the field `no_drift_count`
AND the payload contains the field `spec_tighter_count`
AND the payload contains the field `code_tighter_count`
AND the payload contains the field `contradiction_count`
AND the payload contains the field `unmapped_count`
AND the payload contains the field `errored_count`
AND each of the six count fields is a non-negative integer
AND `no_drift_count + spec_tighter_count + code_tighter_count + contradiction_count + unmapped_count + errored_count` equals N
AND the payload does NOT include per-AC drift details

---
*This document follows the https://specscore.md/scenario-specification*
