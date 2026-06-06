---
format: https://specscore.md/scenario-specification
---

# Scenario: spec/ideas/ is auto-created when missing

**Validates:** [ideate#req:auto-create-ideas-dir](../README.md#req-auto-create-ideas-dir)

## Steps

GIVEN a project with no `spec/ideas/` directory
AND the user invokes `specstudio:ideate` with a vague concept
WHEN the skill reaches Phase 3 and is about to write the artifact
THEN the skill creates the directory `spec/ideas/`
AND the skill creates `spec/ideas/README.md` as a lint-clean Index artifact (`type: index`, empty Contents table, "None at this time." Outstanding Questions)
AND the skill tells the user it bootstrapped the directory
AND the skill then writes `spec/ideas/<slug>.md` as planned

GIVEN a project that already has `spec/ideas/` with an existing index
WHEN the skill reaches Phase 3
THEN the skill MUST NOT recreate the directory
AND the skill MUST NOT overwrite the existing `spec/ideas/README.md`
AND the skill writes `spec/ideas/<slug>.md` as planned

---
*This document follows the https://specscore.md/scenario-specification*
