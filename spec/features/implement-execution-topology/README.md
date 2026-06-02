# Feature: Implement Execution Topology & Branch Model

> [SpecScore.**Studio**](https://specscore.studio): | [Explore](https://specscore.studio/app/github.com/specscore/specstudio-skills/spec/features/implement-execution-topology?op=explore) | [Edit](https://specscore.studio/app/github.com/specscore/specstudio-skills/spec/features/implement-execution-topology?op=edit) | [Ask question](https://specscore.studio/app/github.com/specscore/specstudio-skills/spec/features/implement-execution-topology?op=ask) | [Request change](https://specscore.studio/app/github.com/specscore/specstudio-skills/spec/features/implement-execution-topology?op=request-change) |

**Status:** Approved
**Date:** 2026-06-02
**Owner:** alexander.trakhimenok
**Source Ideas:** implement-execution-topology
**Supersedes:** —

## Summary

Transport-agnostic contract for how `implement` executes parallel agent work: the branch roles, the gated transitions between them, and how a topology is selected. Concrete per-scenario realization lives in sub-Features; approval cadence lives in the `approval-autonomy` layer.

## Problem

`implement` today hardwires a single shared index — every batch subagent stages into one working tree, the parent detects line-overlap conflicts and commits to the current branch, and there is exactly one push (to remote). That conflates internal integration of agent work with external publication, isolates nothing, and offers no shared vocabulary for *where* work lives, *how* parallel agents' work combines, and *where* a human must approve. As parallel — and eventually multi-machine — agents arrive, downstream concerns (approval autonomy, verify, recap, conflict handling) need stable terms to build on. This Feature establishes the transport-agnostic contract; concrete topologies and approval cadence layer on top.

## Behavior

### Branch Roles

`implement` work is described in terms of three **logical** branch roles. A role is a position in the flow, not a physical branch — a topology MAY map several roles onto one physical branch.

#### REQ: branch-roles

The contract MUST define three logical branch roles: **work context** (where an individual agent commits its task's changes), **plan primary branch** (the rendezvous onto which agents' committed work is integrated), and **canonical target** (the destination the plan's accumulated work is ultimately promoted to).

#### REQ: roles-are-logical-and-may-collapse

Branch roles MUST be logical, not physical. A topology MAY map two or more roles onto the same physical branch — for example, the current-branch topology collapses work context, plan primary branch, and canonical target onto the single checked-out branch.

#### REQ: transport-agnostic-roles

Role definitions MUST NOT presume local-versus-remote transport. Any role MAY be realized on the local machine or on a remote — e.g., the plan primary branch may be a local branch or a remote rendezvous branch — without changing the role's meaning or the contract.

### Transitions

Work advances through three transitions between roles. Each transition's transport is chosen by the active topology.

#### REQ: three-transitions

The contract MUST define three transitions: **T1 commit** (record an agent's work into its work context), **T2 integrate** (bring an agent's committed work onto the plan primary branch), and **T3 promote** (advance the plan primary branch's accumulated work to the canonical target).

#### REQ: transport-per-topology

The transport used to realize a transition (e.g., local merge/rebase versus remote push/fetch) MUST be determined by the active topology, not fixed by this contract. The same transition MAY be a local merge in one topology and a remote push in another.

#### REQ: gate-points

By default, T1 and T2 MUST be autonomous (no human gate), and T3 (promote) MUST be the single human-gated transition.

### Promotion Gate

#### REQ: promote-requires-human-approval

T3 (promote to canonical target) MUST require explicit human approval before it proceeds, regardless of transport (local merge or remote push). Human approval MAY be given at promotion time OR as an explicit prior pre-approval of that specific canonical target. A standing or auto-resolved config preference alone MUST NOT substitute for explicit human approval.

#### REQ: promote-routes-through-publication-policy

When promotion publishes to a remote, T3 MUST route through the `change-publication-policy` push branch-safety contract and MUST NOT bypass its branch deny rules (which deny `main`/`master`/`release/*` by default).

#### REQ: protected-branch-promotion-guard

When the canonical target is a publication-policy-protected branch (`main`/`master`/`release/*` by default), promotion MUST obtain explicit human confirmation for that promotion even when a topology opt-in is set, and even when the promotion is a purely local merge (which push branch-safety would not otherwise intercept). Only an explicit human pre-approval of that protected target may authorize autonomous promotion; standing config MUST NOT.

### Topology Selection

#### REQ: default-worktree-per-agent

Absent an explicit opt-in, `implement` MUST use the worktree-per-agent topology — each agent isolated in its own worktree. This default exists because isolation keeps autonomous work revertable.

#### REQ: current-branch-opt-in-is-persisted-preference

Running agents directly in the current branch/checkout (no per-agent worktree) MUST require an explicit opt-in expressed as a persisted, scoped preference. The opt-in MUST be resolvable at run, session, project, and user scopes — mirroring the publication-policy scope ladder — and MUST be persisted through the `specscore` CLI rather than by hand-editing config. A narrower scope overrides a broader one.

#### REQ: opt-in-remembered

Once the current-branch opt-in is set at a scope, `implement` MUST honor it for subsequent runs within that scope without re-prompting, subject to the `protected-branch-promotion-guard`.

## Dependencies

- change-publication-policy

## Acceptance Criteria

### AC: roles-named (verifies REQ: branch-roles)

**Given** any topology described under this contract,
**When** its branches are enumerated,
**Then** a work context, a plan primary branch, and a canonical target are each identified.

### AC: roles-collapse (verifies REQ: roles-are-logical-and-may-collapse)

**Given** the current-branch topology,
**When** roles are mapped to physical branches,
**Then** all three roles map to the single checked-out branch and the contract remains satisfied.

### AC: remote-plan-primary-allowed (verifies REQ: transport-agnostic-roles)

**Given** a topology where agents run on different machines,
**When** the plan primary branch is realized as a remote rendezvous branch,
**Then** the role definitions and the contract still apply unchanged.

### AC: transitions-enumerated (verifies REQ: three-transitions)

**Given** an `implement` run,
**When** its progress is described,
**Then** each unit of work passes through T1 commit, T2 integrate, and T3 promote.

### AC: transport-follows-topology (verifies REQ: transport-per-topology)

**Given** one topology declaring T2's transport as a remote push and another declaring it as a local merge,
**When** each performs T2 integrate,
**Then** each uses the transport its topology declares, with no change to the contract.

### AC: default-auto-promote-gated (verifies REQ: gate-points)

**Given** a default `implement` run,
**When** T1 and T2 occur,
**Then** no human approval is requested; **and when** T3 promote is reached, **then** human approval is requested.

### AC: promote-needs-approval (verifies REQ: promote-requires-human-approval)

**Given** a completed plan ready to promote and no explicit human approval or pre-approval,
**When** T3 is attempted,
**Then** promotion does not proceed and the human is asked to approve.

### AC: promote-respects-publication-policy (verifies REQ: promote-routes-through-publication-policy)

**Given** a canonical target on a publication-policy-denied branch and a promotion that publishes to a remote,
**When** T3 is attempted,
**Then** publication-policy push branch-safety refuses it and promotion does not occur.

### AC: protected-local-promotion-guarded (verifies REQ: protected-branch-promotion-guard)

**Given** canonical target `main` and a current-branch opt-in set at project scope,
**When** T3 would merge locally into `main`,
**Then** `implement` still requires explicit per-promotion human confirmation — standing config is insufficient — unless `main` was explicitly pre-approved.

### AC: default-is-worktree (verifies REQ: default-worktree-per-agent)

**Given** no topology opt-in is set at any scope,
**When** `implement` starts a plan,
**Then** each agent runs in its own worktree.

### AC: opt-in-persisted-and-remembered (verifies REQ: current-branch-opt-in-is-persisted-preference, REQ: opt-in-remembered)

**Given** the user sets the current-branch opt-in at session scope via the `specscore` CLI,
**When** a later run in the same session starts,
**Then** `implement` uses the current-branch topology without re-prompting, subject to the protected-branch guard.

## Open Questions

- Per-branch-mask/regex approval *levels* and the explicit pre-approval mechanism — should live as a shared branch-pattern approval policy consumed by the `approval-autonomy` layer rather than being defined here. Tracked for `approval-autonomy` specify.
- The exact `specscore` CLI surface for persisting the topology opt-in (a new verb versus reusing a publication-style group) — decided when the topology CLI / sub-Features are specified.
- Whether the plan primary branch is created per-plan or reuses an existing branch — concrete per sub-Feature (worktree-per-agent versus current-branch).
- How T2 integrate detects and handles conflicts (reuse today's line-overlap check versus real git merge semantics) — concrete per sub-Feature.

## Rehearse Integration

This Feature is a definitional contract — it fixes the vocabulary (branch roles), transition semantics, the promotion gate, and the topology opt-in mechanism. The concrete, executable behavior these ACs describe (worktree creation, integration transport, promotion gating, opt-in persistence) is realized by the per-scenario sub-Features (`worktree-per-agent`, `current-branch`), which own the Rehearse stubs for runtime behavior. No stubs are scaffolded at the parent level; per-AC test coverage is deferred to the sub-Features that implement each transition.

---
*This document follows the https://specscore.md/feature-specification*
