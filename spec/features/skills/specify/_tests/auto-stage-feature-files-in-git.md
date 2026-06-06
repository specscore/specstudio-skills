---
format: https://specscore.md/scenario-specification
---

# Scenario: All Feature files the skill creates are auto-staged in git

**Validates:** [specify#req:auto-stage-on-create](../README.md#req-auto-stage-on-create)

## Steps

GIVEN a project with `spec/features/` already initialized
AND the project is a git repository with no other staged changes
WHEN the skill writes `spec/features/checkout-v2/README.md`, `spec/features/checkout-v2/requirements/payment.md`, and `spec/features/checkout-v2/_tests/payment-rejected-without-token.md`
THEN the skill runs `git add` on every created path
AND all three files appear in the git index
AND the skill tells the user the staged paths
AND the skill MUST NOT run `git commit`

GIVEN a project that does NOT yet have `spec/features/` (bootstrap path)
AND the project is a git repository
WHEN the skill bootstraps the directory and writes the index README plus the Feature directory plus its contents
THEN the skill stages every created file in a single response
AND the skill tells the user the full list of staged paths
AND the skill MUST NOT run `git commit`

GIVEN the working directory is NOT a git repository (or git is not on PATH)
WHEN the skill writes the Feature
THEN the skill MUST NOT abort the artifact write
AND the skill MUST surface the staging failure to the user with the underlying reason
AND the artifact remains on disk and lint-clean

GIVEN the user has unrelated staged changes prior to invocation
WHEN the skill creates and stages the new Feature files
THEN only the paths the skill created are added to the index
AND the user's pre-existing staged changes remain untouched

---
*This document follows the https://specscore.md/scenario-specification*
