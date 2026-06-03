# Plan: Stage lint --fix changes from the CLI fixed-files report

**Status:** Completed
**Source Feature:** skills/lint-fix-staging
**Date:** 2026-06-02
**Owner:** alex
**Supersedes:** —

## Summary

Decomposes the [lint-fix-staging Feature](../features/skills/lint-fix-staging/README.md) into three linearly-ordered tasks that establish the shared "stage exactly what `lint --fix` reported" contract for specstudio skills: read the CLI's `.fixed[]` report and stage the union of artifacts + reported paths; report that set explicitly while staying stage-only; and guard with block-not-guess when the report is unavailable. All five source-Feature ACs are covered by tasks; none deferred.

## Approach

The three tasks map to the Feature's three behavioral concerns in implementation order. Task 1 lands the core protocol — read the machine-readable fixed-files report and stage the union — because every other behavior builds on having the authoritative changed-file set in hand. Task 2 adds the user-facing reporting and the stage-only (never commit/push) guarantee, which operate on the set Task 1 stages. Task 3 adds the degraded-mode guard (block, do not fall back to git-diff) when the running CLI lacks the report. The linear order 1 → 2 → 3 reflects that dependency: you cannot report or guard a staging step that does not yet exist.

Implementation edits the shared skill guidance under `skills/shared` (the post-`lint --fix` staging protocol) that the per-skill instructions reference. This Plan delivers the shared contract; the breadth of per-skill rollout and the upstream CLI dependency are tracked under Outstanding Questions, mirroring the Feature's own open questions.

## Tasks

### Task 1: Read the fixed-files report and stage the union

**Verifies:** skills/lint-fix-staging#ac:reads-report-not-git-diff, skills/lint-fix-staging#ac:stages-every-reported-path

Add shared guidance that, after any `specscore spec lint --fix`, obtains the modified-file set from the CLI's `--format json` `.fixed[]` array (never from `git status`/`git diff`) and `git add`s the union of the skill's own created/edited artifacts and every reported path. The guidance MUST state that no reported path may be dropped as "unrelated."

### Task 2: Report the staged set explicitly and stay stage-only

**Verifies:** skills/lint-fix-staging#ac:surfaces-staged-lint-fixes, skills/lint-fix-staging#ac:stages-never-commits

Extend the shared guidance so the skill reports the staged set back to the user, naming the lint-induced paths distinctly from its own artifacts, and explicitly prohibits `git commit`/`git push` — staging only, leaving the commit decision (including any separate-commit choice) to the user.

### Task 3: Guard with block-not-guess when the report is missing

**Verifies:** skills/lint-fix-staging#ac:blocks-when-report-missing

Add the degraded-mode rule: when the running `specscore` CLI does not emit a fixed-files report under `--fix`, the skill surfaces the missing-capability limitation to the user and does NOT reconstruct the change set from a `git status`/`git diff` heuristic. Include how the absence is detected (e.g., absence of the `fixed` key in `--format json` output, or a CLI version check).

## Open Questions

- Implementation is blocked until the upstream producer ships the report: the `specscore` Spec Lint "Fixed-files reporting" behavior (source Idea `specscore-cli:lint-fix-reports-modified-files`) must provide `.fixed[]`, and the skills must require a CLI version that includes it.
- Per-skill rollout breadth — whether each skill's existing `auto-stage-on-create` REQ references this shared contract or the staging behavior is consolidated here — is carried over from the Feature's Open Questions and should be settled before the per-skill instruction files are edited.

---
*This document follows the https://specscore.md/plan-specification*
