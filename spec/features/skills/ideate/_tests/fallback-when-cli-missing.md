---
format: https://specscore.md/scenario-specification
---

# Scenario: Fallback to direct write when specscore CLI is missing

**Validates:** [ideate#ac:cli-vs-fallback](../README.md#ac-cli-vs-fallback)

## Steps

GIVEN the `specscore` binary is NOT on PATH
AND the skill is in Phase 3 with a complete set of inputs
WHEN the skill crystallizes the Idea
THEN the skill writes `spec/ideas/<slug>.md` directly using the authoritative schema documented in `skills/ideate/SKILL.md`
AND the resulting artifact is byte-equivalent (modulo whitespace) to what `specscore idea new <slug>` would have produced
AND `specscore spec lint` exits zero when the CLI is later installed and re-run against the same file

GIVEN the fallback path was used
WHEN the user inspects the artifact's title, body metadata, and section structure
THEN the title is `# Idea: <name>` (the `Idea:` prefix is the dispatch key)
AND the body metadata immediately after the title contains the required fields in order (`**Status:**`, `**Date:**`, `**Owner:**`) followed by `**Promotes To:**`, `**Supersedes:**`, `**Related Ideas:**` (with value `—` when empty)
AND every required Idea section (`Problem Statement`, `Recommended Direction`, `MVP Scope`, `Not Doing`, `Key Assumptions to Validate`) is present and non-empty

---
*This document follows the https://specscore.md/scenario-specification*
