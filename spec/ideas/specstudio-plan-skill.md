# Idea: SpecStudio plan skill

**Status:** Implementing
**Date:** 2026-04-20
**Owner:** alexander.trakhimenok
**Promotes To:** skills/plan
**Supersedes:** —
**Related Ideas:** —

## Problem Statement

How might we turn an approved SpecScore Feature into a small set of verifiable, ordered tasks without re-litigating the spec?

## Context

SpecStudio today ships two skills: `ideate` produces a lint-clean `spec/ideas/<slug>.md`, and `specify` produces a lint-clean `spec/features/<slug>/`. The README's roadmap names `plan` as the next lifecycle phase, but no skill exists to bridge from a lint-clean Feature to actionable work.

In the gap, users fall back to generic planning skills (`superpowers:writing-plans`, `agent-skills:planning-and-task-breakdown`). Those skills are fine as process, but they are SpecScore-blind: they do not consume Feature body metadata, they do not map tasks to acceptance-criteria IDs, they do not lint, and they do not emit Synchestra events. That means the spec↔code coherence story — a stated SpecStudio principle — breaks exactly at the handoff where it matters most.

This Idea proposes a `specstudio:plan` skill that consumes an approved Feature and produces a lint-clean `spec/plans/<slug>.md` of ordered, AC-mapped tasks, preserving the same gate discipline `ideate` and `specify` already enforce.

## Recommended Direction

Ship a **thin, lint-gated `plan` skill** that mirrors the shape of `ideate` and `specify`: read an approved Feature, run a short convergent task-breakdown, and write a SpecScore-compliant Plan artifact at `spec/plans/<slug>.md`. The plan's core payload is an ordered list of tasks where each task references one or more AC IDs from its source Feature. The skill hard-gates on lint-clean output and user approval, then emits `plan.drafted` / `plan.approved` events.

This is the narrowest thing that closes the most-painful gap. It keeps the "gates are non-negotiable" philosophy intact, keeps the artifact surface familiar to anyone who has used `ideate` or `specify`, and leaves the more speculative directions (Rehearse test scaffolding, runner dispatch) as follow-on Ideas rather than prerequisites.

The MVP does not need a CLI scaffold — it can follow the same fallback direct-write path `ideate` already supports — but should be structured so a future `specscore new plan` command can take over without skill changes.

## Alternatives Considered

- **Fold planning into `specify`.** Rejected. Collapsing two gates into one removes the user-review checkpoint where spec intent is verified *before* it is decomposed into work. The gates earn their keep.
- **Plan as a pure markdown template with no skill.** Rejected. A template without lint and a gate is a suggestion, not a contract. SpecStudio's value is exactly the contract.
- **Plan as a Rehearse-test scaffolder first.** Rejected *for the MVP*. Strong idea in a Rehearse-first workflow, but Rehearse's markdown format is still evolving and coupling the plan skill to it now would bind the MVP to a moving target. Worth revisiting as a separate Idea.
- **Plan as a dispatcher to Synchestra runners.** Rejected *for the MVP*. Dispatch matters eventually, but the planning format needs to stabilize before it's worth hooking into Hub. Premature coupling.

## MVP Scope

A two-week spike: implement `specstudio:plan` as a single skill folder under `skills/plan/`, producing a lint-clean `spec/plans/<slug>.md` from an approved Feature. Ship it, dogfood it on one real Feature in this repo, and stop. No Rehearse scaffolding, no dispatch, no editor UI. If the artifact isn't embarrassingly minimal on day one, it waited too long.

## Not Doing (and Why)

- Automatic code generation from plan tasks — belongs in a future `implement` skill, not `plan`.
- Rehearse test-stub scaffolding — separate Idea; couples to an evolving format.
- Dispatch of tasks to Synchestra runners or subagents — premature until plan format stabilizes and Hub matures.
- Cross-Feature or roadmap-level planning — different problem, different Idea.
- A web authoring UI for plans — roadmap concern for Hub, not this skill.
- Effort estimation, velocity, or scheduling — out of scope; SpecScore is about verifiable contracts, not project management.

## Key Assumptions to Validate

| Tier | Assumption | How to validate |
|------|------------|-----------------|
| Must-be-true | The Feature → Plan handoff is the actually-painful gap, and users want an opinionated plan format rather than free-form tasks. | Dogfood on one real Feature in this repo; ask two external early users to run `plan` on an approved Feature and compare to their current flow. |
| Should-be-true | Ordered tasks mapped to AC IDs matches how engineers actually decompose SpecScore Features. | Review the first three plan artifacts; check whether every AC has at least one task and whether task order survived implementation unchanged. |
| Might-be-true | A future `implement` skill will consume plan tasks one-at-a-time, so plan ordering must be execution-ready, not just human-readable. | Defer validation until `implement` is designed; design plan schema to allow it but don't optimize for it yet. |

## SpecScore Integration

- **New Features this would create:** TBD at spec time — likely one Feature covering the skill itself and one covering the Plan artifact schema and lint rules.
- **Existing Features affected:** None yet; `ideate` and `specify` Features are downstream-agnostic. Lint rule namespace (`P-xxx` for Plan) needs to be reserved.
- **Dependencies:** None blocking. Benefits from, but does not require, a `specscore new plan` CLI scaffold.

## Outstanding Questions

- Does the plan slug mirror the source Feature slug 1:1, or can one Feature have multiple Plans (e.g., alternative breakdowns)?
- Should `plan` emit `plan.drafted` / `plan.approved` events, or reuse the Feature's event stream?
- Does the Plan body metadata need an explicit `**Source Feature:**` line, mirroring `**Source Ideas:**` on Features, so Synchestra can auto-populate backlinks?
- Should lint enforce that every AC ID in the source Feature is covered by at least one task, or is that a soft warning?

---
*This document follows the https://specscore.md/idea-specification*
