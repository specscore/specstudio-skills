# Scenario: Lint failure triggers one --fix pass, then surfaces remaining violations

**Validates:** [ideate#ac:lint-failure-recovery](../README.md#ac-lint-failure-recovery)

## Steps

GIVEN the skill has just written or edited `spec/ideas/my-idea.md`
AND `specscore spec lint` exits non-zero (e.g., trailing whitespace + missing terminal newline — both auto-fixable)
WHEN the skill processes the lint failure
THEN the skill runs `specscore spec lint --fix` exactly once
AND the skill re-runs `specscore spec lint`
AND lint now exits zero
AND the skill tells the user what was auto-fixed
AND the skill continues to the next workflow step

GIVEN the skill has just written or edited the artifact
AND `specscore spec lint` exits non-zero with a violation that `--fix` cannot repair (e.g., `I-005` placeholder leak)
WHEN the skill processes the lint failure
THEN the skill runs `specscore spec lint --fix` once
AND the skill re-runs `specscore spec lint`
AND lint still exits non-zero
AND the skill MUST NOT run `--fix` a second time
AND the skill surfaces the remaining violations to the user with rule IDs and affected sections

GIVEN the skill has just written or edited the artifact
AND `specscore spec lint --fix` silently repairs a violation that arguably required human input (e.g., it inserts a placeholder `Not Doing` entry)
WHEN the skill processes the post-`--fix` lint state
THEN the skill MUST NOT add its own carve-out for that rule
AND the skill MUST treat the silent fix as a `specscore` CLI bug to be reported against the CLI repo, not as something to compensate for in the skill

---
*This document follows the https://specscore.md/scenario-specification*
