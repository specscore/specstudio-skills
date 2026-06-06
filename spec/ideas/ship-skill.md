---
format: https://specscore.md/idea-specification
status: Specified
---

# Idea: Ship Skill

**Status:** Specified
**Date:** 2026-06-03
**Owner:** alexander.trakhimenok
**Promotes To:** skills/ship
**Supersedes:** —
**Related Ideas:** —

## Problem Statement

How might we let a SpecStudio user release a Feature with confidence that every spec-aware gate has passed, while delegating the risky deploy execution to a tool the project already trusts?

## Context

The placeholder Feature at spec/features/skills/ship/README.md is the last unbuilt skill in the SpecStudio pipeline (review is archived; lint-fix-staging ships as shared behavior). The handoff from 'code merged' to 'feature shipped' is where deploy-time mistakes happen, and generic shipping skills (e.g. gh-deploy, agent-skills:shipping-and-launch) cover the mechanics but not the spec-aware checks: did every AC's verify report come back green? Did recap surface no contradictions? Did the reviewer-gates score pass? The repo already has a config-driven gate mechanism (reviewer-gates plus the gates: block in specscore.yaml, surfaced manually by the score skill) and a third-party-skill-integration-contracts direction — ship should compose those rather than invent. Upstream the pipeline runs recap.completed then (review, archived) then ship; ship becomes a new terminal lifecycle node (recap then ship) not yet in flexible-lifecycle-flows.

## Recommended Direction

Build ship as a thin, spec-aware **gate-and-dispatch** skill that does exactly three things and refuses to do more. First, it enforces pre-deploy gates by reusing the existing reviewer-gates mechanism through a new `gates: ship.ready:` stage in `specscore.yaml` — verify-green, recap-no-contradiction, an optional AI release-readiness review, and a human go/no-go checkpoint. Second, once the gates pass, it dispatches the deploy to a single configurable delegate skill named in a slim new `ship:` block in `specscore.yaml` (for example `gh-deploy`, `agent-skills:shipping-and-launch`, or a project's own deploy skill); if no delegate is configured it runs the gates and hands back to the user — it never guesses how to deploy. Third, on explicit delegate success it transitions the Feature `Implementing` → `Stable` and emits `ship.completed` with publication policy applied.

The discipline that makes this coherent: Studio **gates, records, and performs a single dispatch — it never executes or orchestrates.** Sequencing, retry, rollback, canary, feature-flag flips, scheduling, and multi-feature/multi-project coordination are explicitly barred from ship and from the `ship:` config; they belong to the delegate/executor. This keeps ship cleanly inside SpecStudio's one-project scope: working inside a single project is Studio's job, while executing and dispatching work across things is the orchestration layer's job.

Why this over the alternatives: building deploy mechanics into ship would duplicate mature deploy tools and force ship to own their failure modes — precisely the high-blast-radius area to avoid. A pure checklist that never transitions status would leave the merged→shipped→Stable handoff manual and break pipeline traceability. Gate + single dispatch + transition is the smallest thing that closes the pipeline honestly while staying on the right side of the architectural boundary.

## Alternatives Considered

- **Ship owns the deploy** (canary, rollback, ordered multi-step pipeline). Lost: duplicates `gh-deploy`/provider tools, inherits their failure modes, and an ordered multi-step execution pipeline is orchestration — outside one-project Studio scope.
- **Gate-only checklist** that never deploys or transitions status. Lost: leaves the merged→shipped→Stable handoff manual, emits no lifecycle signal, and the pipeline never reaches a clean terminal state.
- **Invent a fresh ship-specific gate config block.** Lost: the reviewer-gates `gates:` mechanism already does ordered, typed, human-checkpoint-capable dispatch; a second gate system fragments config and re-implements tested machinery.

## MVP Scope

A single-Feature ship skill: pre-flight on one Feature, run a new 'gates: ship.ready' stage through the existing reviewer-gates layer, and — if a 'ship:' delegate is configured — dispatch one delegate skill; on explicit delegate success, transition the Feature Implementing then Stable and emit ship.completed with publication policy; otherwise gate-and-hand-back. No deploy mechanics of its own. Prove it end-to-end on one Feature via this repo's own dogfooding before adding anything.

## Not Doing (and Why)

- Executing deploy mechanics (push/build/migrate) — delegated to a configured deploy skill; ship never deploys
- Sequencing, retry, rollback, canary, or feature-flag flips — operational execution owned by the delegate, not Studio
- Multi-feature or multi-project release coordination — cross-thing orchestration is out of one-project Studio scope
- Scheduling or dispatching work across runs — that belongs to the orchestration layer, not ship
- Resolving the reviewer type-enum extension (cli/skill) inside this Idea — flagged as a reviewer-gates dependency, decided at spec time

## Key Assumptions to Validate

| Tier | Assumption | How to validate |
|------|------------|-----------------|
| Must-be-true | The reviewer-gates mechanism can express ship's pre-deploy checks (verify-green, recap-no-contradiction), likely needing a reviewer type beyond today's `ai`/`human`. | Prototype a `gates: ship.ready` stage; confirm whether a `cli` (and maybe `skill`) reviewer type is required and whether reviewer-gates can adopt it. |
| Must-be-true | A single dispatch to one delegate skill covers real deploy needs without ship owning sequencing. | Wire `gh-deploy` as a delegate against a real Feature; confirm a one-shot handoff suffices or surfaces the (barred) need for sequencing. |
| Should-be-true | `Implementing` → `Stable` is the correct transition for a shipped Feature. | Confirm against the change-status lifecycle (Draft→…→Implementing→Stable→Deprecated). |
| Should-be-true | recap-no-contradiction should be a hard gate (recap mandatory before ship). | Validate via dogfooding: is a contradiction-free recap always available at ship time? |
| Might-be-true | Teams will want an AI release-readiness reviewer distinct from the spec/score reviewer. | Defer; observe whether projects configure one. |


## SpecScore Integration

- **New Features this would create:** a real `skills/ship` Feature replacing the Draft placeholder; possibly a small `ship:`-config Feature (or fold the schema into an existing config Feature).
- **Existing Features affected:** `reviewer-gates` (new `ship.ready` stage and likely a new reviewer `type`); `flexible-lifecycle-flows` (add the `recap → ship` terminal node); `change-publication-policy` (the `ship.completed` checkpoint); `third-party-integration` (the delegate handoff contract).
- **Dependencies:** `third-party-skill-integration-contracts` (Idea) for the delegate handoff shape; the reviewer-gates reviewer-`type` extension.

## Open Questions

- Does the `gates: ship.ready` stage require a new reviewer `type` (`cli`, and possibly `skill`), and does that change belong to `reviewer-gates` rather than ship?
- Single delegate only, or an explicitly ordered delegate list? (Ordered multi-step risks crossing into orchestration — leaning single.)
- Default when no `ship:` delegate is configured: gate-then-hand-back (safe) versus refuse outright.
- Is `Implementing` → `Stable` the right transition, and does ship own it or does lifecycle tooling?
- Should recap (no-contradiction) be a hard gate or a soft/waivable one?
- Where does the `ship:` schema live — its own config Feature, or an extension of an existing one?

---
*This document follows the https://specscore.md/idea-specification*
