---
type: rehearse-stub
status: pending
ac: researcher-briefing-contains-no-judgment
feature: sidekick-consilium
---

# Rehearse: researcher-briefing-contains-no-judgment

## Scenario (from AC)

**Given** a fixture seed and a completed pipeline run
**When** the briefing pack stored on the task is inspected
**Then** the briefing contains only structured facts (file paths, line ranges, related-artifact slugs, git activity); a grep for judgment-laden tokens (e.g., "looks important", "concerning", "should consider", "recommend") finds none. (Test asserts the negative.)

## Verification approach

Run the pipeline against a fixture seed; read `briefing_pack` from the completed task payload; assert a case-insensitive regex search for the curated set of judgment-laden tokens returns zero hits. Also assert the briefing contains the expected structured-fact shape (file paths, line ranges, related-artifact slugs).

---
*This document follows the https://specscore.md/scenario-specification*
