---
format: https://specscore.md/plan-specification
status: Approved
---
# Plan: Autopilot

**Status:** Approved
**Source Feature:** autopilot
**Date:** 2026-07-03
**Owner:** alex
**Supersedes:** —

## Summary

Implements the `specstudio:autopilot` Feature: a new orchestrator skill plus the upstream autonomy surface (`ideate`/`specify`/`plan`), a run-scoped generalization of the `reviewer-gates` human-gate mask, the `autonomy.autopilot` config namespace, the single `confirm_idea` checkpoint, and the local-commit + one-PR publish ceiling. Work spans four subsystems: `specscore.yaml`/config resolution, `skills/shared/reviewer-gates/`, the producer skills (`skills/ideate`, `skills/specify`, `skills/plan`), and the new `skills/autopilot/`. It reuses the `approval-autonomy` implement/plan gate layer unchanged.

## Approach

Bottom-up: establish the config namespace and the run-scoped autonomy signal first (Task 1), then the two primitives the orchestrator consumes — the reviewer-gates mask (Task 2) and the producer decide-and-record branches (Task 3), which are independent of each other. The orchestrator core (Task 4) depends on all three. The three orchestrator behaviors that build on that core — the Idea checkpoint (Task 5), the publish ceiling (Task 6), and disclosure/handback (Task 7) — depend only on Task 4 and can be built in parallel. Tasks are numbered linearly; `**Depends-On:**` encodes the real dependency graph so the implement engine can batch independent tasks. No ACs are deferred — all 12 map to a task.

## Tasks

### Task 1: Config namespace and run-scoped autonomy signal

**Verifies:** autopilot#ac:knobs-under-autonomy-autopilot
**Depends-On:** —
**Status:** complete

Define the `autonomy.autopilot` config block (`publish_ceiling` default `pr`, `confirm_idea` default `true`, `stop_on` including a non-removable `conflict`) and its defaults-when-absent resolution, and specify the run-scoped autonomy signal set on the `change-publication-policy` scope ladder at run scope (evaporating at run end). This is the shared foundation every later task reads.

### Task 2: Run-scoped human-gate mask in reviewer-gates

**Verifies:** autopilot#ac:masks-human-entries-run-scoped, autopilot#ac:quality-gates-still-block
**Depends-On:** 1
**Status:** complete

Generalize the `skills/shared/reviewer-gates/runner.md` mask from branch-scoped (`when:`) to also honor the run-scoped autonomy signal from Task 1: a `type: human` entry is neither dispatched nor counted and the gate releases on its remaining entries. Mask only `type: human` entries — `type: ai`/`type: deterministic` reviewers, lint, and conflict detection still run and still block on `Issues Found`/non-zero/conflict.

### Task 3: Decide-and-record branches and the Autonomous Decisions section

**Verifies:** autopilot#ac:auto-decides-without-asking, autopilot#ac:records-decisions-in-artifact
**Depends-On:** 1
**Status:** complete

Add an "if the autonomy signal is active" branch to the clarifying-question steps of `skills/ideate`, `skills/specify`, and `skills/plan` that selects each skill's documented default instead of calling `AskUserQuestion`, and define the lint-clean, omit-when-empty `## Autonomous Decisions` section (Feature/Plan) plus the Idea `## Key Assumptions` placement for recorded decisions.

### Task 4: Autopilot orchestrator core — entry detection, chaining, single arming

**Verifies:** autopilot#ac:enters-at-furthest-artifact, autopilot#ac:chains-producers-in-order, autopilot#ac:armed-once-for-run
**Depends-On:** 1, 2, 3
**Status:** complete

Author `skills/autopilot/SKILL.md`: entry-point detection (resolve the furthest-along artifact; refuse only an empty ask), in-order invocation of `ideate → specify → plan → implement → pull-request` advancing only on each stage's success, and single-arming (arm the run-scoped signal once; `implement` still inherits `approval-autonomy`'s post-anomaly re-arm). Reimplements no producer logic.

### Task 5: Single Idea checkpoint (confirm_idea)

**Verifies:** autopilot#ac:pauses-once-at-idea
**Depends-On:** 4
**Status:** planning

Implement the one human-approval stop: when the run auto-creates an Idea and `confirm_idea` is not `false`, pause once for explicit Idea approval before `specify`; skip the pause when `confirm_idea: false` or when the run enters at/after an already-Approved Idea.

### Task 6: Publish ceiling — local commits plus one PR

**Verifies:** autopilot#ac:ceiling-is-one-pr
**Depends-On:** 4
**Status:** planning

Wire the publish ceiling: commits land via the existing `implementation.pre_commit` gate, then invoke `specstudio:pull-request` exactly once to push the branch and open a single PR. Bar merge, deploy, `ship`, a second PR, and delegate retry.

### Task 7: Up-front disclosure and end-of-run handback

**Verifies:** autopilot#ac:discloses-before-running, autopilot#ac:hands-back-pr-and-log
**Depends-On:** 4
**Status:** planning

Implement the single pre-run disclosure message (entry point, stages, gate-masking except the Idea checkpoint, decide-and-record, local-commit + one-PR ceiling) and the completion handback (artifact paths, commit SHAs, PR URL, aggregated Autonomous Decisions log), with no auto-invocation of `verify`/`ship`.

## Open Questions

- The source Feature defers its Rehearse `_tests/` harness "to plan time." This Plan folds that authoring into implementation — each task lands its own behavior tests as part of the change rather than a dedicated trailing task — so the harness is not silently dropped. Confirm at implement time, or split out a dedicated final test-harness task if a single cohesive `_tests/` set is preferred.
- The three Feature-level open questions (run-scoped autonomy-signal carrier; `plan`/`ideate` gate maskability; shared re-arm vocabulary with `approval-autonomy`) are settled during implementation of Tasks 1–4 against the `reviewer-gates` and `approval-autonomy` contracts.

---
*This document follows the https://specscore.md/plan-specification*
