# Feature: Stage lint --fix changes from the CLI fixed-files report

> [SpecScore.**Studio**](https://specscore.studio): | [Explore](https://specscore.studio/app/github.com/specscore/specstudio-skills/spec/features/skills/lint-fix-staging?op=explore) | [Edit](https://specscore.studio/app/github.com/specscore/specstudio-skills/spec/features/skills/lint-fix-staging?op=edit) | [Ask question](https://specscore.studio/app/github.com/specscore/specstudio-skills/spec/features/skills/lint-fix-staging?op=ask) | [Request change](https://specscore.studio/app/github.com/specscore/specstudio-skills/spec/features/skills/lint-fix-staging?op=request-change) |
**Status:** Approved
**Source Ideas:** autostage-lint-fix-modified-files

## Summary

Specstudio skills that run `specscore spec lint --fix` stage exactly the files the CLI reports as modified — derived from the CLI's machine-readable fixed-files report, not inferred from a `git diff`. This shared contract is referenced by every skill whose lifecycle includes a `lint --fix` step.

## Problem

Every specstudio skill (`ideate`, `specify`, `plan`, `implement`) runs `specscore spec lint --fix` and then `git add`s a *known* set of artifact paths. Anything `--fix` touches outside that known set — chiefly index/footer/lifecycle syncs — is invisible to the staging step. In a prior session a skill dropped exactly such a line from its commit, reasoning it was "unrelated"; it was a direct consequence of the skill's own edit, the skill simply had no authoritative signal for what `--fix` changed.

The auto-stage behavior is duplicated per skill today (each skill Feature carries its own `auto-stage-on-create` REQ), so there is no single place that says "stage what `lint --fix` reported." This Feature establishes that shared contract. It depends on the upstream `specscore-cli` Feature **Spec Lint** (source Idea `specscore-cli:lint-fix-reports-modified-files`), which produces the fixed-files report this Feature consumes; the dependency is recorded in prose because the linter does not support cross-repo dependency links.

## Behavior

### Consuming the fixed-files report

A skill must learn what `--fix` changed from the CLI, never by guessing.

#### REQ: read-fixed-files-report

After any `specscore spec lint --fix` invocation, a specstudio skill MUST obtain the set of files the CLI reports as modified by reading the CLI's machine-readable fixed-files report (the `fixed` array of `specscore spec lint --fix --format json`). The skill MUST NOT reconstruct the modified-file set from `git status` or `git diff`.

#### REQ: stage-union-of-artifacts-and-fixed

A skill MUST `git add` the union of (the artifacts the skill itself created or edited) and (every path the CLI reported in `fixed`). No reported path may be omitted on the grounds that it appears unrelated to the skill's primary change.

### Reporting to the user

#### REQ: report-staged-lint-fixes

When a skill stages one or more lint-fix-reported paths, it MUST report the staged set back to the user, naming the lint-induced paths explicitly and distinctly from the skill's own artifacts, so no staged change is introduced silently.

### Stage-only contract preserved

#### REQ: never-commit-lint-fixes

A skill MUST NOT commit or push the staged changes. Staging is the skill's responsibility; the commit — including any decision to split lint syncs into a separate commit — remains the user's. This preserves manual stage/commit/push workflows.

### Degrading safely when the report is absent

#### REQ: block-not-guess-without-report

When the running `specscore` CLI does not provide the fixed-files report (an older CLI predating the upstream Spec Lint behavior), the skill MUST surface that limitation to the user and MUST NOT fall back to a `git status`/`git diff` heuristic to reconstruct the change set.

## Acceptance Criteria

### AC: reads-report-not-git-diff (verifies REQ:read-fixed-files-report)

**Given** a skill has just run `specscore spec lint --fix` that modified one or more files
**When** the skill determines what to stage
**Then** it derives the modified-file set from the CLI's `--format json` `fixed` array, and does not parse `git status`/`git diff` output to build that set.

### AC: stages-every-reported-path (verifies REQ:stage-union-of-artifacts-and-fixed)

**Given** `lint --fix` reports a fixed file (e.g. an index README or lifecycle sync) that the skill did not directly author
**When** the skill stages its work
**Then** that reported path is included in the `git add` set alongside the skill's own artifacts, and no reported path is dropped as "unrelated".

### AC: surfaces-staged-lint-fixes (verifies REQ:report-staged-lint-fixes)

**Given** the skill staged one or more lint-fix-reported paths
**When** it reports back to the user
**Then** it lists those paths explicitly, labeled as lint-induced and separate from the skill's own artifacts.

### AC: stages-never-commits (verifies REQ:never-commit-lint-fixes)

**Given** the skill has staged its artifacts and the lint-fix-reported paths
**When** the skill finishes its run
**Then** no `git commit` or `git push` has been executed — only the git index has been updated.

### AC: blocks-when-report-missing (verifies REQ:block-not-guess-without-report)

**Given** the installed `specscore` CLI does not emit a fixed-files report under `--fix`
**When** a skill runs `specscore spec lint --fix`
**Then** the skill surfaces the missing-capability limitation to the user and does not reconstruct the change set from a `git diff`.

## Rehearse Integration

No Rehearse stubs are scaffolded. Every AC here asserts **agent behavior** during a skill run (which signal the agent reads, what it stages, what it reports, what it refuses to do) rather than an observable CLI/HTTP/pure-function/filesystem surface a Rehearse scenario can drive deterministically. These ACs are verified by skill-execution review, consistent with how the sibling skill Features (`ideate`, `plan`) treat their behavioral REQs. If the staging logic is later extracted into a testable helper, stubs should be added at that point.

## Open Questions

- What minimum `specscore` CLI version should the skills require, and how is an older CLI (one lacking the fixed-files report) detected at runtime to trigger `REQ:block-not-guess-without-report`?
- How is this shared contract wired into the per-skill Features (`ideate`, `specify`, `plan`, `implement`) — do their `auto-stage-on-create` REQs reference this Feature, or is the staging behavior consolidated here entirely? (The source Idea assumed a single shared home; today each skill duplicates the REQ. Resolving this is follow-on work.)
- Should cross-repo dependency links (this Feature → `specscore-cli:lint-fix-reports-modified-files`) become a structured field once the linter supports them, instead of prose?

---
*This document follows the https://specscore.md/feature-specification*
