---
format: https://specscore.md/scenario-specification
---

# Scenario: Reviewer subagent must approve before the User Review Gate runs

**Validates:** [specify#ac:reviewer-then-user](../README.md#ac-reviewer-then-user)

## Steps

GIVEN the skill has written a lint-clean Feature
AND the reviewer subagent has NOT yet been dispatched
WHEN the skill is about to present the Feature to the user
THEN the skill MUST first dispatch the reviewer subagent per `references/reviewer-prompt.md`
AND the skill MUST wait for the subagent's verdict before any user-facing presentation

GIVEN the reviewer subagent returns `Issues Found` with one or more blocker-severity findings
WHEN the skill processes the verdict
THEN the skill MUST address every blocker finding
AND the skill MUST re-dispatch the reviewer subagent
AND the skill MUST NOT present the Feature to the user until the subagent returns `Approved`

GIVEN the reviewer subagent returns `Issues Found` with only advisory recommendations (no blockers)
WHEN the skill processes the verdict
THEN the skill MAY ignore the advisory recommendations
AND the skill MAY proceed to the User Review Gate
AND the skill SHOULD surface the advisory recommendations to the user as context

GIVEN the reviewer subagent returns `Approved`
AND lint still passes after any fix-and-re-dispatch loop
WHEN the skill presents the Feature to the user
THEN the user's approval is independent of and downstream from the reviewer's
AND both verdicts are required for the hard gate to release

---
*This document follows the https://specscore.md/scenario-specification*
