---
captured_by: user
status: queued
---
# Allow full ideate-to-recap lifecycle on a feature branch without auto-merging to main

The specstudio producer skills (ideate, specify, plan, implement, verify, recap) currently lead the agent to merge each approved artifact into `main` between phases so the next phase can find the source artifact "in the tree". This is wrong: a committed-but-unmerged artifact on a feature branch is already in the tree and sufficient for the downstream skill. Merging to `main` mid-lifecycle is (a) not required, (b) not user-authorized, and (c) not the expected workflow.

**Desired behavior:** support running the entire ideate → specify → plan → implement → verify → recap chain on ONE feature branch, accumulating commits, and merging into `main` exactly once at the end via a single PR. Phase transitions should rely on the source artifact existing in the working tree / current branch, not on it being merged to `main`.

**Hard rule:** the skills MUST NOT merge to `main` (or instruct the agent to) without explicit user authorization. Merge-to-main is an explicit, user-authorized step — never implicit between phases.

**Action:** update the specstudio-skills (ideate/specify/plan/implement/verify/recap + shared publication-policy) so phase transitions branch-and-commit rather than merge, and document the single-PR-at-end flow. Observed 2026-06-03: an agent merged an approved Idea PR into main mid-flow to "enable" specify — unauthorized and unexpected.
