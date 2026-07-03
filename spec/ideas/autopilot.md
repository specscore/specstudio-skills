---
format: https://specscore.md/idea-specification
status: Specified
---

# Idea: Autopilot: end-to-end autonomous run (raw idea → open PR)

**Status:** Specified
**Date:** 2026-07-03
**Owner:** alexander.trakhimenok
**Promotes To:** autopilot
**Supersedes:** —
**Related Ideas:** depends_on:approval-autonomy

## Problem Statement

How might we let a single trigger drive an idea — from a cold raw prompt or any partial artifact — all the way to an open MVP pull request, pausing for a human only once (at the crystallized Idea) and halting only on genuine anomalies?

## Context

The user wants to say "do it autonomously" — even on the very first prompt while ideating — and have the pipeline run to an MVP without per-question stops. The existing approval-autonomy Idea+Feature already deliver the implement/plan half (commit-autonomous, push-gated, expressed as reviewer-gates config under autonomy.implement, with anomaly-halts and a cumulative push review), but explicitly scope OUT ideate/specify and carry no orchestrator. Two gaps remain: (1) nothing chains ideate->specify->plan->implement->PR as one run, and (2) autonomy today releases APPROVAL gates but has no story for the CLARIFYING QUESTIONS that ideate/specify ask — a distinct concern from approval.

## Recommended Direction

Add a new thin orchestrator skill, `specstudio:autopilot`, plus a **run-scoped autonomy context** that the existing producer skills read. Autopilot re-implements no producer logic; it drives the pipeline and reuses `approval-autonomy` unchanged for the implement/plan half.

1. **Entry-point detection, not a precondition.** The trigger ("do it autonomously", "/autopilot", or "autonomously" appended to any pipeline request) inspects the input and repo for the furthest-along artifact, then runs every remaining stage: raw prompt → ideate→specify→plan→implement→pull-request; Draft Idea → same minus a fresh start; Approved Idea → specify onward; Approved Feature → plan onward; Approved Plan → implement onward. The only refusal is a genuinely empty ask. The trigger phrase *is* the consent to auto-create and auto-approve every artifact the run produces.

2. **Run-scoped human-gate masking — generalize the existing `when:` mask.** `reviewer-gates`' runner already defines masked-`type: human` semantics ("neither dispatched nor counted; the gate releases on its remaining entries alone") and calls this "the home for per-branch autonomy masks." Generalize that one mechanism from branch-scoped (`when:`) to also honor a run-scoped autonomy signal, so autopilot releases the human entries on `gates.specify`, `plan`, `implementation.pre_push`, and `pull_request.pre_dispatch` for the run only — no new verdict type, no new gate semantics, and AI/deterministic reviewers still block on `Issues Found`.

3. **Decide-and-record for clarifying questions (the new upstream concern).** Approval-autonomy releases *approval* gates; it says nothing about the *questions* ideate/specify/plan ask. Each producer's question steps gain one branch: when the autonomy context is active, do not call `AskUserQuestion` — pick the skill's own documented default (ideate: highest-conviction Recommended Direction, resolve Open Questions to stated assumptions; specify: proceed-not-decompose, the recommended approach, accept each section; plan: revise-in-place) and record it. Records land in the artifact they belong to: the Idea's `## Key Assumptions`, and a new optional `## Autonomous Decisions` H2 on Feature and Plan.

4. **One checkpoint: `confirm_idea` (default on).** When the run auto-creates an Idea, it pauses exactly once for the user to approve the crystallized Idea before going downstream — the cheapest, highest-leverage place to bound drift from an ambiguous cold prompt. After that single approval the run is unbroken. When the run starts from an already-approved Idea (or later), there is no checkpoint. `confirm_idea: false` makes even the cold start fully unbroken.

5. **Publish ceiling = local commits + one PR.** `implementation.pre_commit` is already `auto-approve` (commits land locally via approval-autonomy); autopilot then invokes `specstudio:pull-request` to push the feature branch and open exactly one PR. It never merges, deploys, or invokes `ship`. The PR URL + the aggregated Autonomous Decisions log are the hand-back.

## Alternatives Considered

- **Config-flag only, no orchestrator (each producer self-chains off its lifecycle event).** Set a run-scoped `autonomy.mode` flag; `specify` auto-invokes on `idea.approved`, `plan` on `feature.approved`, etc. Rejected as the primary shape: the hand-off, entry-point detection, the single `confirm_idea` checkpoint, the PR ceiling, and the aggregated decision-log hand-back have no owner — they get smeared across five skills with no single place to reason about "the run." An orchestrator skill localizes those concerns; the config namespace still exists underneath it.
- **Extend approval-autonomy to cover ideate/specify (no new skill).** Rejected: approval-autonomy is deliberately scoped to the *approval-gate* concern for implement/plan and says so. Clarifying-question autonomy and cross-stage orchestration are genuinely different concerns; bolting them onto approval-autonomy would overload a clean, Grade-A contract. Autopilot depends on approval-autonomy instead of swallowing it.
- **Fully silent cold start (no `confirm_idea` checkpoint at all).** Rejected as the default: auto-approving an Idea shaped from a one-line prompt is where drift risk is highest, and a 30-second Idea read is the cheapest possible guard. Kept as an opt-out (`confirm_idea: false`) for users who want zero stops.
- **Auto-merge / auto-deploy at the end (bypass-style full autonomy).** Rejected: the irreversible action is the value-destroying one. Stopping at an open PR preserves exactly one human review surface for the actual publish without reintroducing per-step stops.

## MVP Scope

A single-repo spike delivering: (a) the `specstudio:autopilot` orchestrator skill covering ideate→specify→plan→implement→pull-request with entry-point detection; (b) the run-scoped autonomy signal and the one-line generalization of `reviewer-gates`' `when:` mask to honor it (releasing the human entries on specify / plan / pre_push / pull_request for the run); (c) decide-and-record branches in ideate/specify/plan and the `## Autonomous Decisions` section on Feature and Plan; (d) the `confirm_idea` single checkpoint (default on); (e) publish ceiling = local commits (reusing approval-autonomy's `implementation.pre_commit`) + one PR via `specstudio:pull-request`; (f) up-front disclosure and an end-of-run hand-back carrying the PR URL and the aggregated decision log. Anomaly-halts and the implement/plan gate layer come from approval-autonomy unchanged.

## Not Doing (and Why)

- Re-implementing implement/plan autonomy — reuse the approval-autonomy Feature's gate-config layer unchanged
- Merge / deploy / ship — autopilot stops at an open PR
- More than one PR per run, PR stacking, or retrying a failed PR delegate — exactly one PR
- Cross-repo master-plan autonomy — single-repo run only for MVP
- Standing/persistent autonomous mode as the primary surface — the verbal per-run trigger is the MVP path
- Weakening AI/deterministic reviewer gates, lint, or conflict detection — only human-approval gates are released

## Key Assumptions to Validate

| Tier | Assumption | How to validate |
|------|------------|-----------------|
| Must-be-true | The `type: human` mask can be generalized from branch-scoped (`when:`) to run-scoped in the `reviewer-gates` runner without a new verdict type or breaking its AND-composition | Confirm with the reviewer-gates owner that a run-scoped mask reuses the exact "neither dispatched nor counted; release on remaining entries" semantics; prototype the masked release on `gates.specify` |
| Must-be-true | `plan`'s and `ideate`'s human review gates are maskable the same way — i.e. they are (or can be moved to) reviewer-gates entries rather than hardcoded prose stops | Audit `plan` (currently a hardcoded user-review gate + baseline reviewer) and `ideate` (`idea.approved` prose gate); decide at specify time whether autopilot masks a gate entry or the skill's autonomy branch skips the prose stop |
| Must-be-true | `specstudio:pull-request` can be driven programmatically to create exactly one PR and reports the URL back to the orchestrator | Confirm the pull-request skill's `pull_request.pre_dispatch` gate + single-PR contract and that it surfaces the created PR URL |
| Should-be-true | Decide-and-record defaults per skill (recommended approach, proceed-not-decompose, revise-in-place) produce a coherent MVP often enough to be useful without a human in the loop | Dogfood autopilot on 2–3 real Ideas end-to-end; measure how often the auto-decisions needed post-hoc correction |
| Should-be-true | A single `confirm_idea` checkpoint is the right amount of friction for a cold-start run (vs. zero stops or a checkpoint per stage) | Dogfood: run cold-start with `confirm_idea` on and off; judge whether the one Idea read caught divergence cheaply |
| Might-be-true | The verbal per-run trigger is sufficient long-term and a standing project/user autonomous mode is rarely needed | Defer; revisit if users repeatedly hand-set the config at project scope |


## SpecScore Integration

- **New Features this would create:** an `autopilot` Feature owning the orchestrator skill (entry-point detection, stage chaining, `confirm_idea` checkpoint, PR ceiling, decision-log hand-back) and the run-scoped autonomy context; plus the `## Autonomous Decisions` section contract and the decide-and-record branch each producer skill adds.
- **Existing Features affected:** `reviewer-gates` (generalize the `when:` mask to honor a run-scoped autonomy signal), `specstudio-specify-skill` and `specstudio-plan-skill` and the ideate skill (decide-and-record branches + maskable human gate), `pull-request-skill` (driven as the publish-ceiling delegate), `change-publication-policy` (the run-scope rung of the ladder carries the autonomy signal).
- **Dependencies:** **depends on `approval-autonomy`** for the implement/plan gate-release layer and anomaly-halts (reused unchanged); leans on `reviewer-gates` for the mask mechanism and `change-publication-policy` for the scope ladder.

## Open Questions

- Is the run-scoped human-gate release best expressed as (a) a generalized `when:`/mask signal in the reviewer-gates runner, or (b) a run-scope resolution of an `auto-approve` reviewer via the publication-policy scope ladder? Lean (a) — it reuses the mechanism reviewer-gates already documents as the home for autonomy masks. Decide with the reviewer-gates owner at specify time.
- Should `confirm_idea` also offer a "confirm Feature" variant (a second optional checkpoint at the spec) for higher-stakes work, or is one Idea checkpoint the whole story for MVP? Lean: one checkpoint for MVP; Feature-level checkpoint is an additive overlay later.
- When autopilot enters mid-pipeline (e.g. at an Approved Feature), does the run-scoped autonomy signal need re-arming semantics analogous to approval-autonomy's post-anomaly re-arm, or is the single trigger arming sufficient for the whole run? Lean: single arming per run; anomaly-halt re-arm is inherited from approval-autonomy for the implement stage.
