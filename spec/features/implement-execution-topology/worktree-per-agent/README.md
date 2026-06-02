# Feature: Worktree-Per-Agent Topology (Single Machine)

> [SpecScore.**Studio**](https://specscore.studio): | [Explore](https://specscore.studio/app/github.com/specscore/specstudio-skills/spec/features/implement-execution-topology/worktree-per-agent?op=explore) | [Edit](https://specscore.studio/app/github.com/specscore/specstudio-skills/spec/features/implement-execution-topology/worktree-per-agent?op=edit) | [Ask question](https://specscore.studio/app/github.com/specscore/specstudio-skills/spec/features/implement-execution-topology/worktree-per-agent?op=ask) | [Request change](https://specscore.studio/app/github.com/specscore/specstudio-skills/spec/features/implement-execution-topology/worktree-per-agent?op=request-change) |

**Status:** Approved
**Date:** 2026-06-02
**Owner:** alexander.trakhimenok
**Source Ideas:** implement-execution-topology
**Supersedes:** —

## Summary

Single-machine, worktree-per-agent realization of the `implement-execution-topology` contract: each agent commits in its own git worktree (T1), branches integrate onto a dedicated per-plan plan-primary branch via real git merge (T2), and the plan primary is promoted to the launch branch behind the human gate (T3). This is the contract's default topology.

## Problem

The parent contract defines roles and transitions abstractly but leaves the default topology unrealized. `implement` today shares one index across all batch agents — no isolation, and a line-overlap heuristic standing in for conflict detection. This sub-Feature makes the default concrete: isolate each agent in its own worktree so parallel work cannot corrupt a shared index, integrate agent branches via real git merges onto a dedicated plan branch (so conflict detection is git's own 3-way merge), and gate promotion of that plan branch onto the launch branch. It inherits the parent's vocabulary and gate semantics and MUST NOT redefine them.

## Behavior

### Work Contexts (T1)

Each agent's work context is an isolated git worktree on its own branch.

#### REQ: agent-worktree-isolation

Each agent MUST run in its own dedicated git worktree on its own branch. Agents in the same batch MUST NOT share a working tree or index.

#### REQ: worktree-base-ref

Each agent's worktree branch MUST be created from the current tip of the plan primary branch at the start of that agent's batch, so a later batch builds on the integrated work of earlier batches.

#### REQ: agent-commits-in-worktree

Each agent MUST commit its task's changes on its own worktree branch (T1). Agents MUST NOT commit to, merge into, or push the plan primary branch or the canonical target directly.

### Plan Primary Branch

The plan primary branch is the run's integration rendezvous, distinct from the canonical target.

#### REQ: dedicated-plan-primary

`implement` MUST create a dedicated plan primary branch for the run, branched from the canonical target (the launch branch) at run start. The plan primary branch MUST be distinct from the canonical target.

### Integration (T2)

#### REQ: integrate-via-git-merge

T2 MUST integrate each agent's worktree branch onto the plan primary branch with a real git merge. Conflict detection MUST use git's native 3-way merge — not a line-overlap heuristic.

#### REQ: integrate-serially

Within a batch, agent branches MUST be merged onto the plan primary branch one at a time (serially), so any merge conflict is attributable to a specific agent branch.

#### REQ: integration-conflict-halt

On a git merge conflict during T2, `implement` MUST halt integration of the conflicting branch, surface the conflicting branch and paths, and MUST NOT auto-resolve. The run MUST NOT advance past an unresolved integration conflict.

### Promotion (T3)

#### REQ: promote-merge-to-launch

T3 MUST promote by merging the plan primary branch into the canonical target (the launch branch the run started from), behind the parent contract's human gate. The binding of canonical target to the launch branch is specific to this single-machine topology; another topology MAY rebind it without contradicting this Feature. Any subsequent publication of the launch branch to a remote is governed by `change-publication-policy` and is out of scope for this local merge.

#### REQ: pre-approved-target-skips-reprompt

When the human has explicitly pre-approved the canonical target for this run (pre-approval mechanism owned by `approval-autonomy`), T3 MUST promote without an additional approval prompt. Absent such pre-approval, T3 MUST obtain explicit approval before merging into the canonical target. This skip is subject to the parent's `protected-branch-promotion-guard`: when the canonical target is a protected branch (`main`/`master`/`release/*`), the stricter parent rule governs and an unconditional skip MUST NOT be implemented.

### Worktree Lifecycle

#### REQ: worktree-cleanup-on-success

After an agent's branch is successfully integrated (T2), `implement` MUST remove that agent's worktree. The integrated commits persist on the plan primary branch.

#### REQ: worktree-retain-on-failure

If an agent ends BLOCKED or its branch fails to integrate (merge conflict), `implement` MUST retain that agent's worktree and branch for inspection and MUST NOT delete unintegrated work.

## Dependencies

- implement-execution-topology

## Acceptance Criteria

### AC: agents-isolated (verifies REQ: agent-worktree-isolation)

**Given** a batch of two agents,
**When** they run,
**Then** each occupies a separate git worktree on its own branch and neither sees the other's uncommitted files.

### AC: batch-builds-on-prior (verifies REQ: worktree-base-ref)

**Given** batch 1 has been integrated onto the plan primary branch,
**When** a batch-2 agent's worktree branch is created,
**Then** it starts from the plan primary tip and contains batch 1's integrated commits.

### AC: commits-land-on-worktree-branch (verifies REQ: agent-commits-in-worktree)

**Given** an agent completes its task,
**When** it commits,
**Then** the commit is on its own worktree branch and neither the plan primary branch nor the canonical target has moved.

### AC: plan-primary-created-distinct (verifies REQ: dedicated-plan-primary)

**Given** a run starts from a launch branch,
**When** `implement` initializes,
**Then** a dedicated plan primary branch exists, branched from the launch branch and distinct from it.

### AC: integration-is-git-merge (verifies REQ: integrate-via-git-merge)

**Given** an agent branch with changes that conflict at the git level with the plan primary,
**When** T2 integrates it,
**Then** integration is performed via git merge and git's 3-way merge reports the conflict.

### AC: serial-merge-attributes-conflict (verifies REQ: integrate-serially)

**Given** two agent branches that both modify the same lines,
**When** they are integrated,
**Then** they merge one at a time and the conflict is reported against the second branch merged.

### AC: conflict-halts-run (verifies REQ: integration-conflict-halt)

**Given** a T2 merge produces a conflict,
**When** `implement` handles it,
**Then** it halts that branch's integration, surfaces the branch and conflicting paths, performs no auto-resolution, and does not advance.

### AC: promote-merges-into-launch (verifies REQ: promote-merge-to-launch)

**Given** an integrated plan primary branch and a released human gate,
**When** T3 runs,
**Then** the plan primary branch is merged into the launch branch.

### AC: pre-approved-promotes-without-prompt (verifies REQ: pre-approved-target-skips-reprompt)

**Given** the canonical target has been explicitly pre-approved for the run,
**When** T3 is reached,
**Then** promotion proceeds without an additional approval prompt; **and given** no pre-approval exists, **then** T3 first requests explicit approval.

### AC: worktree-removed-after-integration (verifies REQ: worktree-cleanup-on-success)

**Given** an agent's branch has been successfully integrated,
**When** cleanup runs,
**Then** that agent's worktree is removed and its commits remain on the plan primary branch.

### AC: failed-worktree-retained (verifies REQ: worktree-retain-on-failure)

**Given** an agent ends BLOCKED or its branch hits a merge conflict,
**When** the batch finishes,
**Then** that agent's worktree and branch are retained for inspection.

## Rehearse Integration

Every AC here is testable against a git-fixture harness (initialize a temporary repo, drive the topology operations — worktree create, commit, serial merge, promote, cleanup — and assert branch/worktree/merge state). Per the Rehearse heuristic these are testable, but stub scaffolding is deferred to the planning stage rather than created at spec time: the harness is uniform across all eleven ACs, so the stubs are better authored as one cohesive `_tests/` set when the plan defines the harness. This is an explicit, recorded deferral — not a "not testable" skip.

## Open Questions

- Should the plan primary branch be deleted after a successful T3 promote, or retained as an audit trail of the run's integration history?
- Naming scheme for the per-plan plan primary branch and per-agent worktree branches (e.g., derived from the Plan slug + task number) — settle at plan time.
- On `integration-conflict-halt`, what resume affordance does the user get (manual resolve in the retained worktree, then re-integrate) — concrete at plan time, and interacts with the `approval-autonomy` re-arm semantics.

---
*This document follows the https://specscore.md/feature-specification*
