# Idea: Implement Execution Topology & Branch Model

**Status:** Specified
**Date:** 2026-06-02
**Owner:** alexander.trakhimenok
**Promotes To:** implement-execution-topology, implement-execution-topology/current-branch, implement-execution-topology/worktree-per-agent
**Supersedes:** —
**Related Ideas:** —

## Problem Statement

How might we define a transport-agnostic execution topology for implement — branch roles and the gated transitions between them — so parallel (and eventually multi-machine) agents can integrate work onto a shared plan branch and promote it to a canonical target, with one clear human checkpoint, regardless of local-vs-remote transport?

## Context

implement today assumes a single shared index: all batch subagents stage into one working tree, the parent detects line-overlap conflicts and commits to the current branch — one branch, one push-to-remote. The approval-autonomy Idea (Draft) tried to define commit/push/approval gates but stalled because this underlying topology was never modeled. Two realizations forced the split: (a) 'push' conflates internal integration onto the plan's shared branch with external publication; (b) future multi-machine agents mean the rendezvous sync may itself be a remote push, so 'remote' is NOT a synonym for 'the human gate'. This Idea models the foundation; cadence/autonomy layers on top. Neighbors: specstudio-implement-skill (Implementing), approval-autonomy (Draft, depends on this), reviewer-gates, change-publication-policy / publication-policy.

## Recommended Direction

Model implement execution as transport-agnostic BRANCH ROLES and GATED TRANSITIONS, not concrete local/remote wiring. Branch roles: (1) work context — per agent, the tree/worktree/remote-machine where a task is committed; (2) plan primary branch — the rendezvous where agent work integrates, local for co-located agents and remote for multi-machine; (3) canonical target — the real destination (feature branch / main / PR). Transitions, each independently gateable and each local-or-remote under the hood: (1) commit work into the work context (autonomous); (2) integrate work onto the plan primary branch / rendezvous (autonomous plumbing; local merge or remote push by topology); (3) promote the plan primary branch to the canonical target (the single human gate — cumulative review). The safety boundary is transition 3 (promotion to canonical target), NOT any remote push — a rendezvous remote branch is cheap and disposable. By default each agent works in a DEDICATED WORKTREE (isolation on); working directly in the current branch — including main — is a supported topology but MUST be explicitly requested, precisely because it forgoes the worktree isolation that keeps autonomous runs revertable. MVP topology: single-machine, worktree-per-agent (default), local plan primary branch, promote to canonical target. Multi-machine (remote rendezvous transport) is designed-for in the vocabulary but not built.

## Alternatives Considered

- **Keep shared-index-only (status quo).** Simplest, zero new mechanism — but it cannot cleanly isolate parallel agents (it leans on line-overlap conflict detection), and it has no path to multi-machine. Rejected *as the abstraction*; it may survive as one supported simple topology, but it can't be the model.
- **Hardcode local/remote into the gate model ("push to remote = the human gate").** Rejected: it breaks the moment agents are distributed, because the rendezvous sync onto the plan primary branch is itself a remote push. Conflating transport with checkpoint semantics is exactly the mistake that stalled `approval-autonomy`.
- **Per-agent full branches integrated via PRs, even on one machine.** Heavyweight for MVP. PR-based promotion is a legitimate *option* for transition 3, not a mandated mechanism — so this becomes a config choice, not the baseline.

## MVP Scope

Define the branch-role + transition vocabulary as a shared contract, and have implement support ONE concrete topology end-to-end: single-machine worktree-per-agent, integrating onto a local plan primary branch (transition 2), promoting to a canonical target behind a single human gate (transition 3). Worktree-per-agent is the DEFAULT; a current-branch mode (agents work directly in the checkout, including main) is the explicit opt-in escape hatch. Transport-agnostic naming throughout so multi-machine slots in later with only a new transport, no renames. No multi-machine, no remote rendezvous, no distributed coordination in MVP.

## Not Doing (and Why)

- Multi-machine execution / remote-rendezvous transport — future-proofed in the vocabulary, deliberately not built in MVP
- Defining commit/approval/push cadence — that is the approval-autonomy Idea, layered on top of these transitions
- Mandating one transition-3 mechanism (merge vs rebase vs PR) — left as a per-topology/config choice
- Defaulting to the current branch / shared checkout — rejected as the default; working directly on the current branch (including main) is supported only as an explicit opt-in, precisely because it gives up the worktree isolation that makes autonomous runs safe
- Distributed-agent identity/auth/coordination protocol — far-future concern, out of scope

## Key Assumptions to Validate

| Tier | Assumption | How to validate |
|------|------------|-----------------|
| Must-be-true | The three branch roles (work context / plan primary / canonical target) cleanly cover BOTH today's shared-index run and a worktree-per-agent run | Map today's flow (shared index → current branch → remote) onto the roles; confirm no role is missing and nothing is forced |
| Must-be-true | Worktree-per-agent isolation is achievable with current tooling (Agent `isolation: worktree` / git worktrees) | Prototype a 2-agent batch in separate worktrees integrating onto a local plan primary branch |
| Should-be-true | Keeping transport (local/remote) orthogonal to checkpoint semantics means multi-machine later needs only a new transport — no rename of roles or transitions | Sketch the multi-machine variant against the same vocabulary and confirm the role/transition names still fit |
| Should-be-true | Promotion to the canonical target (transition 3) is the right SINGLE human gate, versus also gating integration (transition 2) | Reconcile against `approval-autonomy`'s cumulative-review decision; confirm gating transition 2 adds friction without safety |
| Might-be-true | Existing line-overlap conflict detection can be repurposed as the transition-2 integration check, or cleanly replaced by real git merge semantics | Revisit at specify; spike a merge-based integration and compare to the current post-batch line-overlap check |


## SpecScore Integration

- **New Features this would create:** a **parent `implement-execution-topology` Feature** defining the transport-agnostic contract — the three branch roles, the three gated transitions, gate semantics, and the current-branch opt-in mechanism — plus one **sub-Feature per topology scenario**: `worktree-per-agent` (single-machine, MVP default), `current-branch` (single-machine shared checkout incl. `main`, opt-in, reduced autonomy), and `multi-machine-rendezvous` (future, not specified in MVP). The parent owns the vocabulary; each sub-Feature specifies how that scenario realizes transitions 1–3.
- **Existing Features affected:** `specstudio-implement-skill` (batch dispatch, subagent staging, line-overlap conflict detection, commit/push — all reframed onto work-context / integrate / promote); `approval-autonomy` (depends on this — its cadence/gates are defined *over* these transitions); `reviewer-gates` & `change-publication-policy` / `publication-policy.md` (the canonical-target promotion gate must interplay with push branch-safety and the human gate); `specstudio-verify-skill` / `specstudio-recap-skill` (they consume commits that now live on the plan primary branch / canonical target).
- **Dependencies:** **blocks `approval-autonomy`** (autonomy cannot be specified until these transitions exist). Interacts with `publication-policy` push safety — canonical-target promotion should route through it. Future multi-machine would depend on distributed-agent identity/transport work that is explicitly out of scope here.

## Open Questions

- For the explicit current-branch opt-in: what is the precise opt-in mechanism (per-run flag, `specscore.yaml` setting, or both), and should opting onto `main`/`master` specifically demand a stronger confirmation than opting onto another current branch?
- For transition 2 on one machine: a real git merge/rebase onto the plan primary branch, or keep today's stage-into-shared-index aggregation *as* the integration? (The latter is far less of a rewrite.)
- Is the "plan primary branch" a new branch `implement` creates per plan, or just the branch the user launched from? (Interplay with the main/feature/worktree branch-shape axis.)
- For transition 3, which promotion mechanisms do we support — direct merge, push to a remote branch, open a PR — and is that per-project config?
- How does canonical target relate to the launch branch — is it always "the branch you started `implement` from", or independently configurable?

---
*This document follows the https://specscore.md/idea-specification*
