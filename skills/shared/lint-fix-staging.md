# Lint --fix Staging Contract

**Status:** Contract — shared by specstudio skills that run `specscore spec lint --fix`.

## Purpose

Every specstudio skill whose lifecycle includes a `specscore spec lint --fix` step must stage **exactly** the files that fix pass changed — no more, no less. A `--fix` run touches not only the artifact the skill authored but also index READMEs, footers, and lifecycle syncs that are direct consequences of that artifact's edit. A skill has no authoritative signal for those side-effect paths unless it reads the CLI's own report. This contract makes the CLI's machine-readable fixed-files report the single source of truth for what to stage, so nothing the fix pass changed is ever dropped as "unrelated" and nothing is staged silently.

## The producer signal

`specscore spec lint --fix --format json` emits a single stdout object:

    { "fixed": [ <project-relative paths> ], "violations": [ ... ] }

The `fixed` array lists every file the fix pass modified, **project-relative** — each entry is ready to pass straight to `git add`. The report is default-on; no enabling flag is required. In default text format, `--fix` writes a `Fixed N file(s):` summary to stderr. Older CLIs predating this behavior emit **no** `fixed` key.

## Protocol

After any `specscore spec lint --fix`, the skill MUST follow these steps in order:

1. **Run the fix pass in machine-readable mode.** Invoke `specscore spec lint --fix --format json` and read its stdout object. The skill MUST obtain the modified-file set from the `.fixed[]` array — it MUST NOT reconstruct that set by parsing `git status` or `git diff` output.

2. **Degraded-mode guard (do this before staging anything).** If the running CLI did not emit a fixed-files report — detected via **absence of the `fixed` key** in the `--format json` output, or via a CLI version check — the skill MUST surface the missing-capability limitation to the user and MUST NOT fall back to a `git status`/`git diff` heuristic to reconstruct the change set. The skill stops the auto-stage step here; it does not guess.

3. **Stage the union.** When the report is present, the skill MUST `git add` the **union** of:
   - the artifacts the skill itself created or edited, and
   - every path in the CLI's `.fixed[]` array.

   No reported path may be omitted on the grounds that it looks unrelated to the skill's primary change. Every `.fixed[]` entry is staged.

4. **Report the staged set back to the user.** When one or more lint-fix-reported paths were staged, the skill MUST report the staged set, listing the lint-induced paths **explicitly** and **distinctly** from the skill's own artifacts (for example, under a separate "lint-fix syncs" label). No staged change is introduced silently.

5. **Never commit.** The skill MUST NOT run `git commit` or `git push`. Only the git index is updated. The commit — including any decision to split lint syncs into a separate commit — belongs to the user. This preserves manual stage/commit/push workflows.

## Rules

- **reads-report-not-git-diff:** The modified-file set comes from the CLI's `--format json` `.fixed[]` array only. The skill does not parse `git status`/`git diff` to build that set.
- **stages-every-reported-path:** The skill `git add`s the union of (its own artifacts) and (every `.fixed[]` path). No reported path is dropped as "unrelated" — for example, an index README or a lifecycle sync the skill did not directly author is staged alongside the skill's artifacts.
- **surfaces-staged-lint-fixes:** When ≥1 lint-fix-reported path is staged, the skill lists those paths explicitly, labeled as lint-induced and separate from the skill's own artifacts, so nothing is staged silently.
- **stages-never-commits:** No `git commit` or `git push` runs. Staging only; the user owns the commit.
- **blocks-when-report-missing:** When the running CLI does not emit a fixed-files report under `--fix` (older CLI — no `fixed` key, or a failing version check), the skill surfaces the limitation and does not reconstruct the change set from a `git status`/`git diff` heuristic.

## Adoption

The per-skill auto-stage steps (`specstudio:ideate`, `specstudio:specify`, `specstudio:plan`, `specstudio:implement`) reference this contract as the single home for the post-`lint --fix` staging protocol, rather than each duplicating the rule.

The precise per-skill wiring — whether each skill's existing `auto-stage-on-create` step links here or the staging behavior is consolidated entirely into this contract — is tracked as follow-on work, per the source Feature's open questions. The individual skill files are **not** edited as part of establishing this contract.
