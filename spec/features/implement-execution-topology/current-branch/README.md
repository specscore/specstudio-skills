# Feature: Current-Branch Topology (Opt-In, No Isolation)

> [SpecScore.**Studio**](https://specscore.studio): | [Explore](https://specscore.studio/app/github.com/specscore/specstudio-skills/spec/features/implement-execution-topology/current-branch?op=explore) | [Edit](https://specscore.studio/app/github.com/specscore/specstudio-skills/spec/features/implement-execution-topology/current-branch?op=edit) | [Ask question](https://specscore.studio/app/github.com/specscore/specstudio-skills/spec/features/implement-execution-topology/current-branch?op=ask) | [Request change](https://specscore.studio/app/github.com/specscore/specstudio-skills/spec/features/implement-execution-topology/current-branch?op=request-change) |

**Status:** Approved
**Date:** 2026-06-02
**Owner:** alexander.trakhimenok
**Source Ideas:** implement-execution-topology
**Supersedes:** —

## Summary

Explicit-opt-in realization of the `implement-execution-topology` contract where agents work directly in the current checkout/branch (including `main`) with no per-agent worktree. The three branch roles collapse onto the one checked-out branch; execution is serial by default; and a clean-tree precondition plus a pre-run revert ref substitute for the missing worktree isolation. This is the cautious escape hatch, not the default.

## Problem

Some workflows need `implement` to operate directly on the branch the user is sitting on — to iterate in place, or because a worktree is undesirable. The parent contract permits this as an explicit opt-in but leaves its realization unspecified. Running without worktree isolation removes the very safety net that makes autonomous runs revertable, so this topology must (a) define how transitions behave when all three roles are the same branch, (b) avoid the parallel-agent clobbering that isolation otherwise prevents, and (c) reconstruct a revert path. It inherits the parent's vocabulary, gate semantics, and opt-in mechanism and MUST NOT redefine them.

## Behavior

### Role Collapse

#### REQ: roles-collapse-to-current-branch

In this topology the three roles — work context, plan primary branch, and canonical target — MUST all map to the single checked-out branch. `implement` MUST NOT create a separate plan primary branch or any per-agent worktree in this topology.

### Opt-In and Protected-Branch Guard

#### REQ: explicit-opt-in-required

This topology MUST be used only when the current-branch opt-in is set per the parent's `current-branch-opt-in-is-persisted-preference`. Absent the opt-in, `implement` MUST fall back to the default worktree-per-agent topology.

#### REQ: protected-branch-commit-confirmation

Because work lands directly on the checked-out branch, when that branch is protected (`main`/`master`/`release/*` per the parent's `protected-branch-promotion-guard`), `implement` MUST obtain explicit human confirmation before the run's first commit lands — even with the opt-in set. Standing config alone MUST NOT authorize committing onto a protected branch. This confirmation is scoped to protected branches only and does NOT weaken the parent's default (`gate-points`): commits onto a non-protected current branch remain autonomous.

### Execution Model

#### REQ: serial-by-default

In this topology agents MUST run serially — one task at a time on the shared checkout — by default, so concurrent agents cannot clobber the shared working tree or index.

#### REQ: plan-designated-parallel-only

Agents MAY run concurrently only for a set of tasks the Plan explicitly designates as parallel-safe (the Plan author asserting those tasks touch disjoint paths). Absent such an explicit designation, execution MUST remain serial.

### Safety Net

#### REQ: clean-tree-precondition

`implement` MUST refuse to start a current-branch run when the working tree is dirty (uncommitted or untracked changes that would entangle with the run), so the run does not mix with pre-existing user work.

#### REQ: pre-run-revert-ref

Before the run's first commit, `implement` MUST record the starting commit (HEAD) as a revert ref for the run and surface it to the user, so the entire run can be reverted as a unit despite the absence of worktree isolation.

### Transitions Under Collapse

#### REQ: commit-on-current-branch

T1 commit MUST place each task's changes directly on the current branch.

#### REQ: integrate-is-noop

Because work context and plan primary are the same branch, T2 integrate MUST be a no-op in this topology — there is no separate integration or merge step.

#### REQ: promote-is-local-noop

Because the plan primary and canonical target are the same branch, T3 promote MUST be a local no-op; the human gate the parent places at promotion instead applies at commit time onto a protected branch (see `protected-branch-commit-confirmation`). Any publication of the branch to a remote is governed by `change-publication-policy` and is out of scope for this topology.

## Dependencies

- implement-execution-topology

## Acceptance Criteria

### AC: roles-collapse-no-extra-branches (verifies REQ: roles-collapse-to-current-branch)

**Given** a current-branch run,
**When** it executes,
**Then** no plan primary branch and no per-agent worktree are created, and all work occurs on the checked-out branch.

### AC: opt-in-required-else-worktree (verifies REQ: explicit-opt-in-required)

**Given** no current-branch opt-in is set at any scope,
**When** `implement` starts,
**Then** it uses the worktree-per-agent topology rather than the current-branch topology.

### AC: protected-branch-confirmed-before-commit (verifies REQ: protected-branch-commit-confirmation)

**Given** the checked-out branch is `main` and the current-branch opt-in is set at project scope,
**When** the run is about to make its first commit,
**Then** `implement` first requires explicit human confirmation — standing config alone does not authorize it.

### AC: serial-by-default-execution (verifies REQ: serial-by-default)

**Given** a Plan with no parallel-safe designation and the current-branch topology,
**When** a batch of tasks runs,
**Then** the tasks execute one at a time on the shared checkout.

### AC: parallel-only-when-designated (verifies REQ: plan-designated-parallel-only)

**Given** a Plan that explicitly designates a set of tasks as parallel-safe,
**When** those tasks run in current-branch mode,
**Then** they MAY run concurrently; **and given** no such designation, **then** execution stays serial.

### AC: dirty-tree-refused (verifies REQ: clean-tree-precondition)

**Given** the working tree has uncommitted changes,
**When** a current-branch run is requested,
**Then** `implement` refuses to start and reports the dirty-tree precondition.

### AC: revert-ref-recorded (verifies REQ: pre-run-revert-ref)

**Given** a current-branch run on a clean tree,
**When** it is about to make its first commit,
**Then** `implement` records the starting HEAD as a revert ref and surfaces it to the user.

### AC: commits-land-on-current-branch (verifies REQ: commit-on-current-branch)

**Given** an agent completes a task,
**When** it commits,
**Then** the commit lands on the current branch.

### AC: integrate-noop (verifies REQ: integrate-is-noop)

**Given** a task's work is committed on the current branch,
**When** T2 integrate would run,
**Then** there is no separate merge or integration step.

### AC: promote-local-noop (verifies REQ: promote-is-local-noop)

**Given** the run's commits are on the current branch,
**When** T3 promote would run locally,
**Then** it is a no-op; any subsequent remote publication is governed by `change-publication-policy`.

## Rehearse Integration

Every AC here is testable against a git-fixture harness (initialize a temp repo, set/clear the opt-in, dirty/clean the tree, run the topology, and assert branch state, the recorded revert ref, serial-vs-parallel execution, and the protected-branch confirmation). Per the Rehearse heuristic these are testable, but stub scaffolding is deferred to the planning stage — the harness is uniform across these ACs and is better authored as one cohesive `_tests/` set when the plan defines it. This is an explicit, recorded deferral, not a "not testable" skip.

## Open Questions

- Approval cadence in this reduced-safety context (how much commit autonomy is permitted on a no-isolation, possibly-`main` branch) is owned by `approval-autonomy`, which MUST treat current-branch mode more strictly. This Feature deliberately does not define cadence.
- How "parallel-safe" task designation is expressed in the Plan (a per-task flag, a `Parallel-Safe:` group, etc.) — settle when the Plan schema is touched.
- Whether "dirty tree" should be configurable to allow an explicit user override (e.g., stash-and-restore) rather than a hard refusal — revisit at plan time.
- How the `pre-run-revert-ref` is surfaced (a printed commit SHA vs. a named ref such as `refs/specscore/revert/<run>`) — left open here so the plan can pick a concrete form for the `revert-ref-recorded` harness assertion.

---
*This document follows the https://specscore.md/feature-specification*
