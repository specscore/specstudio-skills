# Idea: Pull Request Skill

**Status:** Specified
**Date:** 2026-06-04
**Owner:** alexander.trakhimenok
**Promotes To:** skills/pull-request
**Supersedes:** —
**Related Ideas:** —

## Problem Statement

How might we let a SpecStudio user open a pull request only after every project-configured gate (tests, coverage, optional review) has passed locally, so failures are caught before the PR exists rather than after CI flags them?

## Context

Triggered by specscore-cli PR #48: a branch was merged-ready, then CI flagged total coverage at 99.7 percent against a 100 percent gate. The gap is timing: gating runs post-PR in CI, not before the PR is opened. SpecStudio already has the machinery to close this. The reviewer-gates layer defines a type: deterministic reviewer whose verdict comes from a run: command exit code (zero approves, non-zero is Issues Found with diagnostics as Blockers), dispatched through the shared loader and runner against a gates: block in specscore.yaml. The ship skill is the proven gate-and-dispatch pattern this mirrors. The loader recognizes canonical pre-action gate-point event keys such as implementation.pre_push. A pull-request skill should compose these rather than invent new machinery.

## Recommended Direction

Build pull-request as the structural twin of ship: a thin gate-and-dispatch skill that gates, then creates one PR, and refuses to do more. The gate reuses reviewer-gates unchanged. The verify check (tests, coverage) is expressed as an existing type: deterministic entry whose run: command is the project verify command (for specscore-cli, scripts/pull-request-gate.sh, which delegates to the shared scripts/coverage-gate.sh), alongside optional type: ai and type: human reviewers, all under a gates block keyed on a pull-request gate-point event and dispatched AND-composed through the shared runner. No new reviewer type is needed; this was the key correction during ideation. On release, the skill creates the PR itself by default (git push then gh pr create), since PR creation is low-blast-radius and standardized unlike deploy; a slim optional pull_request: delegate block can override the built-in path with a project skill such as commit-push-pr. On success it emits pull_request.created with publication policy applied. The discipline that keeps this coherent mirrors ship: gate, create one PR, record, never orchestrate. Stacked or multi-PR coordination, merge, and deploy are explicitly barred. Why this over the alternatives: a git pre-push hook enforces for everyone but lives per-clone outside the spec graph and cannot run AI or human reviewers; pushing gating fully into CI is the status quo that opens the PR before failing. Gate-then-create is the smallest thing that catches failures before the PR exists while staying inside one-project Studio scope.

## Alternatives Considered

- **Git pre-push hook only.** A `pre-push` hook running the verify command enforces for every pusher (human or agent) regardless of how the PR is opened, and it is the only airtight local backstop. Lost as the *primary* mechanism: it lives per-clone outside the spec graph (must be installed with `core.hooksPath`, easily bypassed with `--no-verify`), runs no `type: ai` or `type: human` reviewer, and emits no lifecycle event. Complementary, not a substitute — pair the two (the hook is the hard floor, the skill adds reviewers + traceable events).
- **CI-only gating (status quo).** Let the `pull_request` CI workflow be the sole gate. Lost: this is exactly the failure mode that motivated the Idea — the PR is opened *before* the gate runs, so failures surface as red checks on an existing PR instead of being caught beforehand. CI stays the backstop, but it cannot be the only gate if the goal is "no PR opens below the bar."
- **New `cli` reviewer type in reviewer-gates.** Add a bespoke reviewer type that shells out to the verify command. Lost: reviewer-gates already ships `type: deterministic` (verdict from a `run:` command exit code), which expresses the verify gate exactly. A second type fragments the gate config and re-implements tested machinery — the same reason the ship idea rejected a fresh gate block.

## MVP Scope

A single-skill, current-branch flow: pre-flight the branch, fire the pull-request gate-point event, run gates.pull_request.pre_dispatch through the shared reviewer-gates runner (a type: deterministic verify reviewer plus any optional ai or human reviewers), and only on release create exactly one PR via the built-in git push then gh pr create path (or a configured delegate). Emit pull_request.created with publication policy. Prove it by dogfooding on specscore-cli: a branch below 100 percent coverage must be blocked before the PR is created, and a clean branch must open the PR.

## Not Doing (and Why)

- A new reviewer type (cli) — the existing type: deterministic run-command-by-exit-code already expresses the verify gate
- Stacked, multi-PR, or cross-repo PR coordination — cross-thing orchestration is outside one-project Studio scope
- Merge, auto-merge, or PR-approval automation — downstream of opening the PR, a separate concern
- Deploy or release mechanics — that is ship and its delegate, not PR creation
- Owning CI configuration — the skill mirrors the gate locally; CI remains the backstop

## Key Assumptions to Validate

| Tier | Assumption | How to validate |
|------|------------|-----------------|
| Must-be-true | A canonical gate-point event key for the PR checkpoint exists or can be added, since the loader rejects bare skill names and only accepts canonical event keys. | Decide between reusing `implementation.pre_push` and registering a new `pull_request.pre_dispatch` (plus a `pull_request.created` lifecycle event) in `events.md`; confirm the loader accepts the chosen key. |
| Must-be-true | A `type: deterministic` reviewer running the project verify command satisfies the read-only contract (reviewers MUST NOT write under `spec/`). | Run `scripts/pull-request-gate.sh` (which delegates to `scripts/coverage-gate.sh`) as a deterministic reviewer on specscore-cli; confirm its only writes (`cover.out` at repo root) are outside `spec/` and do not trip the misclassified-Producer refusal. |
| Should-be-true | Built-in `git push` + `gh pr create` covers the common case so most projects need no `pull_request:` delegate. | Dogfood on specscore-cli with no delegate configured; confirm a PR opens end-to-end and the delegate override is genuinely optional. |
| Should-be-true | The PR checkpoint belongs in the lifecycle after implement and before ship, as a non-terminal node. | Validate against `flexible-lifecycle-flows`; confirm where the PR node sits relative to `implementation.pre_push` and `ship`. |
| Might-be-true | Teams will want an AI PR-description / diff-sanity reviewer distinct from the spec/score reviewers. | Defer; observe whether projects configure a `type: ai` entry on the PR gate. |


## SpecScore Integration

- **New Features this would create:** a `skills/pull-request` Feature (gate-and-create-one-PR); a slim `pull_request:` config schema (delegate override), likely folded into an existing config Feature.
- **Existing Features affected:** `reviewer-gates` (a new gate-point event key on the `gates:` map, but no new reviewer type); `flexible-lifecycle-flows` (a non-terminal PR node before `ship`); `change-publication-policy` (the `pull_request.created` checkpoint); `third-party-integration` (the optional delegate handoff contract, shared with ship).
- **Dependencies:** the canonical gate-point event decision (reuse `implementation.pre_push` vs new `pull_request.pre_dispatch`); `third-party-skill-integration-contracts` (Idea) for the delegate handoff shape.

## Open Questions

- Reuse the existing `implementation.pre_push` gate-point event, or register a new `pull_request.pre_dispatch` (and a `pull_request.created` lifecycle event) in `events.md`?
- Built-in PR creation defaults: target base branch (always `main`/default branch?), draft vs ready, and how the PR title/body are derived (last commit, branch name, or an `--title/--body` prompt).
- When the gate releases but the branch has no commits ahead of base, refuse or no-op?
- Where does the `pull_request:` schema live — its own config Feature or an extension of an existing one (same question ship raised for `ship:`)?
- Should the skill verify a clean working tree / require commits be made first, or compose with `commit-push-pr` for the commit step?

---
*This document follows the https://specscore.md/idea-specification*
