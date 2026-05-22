# Idea: Reviewer Gates

**Status:** Implementing
**Date:** 2026-05-22
**Owner:** alex
**Promotes To:** reviewer-gates
**Supersedes:** —
**Related Ideas:** —

## Problem Statement

How might we make every review step in the SpecStudio pipeline unambiguous about who reviews, what they review, and whether the verdict blocks shipping?

## Context

Today the pipeline (`init → ideate → specify → plan → implement → verify → recap → review → ship`) ends with a generic `review` step whose [draft Feature](../features/skills/review/README.md) explicitly defers naming (human vs AI), placement (one step or per-artifact), and gate semantics (findings or verdict) to ideation. Meanwhile, [`third-party-integration`](../features/third-party-integration/README.md) (Approved) already pins a `reviewers:` registry in `specscore.yaml`, and both [`specify`](../features/skills/specify/README.md) and [`plan`](../features/skills/plan/README.md) already dispatch a built-in reviewer subagent before a separate "User Review Gate." The mechanism exists but is described inconsistently across three Features and is invisible from the root `README.md`. Two sidekick seeds ([`future-review-skill-could-discover-available-claude-code`](seeds/future-review-skill-could-discover-available-claude-code.md), [`extend-consilium-to-review-regular-specscore-ideas-not-just`](seeds/extend-consilium-to-review-regular-specscore-ideas-not-just.md)) hint at the same direction. This Idea consolidates those threads.

## Recommended Direction

Evolve the existing `reviewers:` registry into a typed, per-stage gate model. Three deltas: (1) add a `type:` discriminator on each reviewer entry (`ai`, `human`; extensible to e.g. `lint`, `security`, `ux`, `peer-review-bot`, `third-party-ci`); (2) allow type-specific fields per reviewer — `model:` and `prompt:` for `ai`, `min_approvers:` for `human`, etc.; (3) introduce a `gates:` block in `specscore.yaml` that scopes reviewer lists per skill (`gates: specify: reviewers: [...]`). Collapse the "User Review Gate" — the human is one `type: human` entry inside the gate's reviewer list, not a separate downstream step. Composition stays **AND** (any `Issues Found` blocks); verdict contract `Approved | Issues Found` stays; reviewers MUST NOT write to `spec/` stays. The standalone `review` pipeline step is **removed** — reviews are stage-internal and run as part of each producer's exit. The pipeline reduces to `init → ideate → specify → plan → implement → verify → recap → ship` with every producer's gate doing its own typed reviews. This supersedes the `reviewers:` parts of `third-party-integration` cleanly with no deprecation window (pre-launch). The reviewer-gate model is surfaced from the root `README.md` and cross-linked from every consuming skill's docs so adopters discover it without having to spelunk.

## Alternatives Considered

1. **Two distinct named pipeline steps (`audit` + `signoff`)** — keep the linear pipeline shape; replace today's `review` with an AI `audit` step plus a human `signoff` step before `ship`. **Lost** because it still hardcodes the reviewer set per step (no path to add a future security or UX reviewer without editing skill code), it duplicates the implicit human-approval gates already baked into every producer, and it doesn't reuse the existing `reviewers:` registry mechanism that `third-party-integration` already pins.
2. **Typed reviewers only at the final gate** — keep `review` as a single named pipeline step that internally runs a typed reviewer list, but leave `specify` and `plan` with their current built-in baseline reviewer + ad-hoc User Review Gate. **Lost** because the same primitive obviously belongs at every producer's exit — `specify` and `plan` already replicate reviewer-subagent gate logic, proving the abstraction is general — and centralizing it at one stage forfeits most of the value.
3. **Additive contract revision to `third-party-integration` (no supersede)** — treat the new `type:` field and `gates:` block as additive entries to the existing `reviewers:` schema. **Lost** because (a) the user explicitly chose supersede — we're pre-launch and the 30-day deprecation overhead isn't justified; (b) `gates:` is a structural shape change (per-stage segmentation), not a field addition, so claiming it's additive would be dishonest; (c) preserving two parallel shapes for backwards compat creates ongoing migration noise we don't need.

## MVP Scope

A focused spike that pins the new schema and wires only `specstudio:specify` to use it; the human-reviewer entry replaces today's User Review Gate. Specifically: (1) schema — `gates: <skill>: reviewers: [...]` in `specscore.yaml` with `type` discriminator and type-specific fields for `ai` and `human` (`lint` reviewer type deferred per Outstanding Questions); (2) semantics — AND composition unchanged, verdict contract unchanged, `reviewers MUST NOT write to spec/` unchanged, reviewers dispatched serially in registry order; (3) wiring — `specstudio:specify` reads `gates.specify.reviewers`, dispatches each entry, and a `type: human` entry IS the user-approval gate (no separate User Review Gate step); (4) visibility — linked from root `README.md`, from `spec/features/skills/specify/README.md`, from `skills/specify/SKILL.md`, and from `spec/features/README.md`. Pre-launch supersede: no `gates:` block in `specscore.yaml` means the consuming skill refuses to run with a clear error pointing at this Idea — no silent fallback to the old built-in baseline. Wiring `plan`, `implement`, `verify`, `recap` is out of scope here (see Not Doing).

## Not Doing (and Why)

- Wiring plan, implement, verify, recap into reviewer-gates — follow-on Ideas; each has its own status-transition and event-emission wrinkles that deserve dedicated specification
- Auto-skip rule where a human reviewer is auto-passed when preceding ai/lint reviewers return Approved — deferred until at least one gate runs in real dogfood and we have signal on false-positive risk
- Parallel reviewer dispatch — MVP is serial, mirroring verify's MVP discipline; parallelism lands in a follow-on Idea once typical reviewer counts and token-burst behavior are observed
- Skill-discovery for reviewer entries (the future-review-skill-could-discover-available-claude-code seed) — separable; the entry shape can grow a skill: field additively later
- Reviewer prompt-pack publishing/versioning — prompts stay repo-local for MVP; cross-repo prompt distribution is a separate concern
- Lint as a first-class reviewer type — specscore spec lint already runs as a discrete pre-reviewer step; modeling it as a reviewer entry is redundant for MVP and is captured under Outstanding Questions
- The standalone review pipeline-step Feature (spec/features/skills/review/README.md) — slated for archival on this Idea's approval rather than active redesign; no replacement skill is needed because reviews are stage-internal

## Key Assumptions to Validate

| Tier | Assumption | How to validate |
|------|------------|-----------------|
| Must-be-true | The `(ai, human)` typology (extensible to `lint`, `security`, `ux`, `peer-review-bot`, `third-party-ci`) generalizes without leaking through the abstraction. | At specify time, draft each of those 6 reviewer types as a concrete `gates:` entry. If any type requires bespoke wiring outside the `type` + type-specific-fields shape, the abstraction is overfit and the MVP scope must shrink. |
| Must-be-true | The `reviewers MUST NOT write to spec/` invariant survives the introduction of `type: human`. | Trace the `type: human` flow: it triggers an approval-phrase wait, not an artifact write. Confirm the existing approval-phrase recognizer (used by `ideate`/`specify`/`plan`) is the exact mechanism. |
| Should-be-true | `specscore.yaml` is the right config home (not a new file, not per-skill defaults, not `synchestra.yaml`). | Re-read existing `reviewers:` (third-party-integration) and `consilium:` precedent in the SpecScore Repo Config Feature; confirm `unknown-fields-preserved` covers a new top-level `gates:` block. |
| Should-be-true | Per-stage segmentation under `gates:` is more readable than a flat `reviewers:` with `applies_to:` scoping. | At specify time, write the same three-reviewer example in both shapes side-by-side; pick the one that reads cleaner with a 5-second scan. |
| Should-be-true | Pre-launch supersede with no `gates:` fallback is acceptable for adopters who haven't yet wired anything. | This repo has no `gates:` block today and no real adopters; verify before approval that no other consumer in the SpecScore org depends on the old flat `reviewers:` schema. |
| Might-be-true | Serial reviewer dispatch is adequate for typical reviewer counts (≤4 per gate). | Post-MVP: measure real run-time on a Feature with 3 ai + 1 human reviewer; if cumulative latency dominates the gate, follow-on Idea for parallel dispatch. |


## SpecScore Integration

- **New Features this would create:** `reviewer-gates` (schema + semantics + dispatch contract — supersedes the `reviewers:` parts of `third-party-integration`), `specify-uses-reviewer-gates` (wires `specstudio:specify` to consume the new schema; supersedes `reviewer-subagent-required`, `reviewer-baseline-blockers`, `reviewer-extension-hook`, `reviewer-composition`, and `user-approval-required` REQs in the [`specify`](../features/skills/specify/README.md) Feature).
- **Existing Features affected:**
  - [`third-party-integration`](../features/third-party-integration/README.md) — the `reviewers:` registry parts are superseded; the Producer and Capability shapes remain. The Feature is either narrowed in place (Producer + Capability only) or split into a sibling Feature, decided at specify time.
  - [`specify`](../features/skills/specify/README.md) — five REQs superseded; the User Review Gate section is removed in favor of `type: human` reviewer dispatch.
  - [`plan`](../features/skills/plan/README.md) — same shape as `specify`; explicit wiring deferred to a follow-on Idea.
  - [`review` (Draft)](../features/skills/review/README.md) — slated for archival on this Idea's approval; no replacement skill needed because reviews become stage-internal.
- **Dependencies:** SpecScore Repo Config's `unknown-fields-preserved` requirement (relied on; no change). `specstudio:specify` already ships with reviewer-dispatch logic; wiring is a rewrite of internals, not a greenfield build.

## Open Questions

- Default `type:` for any legacy `reviewers:` entry encountered during the supersede — assume `ai`, or refuse to load and require explicit `type:`? (Lean toward refuse: pre-launch means we can demand explicit types and avoid silent ambiguity.)
- Whether `lint` is a useful reviewer type at MVP or redundant with the existing discrete `specscore spec lint` step. (Lean toward dropping it from MVP; revisit if a real reviewer-shaped lint pass surfaces.)
- Exact placement of the visibility link in root `README.md` — under "What's in the box," under a new "How reviews work" section, or both. Resolved at specify time.
- Whether `type: human` accepts `min_approvers > 1` at MVP or pins `min_approvers: 1` until a real multi-approver workflow is requested. (Lean toward pin to 1 for MVP; schema keeps the field optional and forward-compatible.)
- How `gates:` entries identify *which* `ideate`/`specify`/`plan` skill they belong to when the plugin namespace grows (e.g., `gates.specify` vs. `gates.specstudio:specify`). Resolved at specify time when the wiring is pinned.

---
*This document follows the https://specscore.md/idea-specification*
