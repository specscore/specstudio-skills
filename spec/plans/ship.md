# Plan: Ship Skill

**Status:** Implemented
**Source Feature:** skills/ship
**Date:** 2026-06-03
**Owner:** alexander.trakhimenok
**Supersedes:** —

## Summary

Decomposes the approved `skills/ship` Feature into six ordered, topic-bundled tasks that build `specstudio:ship` end-to-end: pre-flight machine gates, a reviewer gate, single-delegate deploy dispatch, lifecycle transition, and the architectural boundary. Each task maps to one topic group in the Feature's `## Behavior` and covers that group's acceptance criteria exactly once.

## Approach

Tasks mirror the Feature's six `## Behavior` topic groups, one bundled task per group, ordered to follow the skill's runtime flow (input → pre-flight → reviewer gate → dispatch → transition/event → boundary). Per the chosen inline strategy, the precedented cross-Feature registrations the Feature's `## Dependencies` calls for are folded into the task that consumes each one rather than split into separate Feature work: the ship gate-point event registration lands in Task 3 (the reviewer-gate task), `ship.completed` registration in Task 5 (the event task), the `ship:` config schema in Task 4 (the dispatch task), and the `recap → ship` lifecycle node in Task 6 (the boundary task). The four Feature-level open questions are resolved inside their consuming tasks: recap-gate waivability → Task 2 (resolved: strictly mandatory in MVP); gate-point event identifier → Task 3 (proposed `ship.pre_dispatch`, final name owned by `reviewer-gates`); transition ownership → Task 5 (resolved: ship invokes `specscore feature change-status`); `ship:` schema home → Task 6 (resolved: defined by this Feature for MVP). All 11 ACs are covered; none are deferred.

## Tasks

### Task 1: Skill scaffold, input resolution, and status pre-flight

**Verifies:** skills/ship#ac:rejects-non-feature-input, skills/ship#ac:refuses-non-implementing-status

Create `skills/ship/SKILL.md` with the standard hard-gate/when-to-use/checklist shape used by sibling skills (verify, recap). Implement single-positional-argument resolution to `spec/features/<feature-slug>/README.md` (reject zero or multiple arguments with a usage error and non-zero exit) and the status pre-flight that refuses unless the Feature's `**Status:**` is `Implementing`, printing the current Status on refusal.

### Task 2: Pre-flight machine gates — verify-green and recap-no-contradiction

**Verifies:** skills/ship#ac:refuses-when-verify-not-green, skills/ship#ac:refuses-on-recap-contradiction

Resolve the latest `_verify/<sha>.md` and `_recap/<sha>.md` reports reachable at HEAD (reusing verify/recap's report-resolution convention) and parse their YAML summary blocks. Refuse unless every verify AC verdict is `pass` and the recap report contains zero `contradiction` verdicts; refuse with a recommendation to run `specstudio:verify` / `specstudio:recap` when a report is missing. The recap-no-contradiction gate is strictly mandatory in the MVP (no waiver path).

### Task 3: Register the ship gate-point event and dispatch the reviewer gate

**Verifies:** skills/ship#ac:gate-releases-only-on-all-approved

Register a ship gate-point event (proposed identifier `ship.pre_dispatch`, final name owned by `reviewer-gates`) in the `reviewer-gates` gate-point event registry and the `events.md` catalog, mirroring `implementation.pre_commit` / `implementation.pre_push`. Wire the reviewer gate to load and dispatch `gates.<ship-event>.reviewers` via the shared [loader](../../skills/shared/reviewer-gates/loader.md) and [runner](../../skills/shared/reviewer-gates/runner.md), releasing only on AND-composed `Approved` and halting (no delegate dispatch) on any `Issues Found`. No new reviewer type is introduced — only `ai` and `human` entries.

### Task 4: Define the `ship:` config schema and implement deploy dispatch

**Verifies:** skills/ship#ac:dispatches-single-configured-delegate, skills/ship#ac:hands-back-when-no-delegate, skills/ship#ac:no-transition-on-delegate-failure

Define the slim `ship:` config block in `specscore.yaml` (a single `delegate` with `skill` and `args`), bounded so it exposes no sequencing/retry/rollback/canary/flag-flip/scheduling fields. Implement dispatch: when a delegate is configured, invoke exactly one delegate skill once with its args; when none is configured, summarize gate results and hand back without deploying or transitioning; treat only an explicit delegate success as success, leaving the Feature in `Implementing` (no retry) on failure or ambiguous result.

### Task 5: Lifecycle transition and ship.completed emission

**Verifies:** skills/ship#ac:transitions-to-stable-on-success, skills/ship#ac:emits-ship-completed-once

On explicit delegate success, transition the Feature `Implementing → Stable` by invoking `specscore feature change-status <feature-slug> --to Stable` (ship owns the transition write). Register the `ship.completed` event in the `events.md` catalog, apply publication policy for its checkpoint, and emit `ship.completed` exactly once on a successful ship with `publication_result` — never on refusal or no-delegate hand-back.

### Task 6: Enforce the architectural boundary and wire the lifecycle node

**Verifies:** skills/ship#ac:bars-execution-and-orchestration

Encode the boundary as load-bearing skill text and config discipline: ship performs no deploy mechanics, sequencing, retry, rollback, canary, flag-flips, scheduling, or multi-feature coordination, and the `ship:` schema (owned by this Feature for the MVP) rejects fields expressing those concerns. Add the terminal `recap → ship` node to the `flexible-lifecycle-flows` valid-flow graph so ship is a recognized downstream of recap.

## Deferred AC Coverage

The following ACs were added to the Ship Feature by the `right-size-recap-cost` Idea (configurable recap-gate waiver) after this Plan was written. They are not yet tasked; they will be planned when the recap-waiver work is scheduled.

- skills/ship#ac:proceeds-when-recap-waived
- skills/ship#ac:records-recap-waiver

## Open Questions

- Final canonical identifier for the ship gate-point event (`ship.pre_dispatch` is the working proposal); naming and registry ownership are confirmed with `reviewer-gates` during Task 3.

---
*This document follows the https://specscore.md/plan-specification*
