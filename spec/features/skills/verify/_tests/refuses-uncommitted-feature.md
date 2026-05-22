# Scenario: verify refuses a Feature that is not committed to git HEAD

**Validates:** [verify#ac:refuses-uncommitted-feature](../README.md#ac-refuses-uncommitted-feature-verifies-reqrequires-feature-in-git-head)

## Steps

GIVEN a Feature exists at `spec/features/example/README.md` with `**Status:** Approved`
AND the Feature exists in the working tree but has not been committed
AND `git cat-file -e HEAD:spec/features/example/README.md` exits non-zero
WHEN the user runs `specstudio:verify example`
THEN the skill refuses to dispatch any subagent
AND the skill instructs the user to commit the Feature first
AND no report is written under `spec/features/example/_verify/`
AND the skill exits non-zero

---
*This document follows the https://specscore.md/scenario-specification*
