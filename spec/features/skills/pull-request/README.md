# Feature: Pull Request Skill

> [SpecScore.**Studio**](https://specscore.studio): | [Explore](https://specscore.studio/app/github.com/specscore/specstudio-skills/spec/features/skills/pull-request?op=explore) | [Edit](https://specscore.studio/app/github.com/specscore/specstudio-skills/spec/features/skills/pull-request?op=edit) | [Ask question](https://specscore.studio/app/github.com/specscore/specstudio-skills/spec/features/skills/pull-request?op=ask) | [Request change](https://specscore.studio/app/github.com/specscore/specstudio-skills/spec/features/skills/pull-request?op=request-change) |

**Status:** Approved
**Date:** 2026-06-04
**Owner:** alexander.trakhimenok
**Source Ideas:** pull-request-skill
**Supersedes:** —
**Grade:** B

## Summary

`specstudio:pull-request` is the structural twin of `specstudio:ship`: a gate-and-dispatch skill that enforces the project's pre-PR gates on the current branch, then creates exactly one pull request, and refuses to do more. It gates (through the existing reviewer-gates layer), creates one PR (built-in `git push` + `gh pr create` by default, or a configured delegate), and emits `pull_request.created`. It never sequences, merges, deploys, or orchestrates.

## Problem

A coverage/test gate that runs only in CI fires *after* the PR is opened — the failure surfaces as a red check on an existing PR rather than being caught beforehand (specscore-cli PR #48: merged-ready, then CI flagged 99.7% against a 100% gate). There is no spec-aware checkpoint that runs the project's verify gate, plus any reviewer judgment, *before* the PR exists. A `pull-request` skill exists to enforce those gates at PR-creation time and to record the lifecycle outcome, reusing the reviewer-gates machinery rather than inventing a parallel one.

## Behavior

### Invocation and Scope

#### REQ: single-branch-input

The skill MUST operate on the current git branch only and MUST create at most one pull request per invocation. It MUST NOT accept, batch, or coordinate multiple branches or stacked pull requests in one invocation.

### Pre-Flight Machine Gates

These are hard gates the skill enforces itself before any reviewer or delegate is dispatched. Any failure refuses, creates no PR, and exits non-zero.

#### REQ: branch-preflight

The skill MUST refuse when the current branch is the repository's default branch (there is no PR from the default branch into itself), and MUST refuse when the current branch has no commits ahead of the base branch (there is nothing to open a PR for). On refusal it MUST print the reason and recommend the appropriate prior step.

### Reviewer Gate (Verify + Judgment)

#### REQ: pull-request-reviewer-gate

After the pre-flight machine gates pass, the skill MUST fire the `pull_request.pre_dispatch` gate-point event (single-fire per run) and dispatch the reviewer gate registered under it via the shared reviewer-gates [loader](../../../../skills/shared/reviewer-gates/loader.md) and [runner](../../../../skills/shared/reviewer-gates/runner.md). The gate releases only when every configured reviewer entry returns `Approved` under AND-composition. The skill MUST carry no hardcoded baseline reviewer and MUST NOT fall back to a built-in reviewer when the gate is unconfigured — it follows the loader's refuse-or-release outcome exactly.

#### REQ: deterministic-verify-reviewer

The project verify check (tests, coverage) MUST be expressed as an existing `type: deterministic` reviewer entry whose `run:` command's exit code is the verdict (zero → `Approved`; non-zero → `Issues Found` with the command's diagnostics captured as `Blocker`(s)). The skill MUST introduce no new reviewer type. A non-zero verify command therefore blocks PR creation before any push or `gh pr create` runs. The configured `run:` command MUST be read-only with respect to `spec/` per the reviewer-gates contract; a command whose only side effects are outside `spec/` (e.g. a `cover.out` at the repo root) is conformant.

### PR Creation

#### REQ: builtin-pr-creation

When the reviewer gate releases AND no `pull_request.delegate` is configured, the skill MUST create exactly one pull request itself: push the current branch to its remote, then open a ready-for-review (non-draft) pull request whose base is the repository's default branch. It MUST create the PR exactly once and MUST NOT interpose any orchestration.

#### REQ: pr-title-body-derivation

Before creating the PR, the skill MUST derive a draft title and body from the current branch's commits (title from the latest/squash subject; body summarizing the commits) and present them to the user for confirmation or edit. The PR MUST be created with the confirmed title and body.

#### REQ: optional-delegate-override

When a `pull_request.delegate` is configured in `specscore.yaml`, the skill MUST dispatch that single delegate skill instead of the built-in creation path, passing the delegate's configured `args`. It MUST invoke the delegate exactly once and MUST NOT sequence multiple delegates or retry a failed delegate.

### Lifecycle Event

#### REQ: emit-pull-request-created

On successful PR creation — the built-in path completing, or a configured delegate returning explicit success — the skill MUST emit `pull_request.created` exactly once, with publication policy applied, carrying the created PR's identifier (URL/number) in the payload. If creation fails, it MUST NOT emit the event.

### Architectural Boundary

#### REQ: no-orchestration

The skill MUST gate, create one PR, and record — nothing more. It MUST NOT merge, auto-merge, or approve the PR; MUST NOT deploy; MUST NOT sequence or stack multiple PRs; and MUST NOT retry a failed delegate. Those behaviors belong to other skills (`ship`, the merge step) or the delegate, never to this skill.

## Acceptance Criteria

### AC: rejects-default-branch (verifies REQ:single-branch-input, REQ:branch-preflight)

**Given** the current branch is the repository's default branch (`main`)
**When** `specstudio:pull-request` is invoked
**Then** the skill refuses, creates no PR, exits non-zero, and recommends creating a feature branch first.

### AC: refuses-when-no-commits-ahead (verifies REQ:branch-preflight)

**Given** a feature branch with zero commits ahead of the base branch
**When** the skill is invoked
**Then** it refuses with a "nothing to open a PR for" message and creates no PR.

### AC: gate-releases-only-on-all-approved (verifies REQ:pull-request-reviewer-gate)

**Given** a `gates.pull_request.pre_dispatch.reviewers` list whose entries all return `Approved`
**When** the skill runs the reviewer gate via the shared runner
**Then** the gate releases and the skill proceeds to PR creation; and **given** any entry returns `Issues Found`, the gate does NOT release, the `Blocker` findings are surfaced, and no PR is created.

### AC: verify-gate-blocks-pr-below-threshold (verifies REQ:deterministic-verify-reviewer)

**Given** a `type: deterministic` verify reviewer whose `run:` command (e.g. `./scripts/pull-request-gate.sh`) exits non-zero because coverage is below the project threshold
**When** the skill runs the reviewer gate
**Then** the gate returns `Issues Found` and no pull request is opened (the skill never reaches the push / `gh pr create` step).

### AC: creates-ready-pr-against-default-branch (verifies REQ:builtin-pr-creation)

**Given** the reviewer gate has released and no `pull_request.delegate` is configured
**When** the skill creates the PR
**Then** it pushes the branch and opens exactly one ready-for-review pull request whose base is the repository's default branch.

### AC: derives-and-confirms-title-body (verifies REQ:pr-title-body-derivation)

**Given** a branch with one or more commits
**When** the skill prepares to create the PR
**Then** it presents a title and body derived from the branch's commits for the user to confirm or edit, and creates the PR with the confirmed values.

### AC: dispatches-configured-delegate (verifies REQ:optional-delegate-override)

**Given** a configured `pull_request.delegate` (e.g. `commit-push-pr`) and a released gate
**When** the skill reaches PR creation
**Then** it invokes the delegate skill exactly once with the configured `args` and does not run the built-in `git push` / `gh pr create` path.

### AC: emits-created-once (verifies REQ:emit-pull-request-created)

**Given** a pull request was created successfully
**When** the skill finishes
**Then** it emits `pull_request.created` exactly once carrying the PR identifier and `publication_result`; and **given** creation failed, it emits no event.

### AC: bars-orchestration (verifies REQ:no-orchestration)

**Given** any invocation
**When** the skill runs
**Then** it never merges, auto-merges, approves, deploys, stacks multiple PRs, or retries a failed delegate.

## Not Doing / Out of Scope

- A new `cli` reviewer type — the existing `type: deterministic` already expresses the verify gate.
- Stacked, multi-PR, or cross-repo PR coordination — cross-thing orchestration is outside one-project Studio scope.
- Merge, auto-merge, or PR-approval automation — downstream of opening the PR.
- Deploy or release mechanics — that is `ship` and its delegate.
- Owning CI configuration — the skill mirrors the gate locally; CI remains the backstop.

## Dependencies

- **`events.md`** — register two new canonical events before this skill can fire its gate or signal completion: `pull_request.pre_dispatch` (gate-point, single-fire per run, mirroring `ship.pre_dispatch`) and `pull_request.created` (lifecycle, mirroring `ship.completed`). The reviewer-gates loader only accepts canonical event keys, so the gate-point key MUST exist in the catalog.
- **`reviewer-gates`** — keys a new gate-point (`gates.pull_request.pre_dispatch`) on the existing mechanism. No new reviewer type is introduced (the verify gate uses the existing `type: deterministic`).
- **`flexible-lifecycle-flows`** — adds a non-terminal PR node positioned after implement and before `ship`.
- **`change-publication-policy`** — the `pull_request.created` checkpoint is a publication-policy target.
- **`third-party-skill-integration-contracts`** — the optional `pull_request.delegate` handoff reuses the same delegate-dispatch contract as `ship`.

## Rehearse Integration

No `_tests/` stubs are scaffolded. This Feature specifies an **agent-executed skill** (prose instructions), not a unit with a direct CLI/HTTP/pure-function surface; its observable behavior is the end-to-end outcome of an agent following the instructions. Per the Rehearse heuristic, such Features skip stubs. Validation is by **dogfooding on specscore-cli**: a branch below the 100% coverage gate MUST be blocked before the PR is created (the `verify-gate-blocks-pr-below-threshold` AC), and a clean branch MUST open a ready PR against the default branch.

## Open Questions

- Where does the `pull_request:` config schema live — its own config Feature or an extension of an existing one (the same question `ship:` raised for its block)?
- Should the skill require commits to already exist, or optionally compose with `commit-push-pr` for the commit step when the working tree is dirty?
- Should the derived PR body include a link back to the Source Feature / verify report for traceability?

---
*This document follows the https://specscore.md/feature-specification*
