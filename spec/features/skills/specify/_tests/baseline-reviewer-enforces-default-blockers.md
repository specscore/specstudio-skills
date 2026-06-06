---
format: https://specscore.md/scenario-specification
---

# Scenario: Built-in reviewer enforces the six baseline blocker categories

**Validates:** [specify#req:reviewer-baseline-blockers](../README.md#req-reviewer-baseline-blockers)

## Steps

GIVEN a Feature where one requirement's AC has the `Then` clause "Then the dashboard feels uncluttered"
WHEN the built-in reviewer subagent dispatches
THEN the subagent returns `Issues Found`
AND the finding is categorized as `unobservable Then clause` (baseline blocker #2)
AND the skill MUST address the finding before re-dispatching

GIVEN a Feature whose Behavior section describes a "single-tenant" architecture but whose requirements include a `tenant_id` parameter on every endpoint
WHEN the built-in reviewer dispatches
THEN the subagent returns `Issues Found`
AND the finding is categorized as `architecture↔requirements contradiction` (baseline blocker #4)

GIVEN a Feature that bundles "checkout flow + payment processing + invoice generation + analytics" into one Feature
WHEN the built-in reviewer dispatches
THEN the subagent returns `Issues Found`
AND the finding is categorized as `scope spans multiple subsystems` (baseline blocker #1)
AND the skill MUST surface the decomposition recommendation to the user

GIVEN a Feature with `Source Ideas: payment-fraud` declared, but the Feature's Recommended Direction silently abandons the Idea's recommended approach with no explanation
WHEN the built-in reviewer dispatches
THEN the subagent returns `Issues Found`
AND the finding is categorized as `missing source-Idea reasoning` (baseline blocker #6)

GIVEN a Feature where every requirement is concrete, every AC is observable, scope is single-system, architecture is consistent, and source-Idea reasoning is documented
WHEN the built-in reviewer dispatches
THEN the subagent returns `Approved`
AND the User Review Gate becomes eligible to run

GIVEN the reviewer surfaces a finding outside the six baseline categories (e.g., "the Title could be more evocative")
WHEN the skill processes the verdict
THEN the finding is categorized as `Advisory` severity
AND the skill MAY proceed to the User Review Gate without addressing the finding
AND the skill SHOULD surface the advisory recommendation to the user as context

---
*This document follows the https://specscore.md/scenario-specification*
