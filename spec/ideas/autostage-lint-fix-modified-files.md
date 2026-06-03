# Idea: Auto-stage lint --fix changes via the CLI fixed-files report

**Status:** Specified
**Date:** 2026-06-02
**Owner:** alexander.trakhimenok
**Promotes To:** skills/lint-fix-staging
**Supersedes:** —
**Related Ideas:** —

## Problem Statement

How might we stop specstudio skills from silently dropping files that `specscore spec lint --fix` modified, by having every skill stage exactly the paths the CLI reports as fixed instead of guessing from a git diff?

## Context

Triggered by a real failure: a specstudio skill ran `specscore spec lint --fix`, which repaired one line (an index/footer sync caused by the agent's own edits), then the agent omitted that line from its commit — reasoning it was "unrelated." It was related; the agent simply had no authoritative signal for what `--fix` had touched, so it fell back to eyeballing a git diff and guessed wrong.

This is the **consumer half** of a two-repo fix. The **producer half** — making `specscore spec lint --fix` report the exact set of files it modified — is the upstream Idea `specscore-cli:lint-fix-reports-modified-files` (Approved). This Idea cannot start until that report exists. (Cross-repo `**Related Ideas:**` links aren't supported by the linter today — `idea-related-ideas-target-exists` requires a same-repo slug — so the link to the upstream Idea is recorded here in prose.)

Today the specstudio skills (`ideate`, `specify`, `plan`, `implement`, plus the shared auto-stage guidance) instruct the agent to `git add` a known set of artifact paths after `lint --fix`. Any file `--fix` touches *outside* that known set — chiefly index/footer syncs — is invisible to the staging step and gets dropped.

## Recommended Direction

Make the skills stage exactly what the CLI says it changed — never what the agent infers.

After any `specscore spec lint --fix` invocation, a skill reads the CLI's fixed-files report (`--format json` → `.fixed[]`), then `git add`s the **union** of (the artifacts the skill created) ∪ (every path the CLI reports as fixed), and reports that exact set back to the user. The agent never decides a fix is "unrelated" and never reconstructs the change set from `git status`/`git diff` — the CLI is the source of truth.

The skill contract stays **stage-only**: the agent stages and reports; the human owns the commit (and any decision to split lint syncs into a separate commit). When the CLI report is unavailable (older CLI without the upstream capability), the skill blocks and surfaces the limitation rather than silently reverting to the brittle git-diff guess that caused the original bug.

## Alternatives Considered

- **Keep the git-diff heuristic but make it smarter** (e.g. parse `git status --porcelain` after `--fix`). Rejected: it cannot distinguish lint's edits from the agent's own concurrent edits, and it is exactly the class of guess that produced the original dropped-line bug. The point is to stop inferring.
- **Auto-commit the lint syncs in a separate `chore` commit.** Tempting for clean history, but it breaks the stage-only contract every specstudio skill shares and overrides the user's manual stage/commit/push workflow. Rejected; staging + reporting leaves the commit decision with the human.
- **Have each skill independently re-implement fixed-files staging.** Rejected in favor of updating the single shared auto-stage guidance (`skills/shared`) so `ideate`/`specify`/`plan`/`implement` inherit one consistent behavior.

## MVP Scope

A one-to-two-day change to the shared skill guidance, consumed by all skills that run `lint --fix`:

1. Update the shared auto-stage step (`skills/shared`) so that after `lint --fix` the skill reads the CLI's `.fixed[]` report and `git add`s the union of created artifacts and reported fixed paths.
2. Have the skill report the exact staged set (artifacts + lint syncs) to the user, naming the lint-induced paths explicitly so nothing looks "unrelated."
3. Roll the updated step into `ideate`, `specify`, `plan`, and `implement`; add a guard that blocks (not guesses) when the CLI lacks the fixed-files report.

Done when running any specstudio skill that triggers `lint --fix` stages every file the CLI changed and tells the user, with zero reliance on git-diff inference. Depends on `specscore-cli:lint-fix-reports-modified-files` shipping first.

## Not Doing (and Why)

- Auto-committing the staged changes — skills stay stage-only; the human owns the commit and its splitting
- Hunk-level separation of lint's edits from the agent's own edits in a shared file — stage at file granularity, mirroring the CLI contract
- Building the fixed-files report itself — that is the upstream specscore-cli Idea this depends on, not work for this repo
- Reintroducing a git status/diff fallback when the report is unavailable — block on the upstream CLI capability rather than restore the brittle guess that caused the original bug

## Key Assumptions to Validate

| Tier | Assumption | How to validate |
|------|------------|-----------------|
| Must-be-true | The upstream CLI ships a machine-readable fixed-files report (`.fixed[]`) | Track `specscore-cli:lint-fix-reports-modified-files` to Stable; pin the minimum CLI version the skills require |
| Must-be-true | The auto-stage logic lives in one shared place the skills inherit, so a single edit covers all of them | Confirm `skills/shared` is the actual source of the auto-stage step and that `ideate`/`specify`/`plan`/`implement` consume it rather than duplicating it |
| Should-be-true | Staging the union (artifacts ∪ fixed paths) and reporting it satisfies users without a separate commit | Walk the dropped-line scenario through the new step; confirm the previously-lost path is staged and surfaced |
| Might-be-true | Users will sometimes want lint syncs in a separate commit | Defer; the human can split the staged set manually — only revisit if requested repeatedly |


## SpecScore Integration

- **New Features this would create:** "Stage lint --fix changes from the CLI fixed-files report" (shared skill auto-stage step).
- **Existing Features affected:** the auto-stage step in the `ideate`, `specify`, `plan`, and `implement` skills, plus the shared guidance they inherit.
- **Dependencies:** **`specscore-cli:lint-fix-reports-modified-files`** (Approved, upstream repo) — this Idea consumes the `.fixed[]` report that Idea produces and cannot ship before it. Recorded in prose because the linter does not support cross-repo `**Related Ideas:**` targets.

## Open Questions

- What minimum `specscore` CLI version should the skills require, and how do they detect/guard against an older CLI lacking the fixed-files report?
- Is the auto-stage behavior genuinely centralized in `skills/shared`, or does each skill carry its own copy that must be updated in lockstep?
- Should cross-repo `**Related Ideas:**` references become a real linter feature so this dependency can be structured rather than prose? (Latent upstream idea.)

---
*This document follows the https://specscore.md/idea-specification*
