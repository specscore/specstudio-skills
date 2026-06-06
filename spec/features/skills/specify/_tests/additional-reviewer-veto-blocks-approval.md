---
format: https://specscore.md/scenario-specification
---

# Scenario: Any single registered reviewer's Issues Found blocks the User Review Gate (AND composition)

**Validates:** [specify#req:reviewer-composition](../README.md#req-reviewer-composition)

## Steps

GIVEN the project has registered one additional reviewer (e.g., a security-focused reviewer that checks auth/authz coverage)
AND the built-in reviewer returns `Approved`
WHEN the additional security reviewer returns `Issues Found` with a blocker-severity finding
THEN the skill MUST NOT release the User Review Gate
AND the skill MUST address the security blocker
AND the skill MUST re-dispatch the additional reviewer (and any other reviewer that previously returned `Issues Found`)

GIVEN both the built-in reviewer and an additional reviewer return `Approved`
WHEN the skill processes the composite verdict
THEN the User Review Gate becomes eligible to run

GIVEN three reviewers are registered (built-in, security, accessibility)
AND the built-in returns `Approved`
AND the security reviewer returns `Approved`
AND the accessibility reviewer returns `Issues Found` with a blocker
WHEN the skill processes the composite verdict
THEN the User Review Gate is blocked
AND the skill MUST NOT silently downgrade the accessibility blocker to advisory
AND the skill MUST NOT skip the accessibility reviewer in the next dispatch round

GIVEN one reviewer fails to dispatch (e.g., subagent prompt is missing or returns malformed output)
WHEN the skill processes the failure
THEN the skill MUST surface the dispatch failure to the user as a hard error
AND the skill MUST NOT treat the failed dispatch as `Approved` by default
AND the User Review Gate remains blocked until the failed reviewer dispatches successfully or the user explicitly removes it from the registry

---
*This document follows the https://specscore.md/scenario-specification*
