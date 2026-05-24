# Scenario: Source Ideas references must resolve to existing Approved/Specified Ideas

**Validates:** [specify#ac:source-idea-linkage](../README.md#ac-source-idea-linkage)

## Steps

GIVEN an approved Idea exists at `spec/ideas/payment-fraud.md` with `**Status:** Approved` body metadata
AND the user invokes `specstudio:specify` against that Idea
WHEN the skill writes the Feature
THEN the Feature `README.md` contains `**Source Ideas:** payment-fraud` immediately after the Status line
AND `specscore spec lint` accepts the reference

GIVEN the user attempts to declare `**Source Ideas:** does-not-exist` in a Feature
WHEN `specscore spec lint` runs against the Feature directory
THEN lint reports an error: the referenced Idea slug does not resolve to an existing file
AND the hard-gate blocks downstream `writing-plans`

GIVEN an Idea exists at `spec/ideas/draft-only.md` with `**Status:** Draft` body metadata (not yet Approved)
AND the user attempts to declare `**Source Ideas:** draft-only` in a Feature
WHEN `specscore spec lint` runs against the Feature directory
THEN lint reports an error: source Ideas must have `Status ∈ {Approved, Specified}`
AND the skill MUST NOT proceed

GIVEN the user asks `specstudio:specify` to manually edit the source Idea's `**Promotes To:**` or `**Status:**` body-metadata line directly
WHEN the skill processes the request
THEN the skill refuses
AND the skill explains that those lines are managed by lifecycle tooling in response to the Feature's `**Source Ideas:**` declaration

---
*This document follows the https://specscore.md/scenario-specification*
