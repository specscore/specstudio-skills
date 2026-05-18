---
type: rehearse-stub
status: pending
ac: roster-with-malformed-custom-role-rejected
feature: sidekick-consilium
---

# Rehearse: roster-with-malformed-custom-role-rejected

## Scenario (from AC)

**Given** a `specscore.yaml` listing a custom role whose markdown file is missing the `**Group:**` metadata line
**When** the skill is invoked
**Then** the arbiter's roster-load validation fails with a clear error naming the missing field and the file path; no task is claimed; the skill exits non-zero.

## Verification approach

Create a fixture custom-role markdown file missing the `**Group:**` line and reference it from `specscore.yaml`'s `consilium.roster.custom`; run `/consilium` against a fixture project with a queued task. Assert non-zero exit code, the stderr message names both the missing field (`Group`) and the file path, and the queued task remains unclaimed.

---
*This document follows the https://specscore.md/scenario-specification*
