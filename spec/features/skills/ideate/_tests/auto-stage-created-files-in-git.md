---
format: https://specscore.md/scenario-specification
---

# Scenario: Created files are auto-staged in git with explicit user notification

**Validates:** [ideate#req:auto-stage-on-create](../README.md#req-auto-stage-on-create)

## Steps

GIVEN a project that already has `spec/ideas/` initialized
AND the project is a git repository with no other staged changes
WHEN the skill writes `spec/ideas/my-idea.md` for the first time
THEN the skill runs `git add spec/ideas/my-idea.md`
AND `spec/ideas/my-idea.md` appears in the git index
AND the skill tells the user the path was staged
AND the skill MUST NOT run `git commit`

GIVEN a project that does NOT yet have `spec/ideas/` (bootstrap path)
AND the project is a git repository
WHEN the skill bootstraps the directory and writes both `spec/ideas/README.md` (the index) and `spec/ideas/my-idea.md`
THEN the skill stages BOTH files in a single response
AND the skill tells the user both staged paths
AND the skill MUST NOT run `git commit`

GIVEN the working directory is NOT a git repository (or git is not on PATH)
WHEN the skill writes `spec/ideas/my-idea.md`
THEN the skill MUST NOT abort the artifact write
AND the skill MUST surface the staging failure to the user with the underlying reason
AND the artifact remains on disk and lint-clean

GIVEN the user has unrelated staged changes prior to invocation
WHEN the skill creates and stages `spec/ideas/my-idea.md`
THEN only the path the skill created is added to the index
AND the user's pre-existing staged changes remain untouched

---
*This document follows the https://specscore.md/scenario-specification*
