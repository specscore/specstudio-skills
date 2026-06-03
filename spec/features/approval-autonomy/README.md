# Feature: Approval Autonomy (Implement)

> [SpecScore.**Studio**](https://specscore.studio): | [Explore](https://specscore.studio/app/github.com/specscore/specstudio-skills/spec/features/approval-autonomy?op=explore) | [Edit](https://specscore.studio/app/github.com/specscore/specstudio-skills/spec/features/approval-autonomy?op=edit) | [Ask question](https://specscore.studio/app/github.com/specscore/specstudio-skills/spec/features/approval-autonomy?op=ask) | [Request change](https://specscore.studio/app/github.com/specscore/specstudio-skills/spec/features/approval-autonomy?op=request-change) |

**Status:** Approved
**Date:** 2026-06-02
**Owner:** alexander.trakhimenok
**Source Ideas:** approval-autonomy
**Supersedes:** —
**Grade:** A

## Summary

The implement-autonomy layer. `implement` fires the event-keyed gate-point events `implementation.pre_commit` and `implementation.pre_push` at its checkpoints, so "commit-autonomous / push-gated" is expressed purely as `reviewer-gates` config (no human on `pre_commit` → autonomous commits; a `human` on `pre_push` → the single push gate). This Feature owns only the parts that are NOT reviewer decisions: per-batch commit cadence, the anomaly-halts, explicit re-arm, the cumulative review fed to the push gate, and the config namespace. Scoped to `implement` (single machine, both topologies).

## Problem

`implement` today hard-gates per-batch user approval, which blocks prolonged unattended runs — the user's actual goal. The event-keyed [`reviewer-gates`](../reviewer-gates/README.md) revision now supplies the gate-point events (`implementation.pre_commit`/`pre_push`, multi-fire) and the `auto-approve`/`deterministic` reviewer types that *could* express autonomy, but nothing fires those events from `implement`, and the execution dynamics that are not reviewer decisions (cadence, sibling-conflict/BLOCKED halts, re-arm, cumulative review) have no home. This Feature wires `implement` to the gates and owns those dynamics, enabling bounded autonomy — commit freely, gate at push, halt loudly on anomalies — without weakening revert-safety (commits are local and revertable; push is independently branch-safe via [`change-publication-policy`](../change-publication-policy/README.md)).

## Behavior

### Firing the gate-point events

`implement` consults `reviewer-gates` at its checkpoints by firing events; it owns no approval mechanism of its own.

#### REQ: fire-gate-point-events

`implement` MUST fire `implementation.pre_commit` before each commit it makes during a run, and `implementation.pre_push` before promoting/pushing (transition T3). Each action MUST proceed only if its gate releases per [`reviewer-gates`](../reviewer-gates/README.md); on `Issues Found` the action is blocked and the findings surfaced.

#### REQ: autonomy-is-gate-config

"Commit-autonomous" and "push-gated" MUST be expressed solely through the reviewers configured on `gates.implementation.pre_commit` / `gates.implementation.pre_push` (a `auto-approve`/`deterministic`-only gate is autonomous; a gate with a `type: human` reviewer stops for a human). `implement` MUST NOT carry a separate hardcoded per-batch user-approval step — the per-batch human checkpoint exists if and only if a `type: human` reviewer is configured on `implementation.pre_commit`.

### Commit cadence

#### REQ: commit-cadence-config

The commit cadence MUST be resolvable from `autonomy.implement.commit_cadence` with values `task`, `batch`, or `plan`, defaulting to `batch` when unset. It selects the boundary at which `implement` commits: `task` → one commit per task, `batch` → one commit per integrated batch, `plan` → one commit at run end. The value MUST be resolvable across the publication-policy scope ladder (run / session / project / user), narrower overriding broader.

#### REQ: pre-commit-fires-per-commit

`implementation.pre_commit` MUST fire once per commit the resolved cadence produces — per task, per batch, or once for the plan — consistent with `reviewer-gates`' multi-fire semantics. Each firing is an independent gate evaluation.

### Anomaly halts

#### REQ: anomaly-halt-set

Regardless of gate configuration — including a fully autonomous (`auto-approve`) `pre_commit` gate — `implement` MUST halt the run on any of these **execution-state anomalies**: (a) a sibling integration/merge conflict, (b) a BLOCKED subagent, (c) a lint failure that `specscore spec lint --fix` did not resolve in one pass, (d) source-Feature drift (the Plan's source Feature regressed below `Approved`). These are distinct from gate verdicts: a reviewer does not "approve" a conflict or a BLOCKED task.

#### REQ: halt-no-auto-resolve

On an anomaly-halt, `implement` MUST stop, surface the specific cause (conflicting branch and paths; the BLOCKED task and its reason; the unresolved lint violations; or the drifted Feature and its status), and MUST NOT auto-resolve the anomaly or silently continue.

### Re-arm

#### REQ: explicit-re-arm

After the user addresses an anomaly-halt, `implement` MUST require an explicit re-arm signal (e.g., the user typing `continue`) before resuming autonomous execution. It MUST NOT auto-resume when the anomaly clears.

#### REQ: re-arm-scope

A re-arm re-enables autonomy for the remainder of the **current `implement` run** only; a subsequent run starts from the resolved config, not from a prior run's re-armed state.

### Cumulative review at the push gate

#### REQ: cumulative-review-at-pre-push

When `implement` fires `implementation.pre_push` and that gate includes a `type: human` reviewer, `implement` MUST present a **cumulative review** as the context for that reviewer — the full set of commits accumulated during the run (commits plus their consolidated diff, or a per-commit summary when the diff is very large) — not merely the final commit. This is the single human checkpoint of an autonomous run.

### Safety floor

#### REQ: push-safety-preserved

Promote/push MUST still route through [`change-publication-policy`](../change-publication-policy/README.md) push branch-safety (which denies `main`/`master`/`release/*` by default); approval-autonomy MUST NOT weaken or bypass it. A `deterministic` branch-safety reviewer on `pre_push` and publication-policy's own check are complementary, not a substitute for it.

### Config namespace

#### REQ: autonomy-namespace

`implement` autonomy execution config MUST live under a top-level `autonomy:` key in `specscore.yaml`, keyed by skill name (MVP: `autonomy.implement`). Workflow-step names MUST NOT appear as top-level config keys. `gates:` (who approves each event) and `autonomy:` (execution knobs) are distinct top-level concerns.

### Per-branch autonomy masks

#### REQ: branch-mask-via-gate-when

Per-branch autonomy preferences (e.g., "also require a human on `pre_commit` when on `main`") MUST be expressed as an optional `when: <branch-pattern>` condition on a `reviewer-gates` gate entry, NOT as autonomy-local config — keeping branch-matching in the gates layer and out of this Feature. Absent any `when:` mask, autonomy is branch-agnostic (per the Idea's Option-B decision: commits are local and revertable, so the only hard human gate is at push).

### Affected-feature revisions

#### REQ: reviewer-gates-when-condition

The [`reviewer-gates`](../reviewer-gates/README.md) Feature MUST be revised in place to add an optional `when: <branch-pattern>` field to a gate reviewer entry: when present, the entry participates in the gate only if the current branch matches the pattern; when absent, the entry always participates. This is the home for `branch-mask-via-gate-when`.

#### REQ: current-branch-reconciliation

The [`implement-execution-topology/current-branch`](../implement-execution-topology/current-branch/README.md) Feature's REQ `protected-branch-commit-confirmation` MUST be revised in place to align with Option B: a current-branch run MUST NOT require human confirmation before committing onto a protected branch at commit time. The protected-branch human checkpoint moves to the promote/push gate (`implementation.pre_push` + publication-policy push-safety). Commits remain local and revertable; the topology's clean-tree precondition and pre-run revert ref still apply. Note: in the current-branch topology promote is a local no-op (per that Feature's `promote-is-local-auto-approve`), so the relocated checkpoint materializes only on an actual remote push — a local-only run never publishes and is therefore correctly ungated at commit.

## Dependencies

- implement-execution-topology
- reviewer-gates

## Acceptance Criteria

### AC: fires-pre-commit-and-pre-push (verifies REQ: fire-gate-point-events)

**Given** an `implement` run with `gates.implementation.pre_commit` and `gates.implementation.pre_push` configured,
**When** the run makes a commit and later reaches promote,
**Then** it fires `implementation.pre_commit` before the commit and `implementation.pre_push` before the promote, and each action proceeds only when its gate releases (and is blocked, with findings surfaced, on `Issues Found`).

### AC: no-hardcoded-approval (verifies REQ: autonomy-is-gate-config)

**Given** `gates.implementation.pre_commit` configured with only a `auto-approve` reviewer,
**When** a batch is ready to commit,
**Then** `implement` commits without prompting any human (no hardcoded per-batch approval step runs); **and given** the gate instead includes a `type: human` reviewer, **then** `implement` stops for that human before committing.

### AC: cadence-default-and-config (verifies REQ: commit-cadence-config)

**Given** no `autonomy.implement.commit_cadence` set anywhere,
**When** `implement` runs,
**Then** it commits once per integrated batch (default `batch`); **and given** `autonomy.implement.commit_cadence: task` at project scope with no narrower override, **then** it commits once per task.

### AC: pre-commit-fires-per-commit (verifies REQ: pre-commit-fires-per-commit)

**Given** cadence `batch` and a run that produces three batch commits,
**When** the run executes,
**Then** `implementation.pre_commit` is fired three times — once before each commit — each an independent gate evaluation.

### AC: halts-on-each-anomaly (verifies REQ: anomaly-halt-set)

**Given** a fully autonomous `pre_commit` gate (`auto-approve`),
**When** the run encounters any of a sibling merge conflict, a BLOCKED subagent, an unresolved-after-`--fix` lint failure, or source-Feature drift,
**Then** the run halts despite the autonomous gate — the anomaly is not subject to the gate verdict.

### AC: halt-surfaces-cause-no-autoresolve (verifies REQ: halt-no-auto-resolve)

**Given** an anomaly-halt has fired,
**When** `implement` reports it,
**Then** it names the specific cause (conflicting branch/paths, BLOCKED task + reason, lint violations, or the drifted Feature + status), performs no auto-resolution, and does not advance.

### AC: re-arm-required (verifies REQ: explicit-re-arm)

**Given** the user has fixed the cause of an anomaly-halt,
**When** the anomaly condition clears,
**Then** `implement` does not auto-resume; it resumes autonomous execution only after the user issues an explicit re-arm signal.

### AC: re-arm-scoped-to-run (verifies REQ: re-arm-scope)

**Given** a run was re-armed after a halt and then completed,
**When** a new `implement` run starts,
**Then** the new run derives autonomy from the resolved config, not from the prior run's re-armed state.

### AC: cumulative-review-presented (verifies REQ: cumulative-review-at-pre-push)

**Given** a multi-batch run whose `implementation.pre_push` gate includes a `type: human` reviewer,
**When** the push gate is reached,
**Then** the human reviewer is shown the cumulative set of commits accumulated during the run (commits + consolidated diff, or a per-commit summary if very large), not just the final commit.

### AC: push-still-branch-safe (verifies REQ: push-safety-preserved)

**Given** a run targeting a publication-policy-denied branch for push,
**When** promote/push is attempted,
**Then** publication-policy push branch-safety refuses it regardless of autonomy settings, and approval-autonomy does not bypass it.

### AC: config-under-autonomy-namespace (verifies REQ: autonomy-namespace)

**Given** `specscore.yaml`,
**When** implement autonomy config is declared,
**Then** it appears under a top-level `autonomy:` key (e.g., `autonomy.implement.commit_cadence`), and no workflow-step name (e.g., `implement:`) appears as a top-level config key.

### AC: branch-mask-as-gate-when (verifies REQ: branch-mask-via-gate-when, REQ: reviewer-gates-when-condition)

**Given** a `type: human` entry on `implementation.pre_commit` carrying `when: "branch =~ ^(main|master|release/)"`,
**When** the run is on a feature branch and separately on `main`,
**Then** on the feature branch the human entry does not participate (commits stay autonomous) and on `main` it participates (the human is asked before commit) — the per-branch behavior coming entirely from the gate-entry `when:` condition.

### AC: current-branch-reconciled (verifies REQ: current-branch-reconciliation)

**Given** this Feature is approved and its implementation has run,
**When** a reader inspects `spec/features/implement-execution-topology/current-branch/README.md`,
**Then** REQ `protected-branch-commit-confirmation` no longer requires human confirmation before a commit onto a protected branch (the checkpoint having moved to promote/push), and `specscore spec lint` passes.

## Rehearse Integration

Every AC is testable against a git-fixture harness driving an `implement` run with mocked gate configs, mocked subagents, and injected anomalies (conflict, BLOCKED, lint-fail, drift) — asserting event-firing order, commit cadence, halt/no-auto-resolve, re-arm gating, cumulative-review content, push refusal, config-namespace parsing, and `when:`-conditioned participation. The `current-branch-reconciled` AC is a grep-style assertion on the revised file. Per the Rehearse heuristic these are testable, but stub scaffolding is deferred to plan time (the harness is uniform across the set and is best authored as one cohesive `_tests/` set when the plan defines it) — an explicit recorded deferral, not a "not testable" skip.

## Open Questions

- The exact re-arm signal vocabulary (`continue` vs. an explicit `approve`-style phrase vs. a CLI flag) — settle at plan time alongside the anomaly-halt surfacing UX.
- Whether `commit_cadence: plan` (one commit for the whole run) needs additional guard rails given it defers all commits to the end (interacts with the per-batch integration boundary in the worktree topology) — revisit at plan time.
- Whether a future `autonomy.implement` knob should let the anomaly-halt set be tuned (e.g., treat `DONE_WITH_CONCERNS` as a halt) — out of MVP; the halt set is fixed here.
- The concrete "very large diff" threshold at which `cumulative-review-at-pre-push` switches from a consolidated diff to a per-commit summary — left to implementation for MVP; pin a heuristic at plan time so the `cumulative-review-presented` harness assertion is deterministic.
- The exact `when:` match grammar (regex vs glob) for the branch-pattern condition — owned by the `reviewer-gates` `when:` addition (`reviewer-gates-when-condition`); pin the dialect there at plan time so this layer and the gates layer don't guess differently.

---
*This document follows the https://specscore.md/feature-specification*
