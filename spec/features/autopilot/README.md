---
format: https://specscore.md/feature-specification
status: Approved
---

# Feature: Autopilot: end-to-end autonomous run

> [SpecScore.**Studio**](https://specscore.studio): | [Explore](https://specscore.studio/app/github.com/specscore/specstudio-skills/spec/features/autopilot?op=explore) | [Edit](https://specscore.studio/app/github.com/specscore/specstudio-skills/spec/features/autopilot?op=edit) | [Ask question](https://specscore.studio/app/github.com/specscore/specstudio-skills/spec/features/autopilot?op=ask) | [Request change](https://specscore.studio/app/github.com/specscore/specstudio-skills/spec/features/autopilot?op=request-change) |
**Status:** Approved
**Date:** 2026-07-03
**Owner:** alexander.trakhimenok
**Source Ideas:** autopilot
**Supersedes:** —
**Grade:** B

## Summary

A thin orchestrator skill, `specstudio:autopilot`, that drives an idea from any pipeline entry point — a cold raw prompt included — to one open MVP pull request, pausing only at a single Idea checkpoint and halting on genuine anomalies. It re-implements no producer logic: it chains the existing `ideate → specify → plan → implement → pull-request` skills and reuses the [`approval-autonomy`](../approval-autonomy/README.md) Feature unchanged for the implement/plan gate layer. Its net-new surface is (a) cross-stage orchestration with entry-point detection, (b) a run-scoped generalization of `reviewer-gates`' human-gate mask, (c) *decide-and-record* for clarifying questions (distinct from approval gates), (d) the single `confirm_idea` checkpoint, and (e) a local-commit + one-PR publish ceiling.

## Problem

Today reaching an MVP from an idea means walking four interactive producer skills, each stopping for both clarifying questions and approval gates. `approval-autonomy` already released the *approval* gates for `implement`/`plan` (via `reviewer-gates` config) but deliberately scoped out `ideate`/`specify` and provides no orchestrator, and no layer addresses the *clarifying questions* the upstream skills ask. The user wants a single trigger — usable even on the very first prompt while ideating — that runs the whole pipeline to an open PR without per-question stops, pausing only once to confirm the crystallized Idea (the cheapest place to bound drift from an ambiguous cold prompt) and halting loudly on genuine anomalies. This Feature owns the orchestrator and the upstream autonomy that `approval-autonomy` left open.

## Behavior

### Orchestration and entry

#### REQ: entry-point-detection

Autopilot MUST resolve the run's entry point by detecting the furthest-along existing artifact for the work and then run every remaining pipeline stage in order: a raw prompt or a `Draft`/`In Review` Idea enters at `ideate`; an `Approved` Idea enters at `specify`; an `Approved` Feature with no Plan enters at `plan` (or `implement`, per `implement`'s Feature-sourced mode); an `Approved` Plan enters at `implement`. It MUST refuse only when the input is a genuinely empty ask (no prompt and no resolvable artifact).

#### REQ: stage-chaining

Autopilot MUST drive the stages by invoking the existing producer skills (`specstudio:ideate`, `specstudio:specify`, `specstudio:plan`, `specstudio:implement`, `specstudio:pull-request`) in order, advancing to the next stage only after the current stage completes successfully. It MUST NOT reimplement any producer's logic, gates, or artifact writes.

#### REQ: up-front-disclosure

Before running any stage, autopilot MUST disclose in one message: the resolved entry point, the stages that will run, that human-approval gates are masked for the run *except* the single Idea checkpoint, that clarifying decisions will be auto-made and recorded, and that the run will commit locally and open exactly one PR (never merge or deploy).

#### REQ: handback

On successful completion autopilot MUST hand back a summary containing the created artifact paths, the local commit SHAs, the opened PR URL, and the aggregated Autonomous Decisions log. It MUST NOT auto-invoke `specstudio:verify` or `specstudio:ship`; it MAY surface a recommendation to run them.

### Run-scoped autonomy release

#### REQ: run-scoped-gate-mask

Autopilot MUST establish a run-scoped autonomy signal that masks `type: human` reviewer entries on the event-keyed gates it drives (`gates.feature.approved`, `gates.plan.approved`, `implementation.pre_push`, `pull_request.pre_dispatch`) using the identical masking semantics `reviewer-gates` defines for a non-matching `when:` condition — the entry is neither dispatched nor counted toward the verdict, and the gate releases on its remaining entries alone. This generalizes the `reviewer-gates` mask from branch scope to run scope and introduces no new verdict type.

#### REQ: quality-gates-never-masked

The autonomy signal MUST mask only `type: human` entries. Every `type: ai` and `type: deterministic` reviewer, `specscore spec lint`, and `implement`'s conflict detection MUST still run; an `Issues Found`, non-zero, or conflict outcome MUST halt the run and surface the blocker rather than be released by autonomy.

### Clarifying-question autonomy

#### REQ: decide-and-record

When the run-scoped autonomy signal is active, each producer skill's clarifying-question steps MUST NOT call `AskUserQuestion`; instead they MUST select the skill's own documented default (`ideate`: the highest-conviction Recommended Direction and each Open Question resolved to a stated assumption; `specify`: proceed-not-decompose for single-scope intent, the recommended approach, each section accepted as drafted; `plan`: revise-in-place on an existing plan, or for a fresh plan with no prior artifact accept the generated task breakdown and deferred-AC set as computed) and record the choice per REQ: autonomous-decisions-log.

#### REQ: autonomous-decisions-log

Every auto-made clarifying decision MUST be recorded in the artifact it belongs to: Idea-stage assumptions in the Idea's `## Key Assumptions`; Feature- and Plan-stage decisions in a `## Autonomous Decisions` section carrying one bullet per decision (what was decided, the alternatives, why the default was chosen). The section MUST remain lint-clean and MUST be omitted when the stage auto-made no decision.

### Idea checkpoint

#### REQ: confirm-idea-checkpoint

When the run auto-creates an Idea (entry at or before the Idea stage), autopilot MUST pause exactly once to obtain the user's explicit approval of the crystallized Idea before proceeding downstream, UNLESS `autonomy.autopilot.confirm_idea` is `false`. This is the only human-approval stop in an autonomous run. When the run enters at or after an already-approved Idea, no checkpoint fires. `confirm_idea` defaults to `true`.

### Publish ceiling

#### REQ: publish-ceiling-one-pr

Autopilot's publish ceiling MUST be local commits plus exactly one pull request: commits land via the existing `implementation.pre_commit` gate (autonomous per `approval-autonomy`), after which autopilot invokes `specstudio:pull-request` once to push the branch and open one PR. Autopilot MUST NOT merge, deploy, invoke `ship`, open more than one PR, or retry a failed PR delegate.

### Arming and configuration

#### REQ: single-arming-per-run

The trigger MUST arm autonomy once for the whole run; no per-stage re-arming is required to keep the run moving. The `implement` stage still inherits `approval-autonomy`'s post-anomaly explicit-re-arm requirement unchanged (an anomaly-halt during implementation is a genuine stop, not a per-stage gate).

#### REQ: config-namespace

Autopilot's knobs MUST live under the `autonomy.autopilot` namespace: `publish_ceiling` (`stage` | `commit` | `pr`, default `pr`), `confirm_idea` (default `true`), and `stop_on` (a list including a non-removable `conflict`). The verbal trigger MUST set the run-scoped autonomy signal at run scope on the `change-publication-policy` scope ladder, overriding nothing durable and evaporating at run end.

## Dependencies

- approval-autonomy
- reviewer-gates
- change-publication-policy

## Acceptance Criteria

### AC: enters-at-furthest-artifact (verifies REQ: entry-point-detection)

**Given** a repo whose furthest-along artifact for the work is an `Approved` Feature with no Plan,
**When** an autonomous run is triggered,
**Then** the run enters at the `plan` stage and runs `plan → implement → pull-request` (it does not re-run `ideate` or `specify`); **and given** only a raw prompt with no artifact, **then** the run enters at `ideate`; **and given** no prompt and no resolvable artifact, **then** the run refuses.

### AC: chains-producers-in-order (verifies REQ: stage-chaining)

**Given** a cold-start autonomous run,
**When** it executes,
**Then** it invokes `ideate`, `specify`, `plan`, `implement`, and `pull-request` in that order, entering each only after the prior stage completed successfully, and reimplements none of their logic.

### AC: discloses-before-running (verifies REQ: up-front-disclosure)

**Given** an autonomous run about to start,
**When** it begins,
**Then** before any stage runs it emits one disclosure message naming the resolved entry point, the stages to run, the gate-masking (except the Idea checkpoint), the decide-and-record behavior, and the local-commit + one-PR ceiling.

### AC: hands-back-pr-and-log (verifies REQ: handback)

**Given** an autonomous run that completed,
**When** it hands back,
**Then** the summary contains the created artifact paths, the local commit SHAs, the opened PR URL, and the aggregated Autonomous Decisions log, and it did not auto-invoke `verify` or `ship`.

### AC: masks-human-entries-run-scoped (verifies REQ: run-scoped-gate-mask)

**Given** `gates.feature.approved` (the specify-stage gate) with a `type: ai` reviewer and a `type: human` reviewer and an active run-scoped autonomy signal,
**When** the specify gate is evaluated,
**Then** the `type: human` entry is neither dispatched nor counted and the gate releases on the `type: ai` reviewer's `Approved` verdict alone, using the same semantics as a non-matching `when:` condition.

### AC: quality-gates-still-block (verifies REQ: quality-gates-never-masked)

**Given** an active autonomy signal and a `type: ai` reviewer that returns `Issues Found` (or a lint failure, or an `implement` conflict),
**When** the run reaches that gate or check,
**Then** the run halts and surfaces the blocker — autonomy does not release it — because only `type: human` entries are masked.

### AC: auto-decides-without-asking (verifies REQ: decide-and-record)

**Given** an active autonomy signal at the `specify` approach-proposal step,
**When** the step is reached,
**Then** no `AskUserQuestion` is issued, the recommended approach is selected, and the choice is recorded per the decisions-log requirement.

### AC: records-decisions-in-artifact (verifies REQ: autonomous-decisions-log)

**Given** a stage that auto-made one or more clarifying decisions,
**When** the artifact is written,
**Then** Idea-stage decisions appear in the Idea's `## Key Assumptions` and Feature/Plan-stage decisions appear as bullets in a lint-clean `## Autonomous Decisions` section; **and given** a stage auto-made no decision, **then** no `## Autonomous Decisions` section is written.

### AC: pauses-once-at-idea (verifies REQ: confirm-idea-checkpoint)

**Given** a cold-start run with `confirm_idea` unset (default `true`),
**When** the Idea is crystallized,
**Then** the run pauses exactly once for the user's approval before `specify` and makes no other human-approval stop; **and given** `confirm_idea: false`, **then** no pause occurs; **and given** the run entered at an already-`Approved` Idea, **then** no checkpoint fires.

### AC: ceiling-is-one-pr (verifies REQ: publish-ceiling-one-pr)

**Given** an autonomous run that reached implementation,
**When** it finishes the code,
**Then** commits land locally via `implementation.pre_commit` and `specstudio:pull-request` is invoked exactly once to open a single PR, and the run performs no merge, deploy, `ship`, second PR, or delegate retry.

### AC: armed-once-for-run (verifies REQ: single-arming-per-run)

**Given** an autonomous run that spans multiple stages,
**When** it crosses stage boundaries,
**Then** no re-arming prompt is required to continue; **and given** an anomaly-halt during `implement`, **then** `approval-autonomy`'s explicit-re-arm requirement still applies before implementation resumes.

### AC: knobs-under-autonomy-autopilot (verifies REQ: config-namespace)

**Given** `specscore.yaml` with no `autonomy.autopilot` block,
**When** an autonomous run resolves its config,
**Then** `publish_ceiling` is `pr`, `confirm_idea` is `true`, and `stop_on` includes `conflict`; **and given** the verbal trigger, **then** the autonomy signal is set at run scope on the publication-policy scope ladder and does not persist past the run.

## Rehearse Integration

Every AC is testable against a harness that drives an autopilot run over git fixtures with mocked producer-skill invocations, mocked gate configs, and injected states (varied entry-point artifacts, an `Issues Found` reviewer, a lint failure, an `implement` conflict, `confirm_idea` on/off) — asserting entry-point resolution, stage order, the disclosure message, run-scoped human-entry masking vs. quality-gate blocking, absence of `AskUserQuestion` under autonomy, decisions-log placement, the single Idea pause, one-PR ceiling, single-arming, and config resolution. Per the Rehearse heuristic these are testable; stub scaffolding is deferred to plan time (the harness is uniform across the set and is best authored as one cohesive `_tests/` set when the plan defines it) — an explicit recorded deferral, not a "not testable" skip.

## Open Questions

- The exact run-scoped autonomy signal transport (an in-conversation context the orchestrator threads vs. a `when:`-style condition the runner reads vs. a run-scope ladder entry) — settle with the `reviewer-gates` owner at plan time; the masking *semantics* are fixed here, only the carrier is open.
- Whether `plan`'s and `ideate`'s human review gates are masked as `reviewer-gates` entries or bypassed by an autonomy branch in the skill's prose (they are less gate-config-driven than `specify` today) — audit and pin at plan time.
- The concrete re-arm signal vocabulary shared with `approval-autonomy` when an anomaly-halt interrupts an otherwise autonomous run — align with that Feature's open question at plan time rather than inventing a second dialect.

---
*This document follows the https://specscore.md/feature-specification*
