# Plan: Approval Autonomy (Implement)

**Status:** Approved
**Source Feature:** approval-autonomy
**Date:** 2026-06-03
**Owner:** alexander.trakhimenok
**Supersedes:** —

## Summary

Decomposes the [approval-autonomy Feature](../features/approval-autonomy/README.md) into eight linear tasks: the two mandated affected-feature revisions first (the `reviewer-gates` `when:` condition and the `current-branch` Option-B reconciliation), then the core `implement` gate-point wiring, then the non-reviewer execution dynamics (cadence, anomaly-halts, re-arm, cumulative review) and the push-safety floor. All 13 source-Feature ACs are covered by tasks; none deferred.

## Approach

Order is linear and dependency-respecting. Tasks 1–2 are in-place revisions to *other* Features that this Feature mandates, sequenced first because the core wiring leans on them (Task 1 supplies the `when:` field used by branch masks; Task 2 removes the commit-time gate the autonomy model assumes gone). Task 3 wires `implement` to fire the gate-point events; Tasks 4–8 layer the execution dynamics and safety floor on that wiring. **Cross-plan dependency:** the gate-point events (`implementation.pre_commit`/`pre_push`), the `noop`/`deterministic` reviewer types, and multi-fire are delivered by the `reviewer-gates` Plan's pending **Task 9** (event-keyed implementation) — that task is a runtime prerequisite for Tasks 3–8 here, even though it lives in the `reviewer-gates` Plan. Tasks 1–2 are themselves edits to already-approved Features (revise-in-place), tracked as tasks here because this Feature's REQs mandate them.

## Tasks

### Task 1: Add the `when:` branch-pattern gate-entry condition to reviewer-gates

**Verifies:** approval-autonomy#ac:branch-mask-as-gate-when

Revise the `reviewer-gates` Feature and its loader/runner in place to add an optional `when: <branch-pattern>` field on a gate reviewer entry: when present, the entry participates in the gate only if the current branch matches; when absent, it always participates. Pin the match grammar (regex vs glob) in `reviewer-gates` so both layers agree. This is the home for per-branch autonomy masks.

### Task 2: Reconcile current-branch protected-branch confirmation to gate-at-promote (Option B)

**Verifies:** approval-autonomy#ac:current-branch-reconciled

Revise `implement-execution-topology/current-branch` in place so REQ `protected-branch-commit-confirmation` no longer requires human confirmation before committing onto a protected branch; the protected-branch checkpoint moves to promote/push (`implementation.pre_push` + publication-policy push-safety). Preserve the clean-tree precondition and pre-run revert ref.

### Task 3: Fire the gate-point events from implement (no hardcoded approval)

**Verifies:** approval-autonomy#ac:fires-pre-commit-and-pre-push, approval-autonomy#ac:no-hardcoded-approval

Wire `implement` to fire `implementation.pre_commit` before each commit and `implementation.pre_push` before promote, proceeding only when the gate releases. Remove any hardcoded per-batch user-approval step: the per-batch human checkpoint exists iff a `type: human` reviewer is configured on `pre_commit`. Depends on the gate-point events existing (reviewer-gates Task 9).

### Task 4: Commit cadence + autonomy config namespace

**Verifies:** approval-autonomy#ac:cadence-default-and-config, approval-autonomy#ac:pre-commit-fires-per-commit, approval-autonomy#ac:config-under-autonomy-namespace

Resolve `autonomy.implement.commit_cadence` (`task`/`batch`/`plan`, default `batch`) across the publication-policy scope ladder, under a top-level `autonomy:` namespace (no workflow-step names at top level). Commit at the resolved boundary and fire `pre_commit` once per commit produced (multi-fire).

### Task 5: Anomaly-halts

**Verifies:** approval-autonomy#ac:halts-on-each-anomaly, approval-autonomy#ac:halt-surfaces-cause-no-autoresolve

Halt the run on any of sibling merge conflict, BLOCKED subagent, lint failure unresolved after one `--fix`, or source-Feature drift — regardless of gate config — surfacing the specific cause and performing no auto-resolution.

### Task 6: Explicit re-arm after a halt

**Verifies:** approval-autonomy#ac:re-arm-required, approval-autonomy#ac:re-arm-scoped-to-run

Require an explicit re-arm signal before resuming autonomous execution after an anomaly-halt (no auto-resume); scope the re-armed state to the current run only.

### Task 7: Cumulative review at the push gate

**Verifies:** approval-autonomy#ac:cumulative-review-presented

When `implement` fires `implementation.pre_push` and that gate has a `type: human` reviewer, present the cumulative set of commits accumulated during the run (commits + consolidated diff, or a per-commit summary when very large) as the reviewer's context.

### Task 8: Preserve push branch-safety

**Verifies:** approval-autonomy#ac:push-still-branch-safe

Route every promote/push through `change-publication-policy` push branch-safety (deny `main`/`master`/`release/*` by default); autonomy never bypasses it. A `deterministic` branch-safety reviewer on `pre_push` complements, not replaces, the publication-policy check.

## Plan-Time Decisions (pinned)

The source Feature punted four decisions to plan time; pinned here:

- **Re-arm signal (Task 6):** `continue` — a lowercase standalone token, matching the approval-phrase recognizer style.
- **`commit_cadence: plan` guardrail (Task 4):** allowed, but `implement` MUST warn that it defers all commits to run-end (weakening per-batch revert granularity); no extra mechanism in MVP.
- **Cumulative-review threshold (Task 7):** present a consolidated diff up to ~150 changed lines or ~10 files; beyond either bound, switch to a per-commit summary.
- **`when:` match grammar (Task 1):** anchored **regex** (e.g. `^(main|master|release/)`), pinned in the `reviewer-gates` `when:` addition so this layer and the gates layer share one dialect.

## Open Questions

- Task sequencing assumes the `reviewer-gates` Plan's Task 9 lands first (it delivers the gate-point events and `noop`/`deterministic` types that Tasks 3–8 consume at runtime). If Task 9 slips, Tasks 3–8 here are blocked — track that cross-plan ordering at implement time.

---
*This document follows the https://specscore.md/plan-specification*
