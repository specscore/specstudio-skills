---
format: https://specscore.md/scenario-specification
---

# Scenario: per-AC git-log walk collects matching commits and dispatches only mapped ACs

**Validates:** [verify#ac:per-ac-trailer-grep](../README.md#ac-per-ac-trailer-grep-verifies-reqtrailer-grep-per-ac-reqfeature-parse)

## Steps

GIVEN an approved Feature at `spec/features/example/README.md` with three ACs `example#ac:a`, `example#ac:b`, `example#ac:c`
AND the branch contains one commit whose message includes `Verifies: example#ac:a`
AND the branch contains two commits whose messages include `Verifies: example#ac:b`
AND no commit in the branch references `example#ac:c`
WHEN the user runs `specstudio:verify example`
THEN the skill collects one matching commit for AC `a`
AND the skill collects two matching commits for AC `b`
AND the skill collects zero matching commits for AC `c`
AND the skill dispatches a subagent for AC `a`
AND the skill dispatches a subagent for AC `b`
AND the skill does NOT dispatch a subagent for AC `c`

---
*This document follows the https://specscore.md/scenario-specification*
