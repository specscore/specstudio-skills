# Scenario: Writing to docs/ideas/ is rejected

**Validates:** [ideate#req:no-docs-path](../README.md#req-no-docs-path)

## Steps

GIVEN the skill is in Phase 3 and ready to crystallize an Idea
AND the user requests the artifact be written to `docs/ideas/my-idea.md`
WHEN the skill processes the request
THEN the skill refuses to write to `docs/ideas/`
AND the skill explains that SpecScore artifacts live under `spec/`, not `docs/`
AND the skill writes to `spec/ideas/my-idea.md` instead

GIVEN an Idea artifact has been mistakenly committed to `docs/ideas/my-idea.md`
WHEN `specscore spec lint` runs against the project tree
THEN lint reports rule `U-007` (file location matches canonical path for type) as an error
AND `specstudio:specify` cannot consume the misplaced artifact

---
*This document follows the https://specscore.md/scenario-specification*
