---
format: https://specscore.md/scenario-specification
---

# Scenario: Revise-in-place is the default; supersede is reserved for scope-invalidating changes

**Validates:** [specify#req:revise-vs-supersede](../README.md#req-revise-vs-supersede)

## Steps

GIVEN an existing Feature at `spec/features/checkout-v2/` with `**Status:** Implementing` body metadata
AND the user wants to add a new requirement that aligns with the existing scope
WHEN the user invokes `specstudio:specify` against the same slug
THEN the skill defaults to revising the Feature in place
AND the skill adds a new `#### REQ: <slug>` block under an existing or new `### <Topic>` heading inside `## Behavior`
AND the skill updates the Feature `README.md` as needed
AND no new directory is created
AND `**Supersedes:**` is NOT set

GIVEN an existing Feature at `spec/features/checkout-v2/` with multiple Stable ACs
AND the user wants to make a change that invalidates one or more of those ACs
WHEN the user invokes `specstudio:specify` against the same slug
THEN the skill MUST surface the invalidation to the user
AND the skill MUST present a choice: revise in place (and downgrade affected ACs) OR create a successor with `**Supersedes:** checkout-v2` body metadata
AND the skill MUST NOT silently rewrite the existing Feature

GIVEN the user chooses to create a successor
WHEN the skill processes the request
THEN the skill creates `spec/features/checkout-v3/` (user-provided successor slug) with `**Supersedes:** checkout-v2` body metadata
AND the original Feature's `**Status:**` remains as it was (the user manually deprecates if intended)
AND the skill MUST NOT delete or modify the original Feature's content

---
*This document follows the https://specscore.md/scenario-specification*
