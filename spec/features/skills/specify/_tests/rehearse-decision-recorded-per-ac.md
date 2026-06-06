---
format: https://specscore.md/scenario-specification
---

# Scenario: Every AC has a recorded Rehearse decision (stub or skip-reason)

**Validates:** [specify#ac:rehearse-coverage](../README.md#ac-rehearse-coverage)

## Steps

GIVEN a Feature with three ACs, two of which are testable (e.g., HTTP-surface) and one of which is subjective (e.g., "Then the dashboard feels uncluttered")
WHEN the skill applies the per-AC Rehearse heuristic
THEN the two testable ACs each have a scaffolded stub at `spec/features/<slug>/_tests/<scenario-slug>.md` with `**Status:** pending` body metadata
AND the subjective AC has a documented skip-reason in the Feature `README.md` under a `## Rehearse Integration` subsection
AND no AC is silently ignored

GIVEN a Rehearse stub exists for an AC and is staged in git per the auto-stage rule
WHEN the user inspects the stub
THEN the stub uses `Given / When / Then` form
AND the stub's `**Validates:**` line references the AC ID

GIVEN the user wants to override the heuristic and skip a stub for a testable AC
WHEN the user instructs the skill to skip
THEN the skill records the user-provided reason in `## Rehearse Integration`
AND the skill MUST NOT scaffold the stub

GIVEN the user wants to override and force a stub for a subjective AC
WHEN the user instructs the skill to scaffold
THEN the skill scaffolds the stub
AND records the user override and reasoning in `## Rehearse Integration`

---
*This document follows the https://specscore.md/scenario-specification*
