# Scenario: Manual Promotes To edits are out of scope for ideate

**Validates:** [ideate#ac:promotion-boundary-held](../README.md#ac-promotion-boundary-held)

## Steps

GIVEN an approved Idea exists at `spec/ideas/my-idea.md`
AND the user asks `specstudio:ideate` to set `**Promotes To:** feature-x` in the body metadata
WHEN the skill processes the request
THEN the skill refuses to edit `**Promotes To:**`
AND the skill explains that `**Promotes To:**` is managed state, populated by lifecycle tooling in response to a Feature declaring `**Source Ideas:**`

GIVEN the user asks `specstudio:ideate` to scaffold or modify a SpecScore Feature
WHEN the skill processes the request
THEN the skill refuses to write or modify any file under `spec/features/`
AND the skill hands off to `specstudio:specify` with an explanation

GIVEN a Feature is later created at `spec/features/feature-x/` with `**Source Ideas:** my-idea`
WHEN Synchestra tooling reconciles the link
THEN the Idea's `**Promotes To:**` body-metadata line is populated with `feature-x`
AND the Idea's `**Status:**` transitions from `Approved` to `Specified`
AND `ideate` is not involved in either change

---
*This document follows the https://specscore.md/scenario-specification*
