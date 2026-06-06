---
format: https://specscore.md/scenario-specification
---

# Scenario: CLI path is preferred when specscore is available

**Validates:** [ideate#ac:cli-vs-fallback](../README.md#ac-cli-vs-fallback)

## Steps

GIVEN the `specscore` binary is on PATH
AND `specscore --version` exits zero
AND the skill is in Phase 3 with a complete set of inputs (title, owner, hmw, recommended-direction, mvp, not-doing entries)
WHEN the skill crystallizes the Idea
THEN the skill invokes `specscore idea new <slug>` with the documented flags only
AND the skill does NOT invoke any undocumented flag
AND the resulting artifact at `spec/ideas/<slug>.md` is lint-clean on generation
AND `spec/ideas/README.md` is updated by the CLI to include the new Idea

GIVEN sections without matching CLI flags exist (e.g., `Alternatives Considered`, `Key Assumptions to Validate`)
WHEN the skill fills those sections
THEN it uses the `Edit` tool to replace the CLI-generated HTML-comment prompts with real content
AND the artifact remains lint-clean after the edits

---
*This document follows the https://specscore.md/scenario-specification*
