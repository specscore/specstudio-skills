---
format: https://specscore.md/scenario-specification
---

# Scenario: Clear intent triggers explicit hand-off to specify

**Validates:** [ideate#ac:skip-condition-respected](../README.md#ac-skip-condition-respected)

## Steps

GIVEN the user invokes `specstudio:ideate` with a single message that articulates the problem, the recommended direction, and an explicit Not-Doing boundary
WHEN the skill processes the invocation
THEN the skill recognizes the intent as clear
AND the skill does NOT run Phase 1 (divergent expansion)
AND the skill does NOT run Phase 2 (convergent stress-testing)
AND the skill explicitly tells the user it is handing off to `specstudio:specify` and why
AND no Idea artifact is written under `spec/ideas/`

GIVEN the user invokes `specstudio:ideate` with a vague concept that does NOT include all three of (problem, direction, Not-Doing)
WHEN the skill processes the invocation
THEN the skill proceeds with the three-phase dialogue
AND the skill does NOT hand off to `specstudio:specify`

---
*This document follows the https://specscore.md/scenario-specification*
