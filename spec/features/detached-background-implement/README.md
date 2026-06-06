---
format: https://specscore.md/feature-specification
status: Approved
---

# Feature: Detached Background Plan Implementation

> [SpecScore.**Studio**](https://specscore.studio): | [Explore](https://specscore.studio/app/github.com/specscore/specstudio-skills/spec/features/detached-background-implement?op=explore) | [Edit](https://specscore.studio/app/github.com/specscore/specstudio-skills/spec/features/detached-background-implement?op=edit) | [Ask question](https://specscore.studio/app/github.com/specscore/specstudio-skills/spec/features/detached-background-implement?op=ask) | [Request change](https://specscore.studio/app/github.com/specscore/specstudio-skills/spec/features/detached-background-implement?op=request-change) |
**Status:** Approved
**Date:** 2026-06-04
**Owner:** alexander.trakhimenok
**Source Ideas:** —
**Supersedes:** —
**Grade:** A

## Summary

Adds a third choice at the plan-approval checkpoint: besides approving a Plan and
optionally implementing it **in the current session**, the user may approve and
hand the Plan to a **detached background `claude` session** that runs `implement`
autonomously in its own git worktree. The user keeps working in the host session
and can attach to the background run later to steer it.

This Feature owns a new axis only — **where the whole implement run lives**
(in-session vs. a separate background process). It reuses, and does not
re-specify, the commit/push gating of `approval-autonomy` and the agent
worktree placement of `implement-execution-topology`.

## Problem

Today every implement run is in-session and synchronous: approving a Plan and
implementing it blocks the user's session until the run finishes or stops at a
blocker. There is no way to approve a Plan, send its execution off to run on its
own, and continue other work — then drop into that run when it needs a human.
This Feature provides that, without sacrificing host-session safety or the
ability to steer the background run.

## Behavior

### Plan-Approval Checkpoint

#### REQ: present-three-options

After the Plan reviewer gate releases and `plan.approved` is emitted, the plan
skill MUST present exactly three next-step options and wait for the user's
explicit choice: (1) **Approve** (flip status, stop); (2) **Approve + implement
in this session** (run `implement` in the current session); (3) **Approve +
implement in background** (launch a detached background run as defined below).

#### REQ: options-always-offered

In v1 all three options MUST always be offered. There is no configuration
surface that hides or reorders options.

### Detached Background Launch

#### REQ: launch-via-bg

When the user chooses option 3, the host MUST launch the background run with
`claude --bg` — a backgrounded interactive session that is listable via
`claude agents` and attachable via `claude attach`. It MUST NOT use
`claude -p` (headless one-shot, non-attachable) for this option.

#### REQ: scoped-permissions

The background run MUST be launched with `--permission-mode acceptEdits` and a
scoped `--allowedTools` allowlist. An action outside the allowlist MUST cause
the background session to pause for approval (it is steerable), and MUST NOT
abort the run. Because `--allowedTools` is variadic, the launch command MUST NOT
place the positional prompt immediately after it.

#### REQ: run-implement-in-background

The background session MUST run `implement` against the just-approved Plan,
operating inside the dedicated worktree created for the run.

### Host Insulation

#### REQ: precreate-worktree-in-host

Before launching, the host session MUST create the run's worktree with
`git worktree add`. Creating a worktree MUST NOT change the host session's
current branch, index, or working-tree files.

#### REQ: branch-derivation-and-override

The background run's target branch MUST default to `feat/<plan-slug>`. When the
user chooses option 3, they MAY override this with a branch name of their choice.

#### REQ: branch-distinct-from-host

The target branch MUST differ from the host session's current branch. If the
requested branch equals the host's current branch, or is already checked out in
another worktree, the host MUST refuse to launch and report the conflict instead
of creating the worktree.

#### REQ: no-cross-branch-mutation

The background run MUST operate only inside its own worktree. It MUST NOT
`git checkout`/`git switch` to a different branch and MUST NOT modify any branch
other than its own target branch.

### Autonomous Progress Contract

This per-task defer-and-continue behavior is distinct from `approval-autonomy`'s
run-level anomaly-halts. A task the background run cannot complete is **deferred**
under this contract (treated as blocked, skipped, run continues) — it is not a
run-level halt; this overrides treating an unfinishable task as a run-halting
condition in background mode. The integrity anomaly-halts that are not
deferrable tasks — sibling integration conflict, unresolved lint, and
source-Feature drift — still halt the whole background run.

#### REQ: maximize-progress-defer-blockers

The background run MUST complete every task it can. A task that is blocked
(needs a human decision, missing information, an unresolved test failure, or a
permission it lacks) MUST be skipped rather than aborting the run. Only the
blocked task's own dependents are blocked; independent tasks MUST continue.

#### REQ: approval-actions-last

The background run MUST schedule any action that requires human approval after
all independently-completable work, so a pause for approval does not stall work
that could otherwise proceed.

#### REQ: pause-when-only-blockers-remain

When only blocked tasks remain, the background run MUST pause and wait for input.
It MUST NOT abort, and MUST NOT improvise around a decision that requires a
human.

### Steerability

#### REQ: steer-controls

The user MUST be able to list background runs (`claude agents`), attach to steer
one (`claude attach <id>`), watch its output (`claude logs <id>`), and stop it
(`claude stop <id>`). Multiple background runs MAY exist concurrently and the
user MAY switch between them.

### Blocker Reporting (v1)

#### REQ: blockers-via-live-session-only

In v1 the background run MUST surface blockers only through its live (paused)
session, resolved by attaching. It MUST NOT be required to produce a `BLOCKED.md`
or any other durable blocker artifact.

## Acceptance Criteria

### AC: three-options-presented (verifies REQ:present-three-options, REQ:options-always-offered)

**Given** a Plan whose reviewer gate has released and `plan.approved` has been emitted
**When** the plan skill reaches the post-approval transition
**Then** the user is presented with exactly three options — approve, approve + implement in session, approve + implement in background — and no option is hidden by configuration.

### AC: launches-bg-session (verifies REQ:launch-via-bg, REQ:run-implement-in-background)

**Given** the user chose "Approve + implement in background"
**When** the host launches the run
**Then** a detached session is started with `claude --bg` (appearing in `claude agents`) that runs `implement` against the approved Plan, and `claude -p` is not used.

### AC: permissions-pause-not-fail (verifies REQ:scoped-permissions)

**Given** a background run launched with `--permission-mode acceptEdits` and a scoped `--allowedTools`
**When** the run needs an action outside its allowlist
**Then** the session pauses for approval rather than failing or aborting the run.

### AC: worktree-precreated-host-unchanged (verifies REQ:precreate-worktree-in-host)

**Given** a host session on branch `X` choosing option 3
**When** the host creates the run's worktree with `git worktree add`
**Then** the host session remains on branch `X` with its working tree and index unchanged.

### AC: branch-defaults-and-overridable (verifies REQ:branch-derivation-and-override)

**Given** a Plan with slug `<plan-slug>`
**When** the user chooses option 3 without specifying a branch
**Then** the target branch is `feat/<plan-slug>`; and when the user specifies a branch, that name is used instead.

### AC: refuse-same-or-checked-out-branch (verifies REQ:branch-distinct-from-host)

**Given** the requested target branch equals the host's current branch or is already checked out in another worktree
**When** the host attempts to launch
**Then** the host refuses to create the worktree and reports the conflict.

### AC: background-stays-in-worktree (verifies REQ:no-cross-branch-mutation)

**Given** a running background implement run
**When** it performs git operations
**Then** all changes occur inside its own worktree on its target branch, and no other branch is checked out or modified.

### AC: continues-past-a-blocker (verifies REQ:maximize-progress-defer-blockers)

**Given** a Plan with one blocked task and other independent unblocked tasks
**When** the background run reaches the blocked task
**Then** it skips that task (and only its dependents), continues the independent tasks, and does not abort.

### AC: approval-work-deferred-last (verifies REQ:approval-actions-last)

**Given** a Plan mixing tasks that need approval with tasks that do not
**When** the background run executes
**Then** the approval-requiring actions are attempted only after the independently-completable work is done.

### AC: pause-on-remaining-blockers (verifies REQ:pause-when-only-blockers-remain)

**Given** a background run whose only remaining tasks are blocked
**When** it has finished all work it can
**Then** it pauses and waits for input instead of aborting or improvising a human decision.

### AC: steer-controls-available (verifies REQ:steer-controls)

**Given** one or more background runs in progress
**When** the user runs `claude agents`
**Then** each run is listed and can be attached (`claude attach <id>`), watched (`claude logs <id>`), and stopped (`claude stop <id>`).

### AC: no-blocked-artifact-required (verifies REQ:blockers-via-live-session-only)

**Given** a background run that has paused on a blocker in v1
**When** the user inspects the run
**Then** the blocker is available through the live paused session on attach, and no `BLOCKED.md` artifact is required to exist.

## Out of Scope

- Commit and push **gating** (cadence, pre_commit/pre_push gates, anomaly-halts, re-arm) — owned by `approval-autonomy`.
- Agent worktree **placement and integration topology** (worktree-per-agent, serial merge, promote-to-launch) — owned by `implement-execution-topology`.
- Any durable `BLOCKED.md` blocker artifact and its format — deferred beyond v1.
- Multi-machine or remote execution; scheduled or delayed launch.
- `claude -p` headless one-shot execution mode.
- Surviving a full quit of all Claude clients: `--bg` runs are daemon-hosted and end when the transient supervisor daemon exits with no clients. This is accepted for the steer-while-working use case and is not a guarantee this Feature makes.

## Rehearse Integration

No Rehearse stubs are scaffolded in v1. Every AC describes skill-orchestration
behavior of the `plan` and `implement` skills (presenting options, launching a
detached `claude --bg` process, git-worktree side effects, autonomous run
sequencing) whose observable surface is the live CLI and a running background
session — not a unit-testable CLI/HTTP/pure-function boundary this repo can
stub deterministically. These ACs are verified through the skills' own behavior
during use; stubs may be added if a deterministic harness for `claude --bg`
launches becomes available.

## Open Questions

- The concrete contents of the scoped `--allowedTools` allowlist for a background
  run are not pinned by this Feature; they are decided at plan/launch time. A
  later revision may standardize a default allowlist.

---
*This document follows the https://specscore.md/feature-specification*
