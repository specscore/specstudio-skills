---
format: https://specscore.md/scenario-specification
---

# Scenario: Post-approval Feature edits emit feature.updated, never re-emit feature.specified or feature.approved

**Validates:** [specify#ac:lifecycle-events](../README.md#ac-lifecycle-events)

## Steps

GIVEN a Feature at `spec/features/checkout-v2/` with `**Status:** Implementing` body metadata
AND `feature.approved` was previously emitted exactly once
WHEN the user edits the Feature (e.g., adds a new `#### REQ:` block under `## Behavior`)
AND `specscore spec lint` passes
THEN the skill emits `feature.updated`
AND the skill does NOT emit `feature.specified`
AND the skill does NOT re-emit `feature.approved`
AND the `**Status:**` body-metadata value remains `Implementing`

GIVEN the same approved Feature is edited multiple times in succession
WHEN each edit lints clean
THEN the skill emits `feature.updated` exactly once per successful lint pass
AND the count of `feature.approved` events remains exactly 1 across all edits
AND each `feature.updated` payload carries non-null `changed_sections`, `previous_revision`, and `change_summary`

GIVEN this is the very first `feature.specified` emission for a brand-new Feature
WHEN the skill emits the event
THEN the payload's `changed_sections`, `previous_revision`, and `change_summary` are all `null`

---
*This document follows the https://specscore.md/scenario-specification*
