# Scenario: recap refuses a Feature that exists only in the working tree

**Validates:** [recap#ac:refuses-uncommitted-feature](../README.md#ac-refuses-uncommitted-feature-verifies-reqrequires-feature-in-git-head)

## Steps

GIVEN a Feature exists in the working tree at `spec/features/example/README.md` with body metadata `**Status:** Approved`
AND the Feature has not been committed (i.e. `git cat-file -e HEAD:spec/features/example/README.md` exits non-zero)
AND the user runs `specstudio:recap example`
WHEN the skill reaches the pre-flight check
THEN the skill refuses to dispatch any subagent
AND the skill instructs the user to commit the Feature first
AND no report is written under `spec/features/example/_recap/`
AND the skill exits non-zero

---
*This document follows the https://specscore.md/scenario-specification*
