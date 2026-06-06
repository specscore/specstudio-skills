---
format: https://specscore.md/scenario-specification
---

# Scenario: recap walks `git log --grep` for each AC's Verifies trailer

**Validates:** [recap#ac:per-ac-trailer-grep](../README.md#ac-per-ac-trailer-grep-verifies-reqtrailer-grep-per-ac-reqfeature-parse)

## Steps

GIVEN an Approved Feature `example` with three ACs `example#ac:a`, `example#ac:b`, and `example#ac:c`
AND a resolvable verify report at HEAD
AND the branch history contains one commit with `Verifies: example#ac:a` in its message
AND two commits with `Verifies: example#ac:b` in their messages
AND zero commits referencing `example#ac:c`
WHEN the user runs `specstudio:recap example`
THEN the skill collects exactly one commit for AC `a`
AND the skill collects exactly two commits for AC `b`
AND the skill collects zero commits for AC `c`
AND the skill dispatches drift-narrator subagents only for ACs `a` and `b`

---
*This document follows the https://specscore.md/scenario-specification*
