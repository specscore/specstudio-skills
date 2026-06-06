---
type: rehearse-stub
status: pending
ac: scribe-summary-respects-flavor-and-length
feature: sidekick-consilium
format: https://specscore.md/scenario-specification
---

# Rehearse: scribe-summary-respects-flavor-and-length

## Scenario (from AC)

**Given** three pipeline runs with verdicts `should-implement`, `should-not-implement`, `needs-human-review`
**When** the scribe paragraphs on each completed task are inspected
**Then** each is ≤ 500 characters, contains no markdown headers/lists/code blocks, and uses language matching its verdict's flavor (e.g., the `needs-human-review` summary cites the dissenting argument).

## Verification approach

Run the pipeline against three fixture seeds designed to produce each of the three verdict enums (via tuned fixture votes); for each completed task, assert `scribe_summary` length ≤ 500, regex-assert no `^#`, no `^[-*]`, and no triple-backtick fences. For the `needs-human-review` case, assert the paragraph references the dissenting role or its argument.

---
*This document follows the https://specscore.md/scenario-specification*
