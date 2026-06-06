---
format: https://specscore.md/scenario-specification
---

# Scenario: spec/features/ is auto-created when missing

**Validates:** [specify#req:auto-create-features-dir](../README.md#req-auto-create-features-dir)

## Steps

GIVEN a project with no `spec/features/` directory
AND the user invokes `specstudio:specify` with a clear buildable intent
WHEN the skill is about to write the first Feature
THEN the skill creates the directory `spec/features/`
AND the skill creates `spec/features/README.md` as a lint-clean Index artifact (`type: index`, empty Contents table, "None at this time." Outstanding Questions)
AND the skill tells the user it bootstrapped the directory
AND the skill then writes `spec/features/<slug>/README.md` and at least one requirement file

GIVEN a project that already has `spec/features/` with an existing index
WHEN the skill creates a new Feature
THEN the skill MUST NOT recreate the directory
AND the skill MUST NOT overwrite the existing `spec/features/README.md`
AND the skill MUST update the index to list the new Feature

---
*This document follows the https://specscore.md/scenario-specification*
