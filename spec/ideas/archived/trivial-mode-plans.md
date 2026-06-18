# Idea: Trivial-mode plans: skeleton-up-front, journal-in-implement

**Status:** Stale
**Archived:** true
**Date:** 2026-05-19
**Owner:** alexander.trakhimenok
**Promotes To:** —
**Supersedes:** —
**Related Ideas:** extends:specstudio-implement-skill
**Archive Reason:** Merged into the `specstudio-implement-skill` Idea as the Phase-2 "living-plan posture" sub-direction. The plan-side schema concerns (placeholder body, `**Mode:**` metadata, lint rule) and the writeback contract (when `implement` rewrites task bodies, what event fires, concurrent-edit semantics) are fundamentally coupled to the `implement` skill's design and cannot be specified independently without drift. Captured there; this draft retained for traceability.

## Problem Statement

How might we honor the spec→plan→implement gate philosophy without forcing the plan to duplicate the implementation for trivially small Features?

## Context

The `specstudio:plan` skill produces an ordered task list where each task carries a 1–3 sentence body. For genuinely small Features — single AC, single file edit, a typo fix, a one-line config change — that task body ends up being a paraphrase of the diff that `implement` is about to produce. Users feel the duplication twice: once writing the plan, again writing the code. The complaint surfaced inside the SpecStudio repo itself: 'for simple changes the plan basically contains the content of new files or edits — looks like duplication of work.' The current gate philosophy is non-negotiable for good reasons (ideate, specify, and plan all enforce lint-clean artifacts before downstream skills can run), so the answer cannot be 'skip the plan.' But the schema currently demands the same level of task-body detail for a 5-line change as for a 500-line refactor. The opportunity: keep the gate, soften the schema for the small case.

## Recommended Direction

Add a 'skeleton plan' mode to `specstudio:plan` that produces a Plan artifact whose tasks carry only the mandatory frame — `### Task N: <name>` heading + `**Verifies:**` AC mapping line — with the body explicitly permitted to be a single placeholder marker (e.g. `<!-- implement: pending -->`). The skeleton plan is lint-clean by construction; a new lint rule (P-trivial-placeholder) only allows the placeholder while the Plan is `**Status:** Draft` AND a flag in the Plan body metadata marks it as trivial-mode. When `specstudio:implement` (whose design is still in flight) lands each task, it replaces the placeholder with a 1–2 sentence 'what landed' summary linked to the commit SHA and emits `plan.updated`. This treats the plan body content as a journal that accrues during implementation, not as a forecast written before any code exists.

Two trigger paths: (1) opt-in — the user passes `--trivial` to the skill invocation; (2) auto-suggest — `plan` detects a Feature with ≤2 acceptance criteria and asks the user 'this looks small enough for trivial-mode — proceed?' (single question, default Yes). Manual flag always wins; the threshold is a default, not a verdict.

This keeps the gate philosophy intact (a Plan artifact still exists and lints before `implement` can run; the AC-coverage contract is still enforced via `**Verifies:**` lines) while removing the only piece that was genuinely duplicative — the per-task narrative body. The narrative still gets written, but by the skill that already has the diff in hand.

## Alternatives Considered

- **Bypass plan entirely; implement writes the Plan as a post-hoc audit log.** Rejected. Inverts the gate from precondition to postcondition. SpecStudio's value is precisely that downstream skills can't run until the upstream artifact is lint-clean; reversing that for "small" cases creates two parallel mental models and a classification problem (when does post-hoc apply?) that costs more than the duplication it removes.
- **Always-stub plans: drop per-task bodies from the Plan schema for everyone.** Rejected. The narrative body is genuinely useful for medium-and-larger Features — it forces the planner to articulate the decomposition before code is written. Removing it across the board would lose the forcing function. Keep the full schema as the default; trivial-mode is the carve-out.
- **Auto-generate the entire Plan (headings, Verifies lines, bodies) from the source Feature's AC list.** Rejected for the MVP. Tempting, and probably correct for some future state, but it bundles two unrelated bets: "trivial Plans don't need pre-written bodies" (validated by the user's pain point) and "AC-to-task is a deterministic 1:1 mapping" (unvalidated; a single AC sometimes wants two tasks, two ACs sometimes share one task). Ship the first bet; revisit auto-generation as a follow-on.
- **Two-tier plan with a short-form schema (Summary + AC table only).** Rejected. Strictly weaker than skeleton-plus-journal: it still demands all the content up front, just less of it. Doesn't solve the duplication; just authorizes a thinner duplicate.

## MVP Scope

A two-week change to the `plan` skill in this repo: add the `--trivial` flag, the auto-suggest prompt, the placeholder body schema, and the corresponding lint rule. Ship it, then dogfood it on one Feature in this repo (a candidate: a future docs-only Feature). Stop there. No `implement`-side changes in the MVP — those land when the `implement` skill itself ships. While `implement` is unshipped, the user manually replaces placeholders before transitioning Plan status to Approved; lint enforces 'no placeholder bodies in Approved plans.' If the rule + flag together are more than ~150 lines of skill/lint code, the design is wrong.

## Not Doing (and Why)

- Skipping the Plan artifact entirely — breaks the gate philosophy that makes SpecStudio trustworthy; the duplication isn't fundamental, the schema is.
- Auto-generating task bodies from `git diff` projections — premature; let `implement` write the journal once it actually lands work, not a guessed forecast.
- A Feature-level 'trivial' flag in Feature body metadata — pollutes the Feature schema for a plan-side concern; classification belongs in `plan`.
- Generalizing the same skeleton pattern to ideate→specify — Features need real spec content even when small; different problem, different Idea.
- Heuristic / LLM-based complexity classification — deterministic AC-count threshold is auditable; classifier drift is not.
- Effort estimates, velocity, or scheduling metadata on trivial-mode tasks — out of scope; SpecScore is verifiable contracts, not project management.

## Key Assumptions to Validate

| Tier | Assumption | How to validate |
|------|------------|-----------------|
| Must-be-true | A non-trivial share of real Features routed through `specstudio:plan` are small enough that the per-task body is a paraphrase of the diff. If trivial-mode applies to <10% of Features, this is overengineering. | Tag the next 10 Features authored in this repo (or a dogfooding project) with `trivial / non-trivial` at plan time and measure. |
| Must-be-true | Keeping the Plan artifact + AC-coverage gate, while permitting a placeholder body, preserves the spec↔code coherence guarantees that justify the gate philosophy. | Walk the gate's stated job-to-be-done (verifiable handoff from spec to implementation) against the skeleton-mode behavior and confirm no guarantee is weakened. Have the consilium reviewer subagent assess this explicitly. |
| Should-be-true | "≤2 acceptance criteria" is a usable default threshold for auto-suggesting trivial-mode. Threshold should be configurable in `specscore.yaml`, not hard-coded. | After 5 dogfood plans, audit whether the suggestion fired when it should have and stayed silent when it shouldn't. Tune the default once; expose as `plan.trivial_ac_threshold` config key. |
| Should-be-true | Users prefer an explicit single-question prompt ("looks small enough — proceed in trivial mode?") over silent auto-application. | First three dogfood runs: explicit prompt. If users always say yes, consider promoting to default-on (with `--no-trivial` opt-out). If users say no even once, keep the prompt. |
| Might-be-true | The journal-during-implement pattern will generalize: trivial-mode task bodies → all task bodies → all Plan summaries → the Plan itself as something accrued, not authored. | Defer. Don't optimize the schema or events for this. Revisit once `specstudio:implement` has shipped and accumulated real journal entries. |


## SpecScore Integration

- **New Features this would create:** Likely one Feature on the `specstudio:plan` side covering the `--trivial` flag, auto-suggest prompt, placeholder schema, and metadata marker. Possibly a sibling Feature on the `specscore` CLI side reserving the new lint rule (e.g. `P-trivial-placeholder`) in the Plan rule namespace.
- **Existing Features affected:** The Feature(s) describing the `specstudio:plan` skill — schema for Plan body metadata gains an optional `**Mode:**` line; the `## Tasks` section gains a permitted placeholder body form. `specstudio:implement` (when designed) needs a "replace placeholder + emit `plan.updated`" contract baked in from day one.
- **Dependencies:** None blocking. The MVP can ship before `specstudio:implement` exists — placeholders just stay until the user manually replaces them, gated by "no placeholders in Approved plans." Benefits significantly from `implement` once that lands.

## Open Questions

- Is the trivial-mode marker a Plan body metadata line (`**Mode:** trivial`), a flag implicit in the placeholder presence, or both? Body metadata is more discoverable and lintable; implicit is less ceremony.
- What is the exact placeholder token? Candidates: `<!-- implement: pending -->` (HTML comment, machine-friendly, invisible in rendered markdown) vs. `**Implementation:** _pending_` (visible, scannable in a rendered Plan). Visibility trades off against "looks like a real, incomplete plan to a casual reader."
- Should the auto-suggest threshold (≤2 ACs) live in `specscore.yaml` under a `plan:` key from day one, or is hard-coded acceptable for the MVP with config introduced if dogfooding demands it?
- Without `specstudio:implement` shipped, how do users replace placeholders? Manual edit + re-lint is the obvious answer, but should `plan` itself gain a `plan complete-task <slug>` subcommand to make this more ergonomic, or is that scope creep?
- Does `plan.drafted` event payload need a `mode: trivial` field so Synchestra subscribers can filter (e.g., only the consilium reviewer cares about trivial vs non-trivial)?
- Should "trivial" be a one-way door (once a Plan is trivial-mode, it stays that way) or reversible (a Plan can be re-classified as full-mode mid-flight if scope grows)?

---
*This document follows the https://specscore.md/idea-specification*
