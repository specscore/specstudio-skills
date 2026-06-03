# Idea: SpecStudio recap skill

**Status:** Specified
**Date:** 2026-05-22
**Owner:** alex
**Promotes To:** skills/recap
**Supersedes:** —
**Related Ideas:** —

## Problem Statement

How might we close the loop between what a Feature specified and what an implementation actually delivered, so AC-level drift is named and decided on before review?

## Context

Recap is the next roadmap slot in the SpecStudio pipeline (init → ideate → specify → plan → implement → verify → **recap** → review → ship). The existing Draft Feature at spec/features/skills/recap/README.md names the intent but defers scope to this Idea. Verify is shipping (spec/ideas/specstudio-verify-skill.md is Implementing; spec/features/skills/verify/ is Approved) and produces a per-AC verdict report at spec/features/<slug>/_verify/<sha>.md with a grep-friendly YAML summary block — that report is recap's primary input alongside the Feature itself. Verify's transition-to-recap-only REQ already names recap as the sole permitted downstream skill, so the upstream handoff contract is fixed. The pipeline still cannot reach review honestly until something narrates what was actually built vs what was specified.

## Recommended Direction

Ship a minimum-viable recap skill that mirrors verify's structural shape: load an approved Feature, load the latest _verify/<sha>.md report at HEAD, and for each AC dispatch one AI subagent with (AC text, the verify verdict + justification + evidence, the commits the AC's Verifies: trailer points to, the diffs on demand). The subagent returns a drift classification ∈ {no-drift, spec-tighter-than-code, code-tighter-than-spec, contradiction} plus a ≤500-char narrative explaining the divergence. The skill aggregates per-AC drift verdicts into a Markdown report at spec/features/<slug>/_recap/<sha>.md with a grep-friendly YAML summary block at the top (parallel to verify), stages the report with git add, and exits non-zero only on contradiction verdicts (the other three are informational). Recap never edits the Feature, never auto-runs after verify, and only compares against the Feature at HEAD — every other shape is deferred. Pluggable drift detectors and Proposal-stub seeding (_proposals/<slug>.md) are explicitly out of scope: the same N=1 overfit trap verify's Idea named applies here, and proposal-stub seeding is a separable next Idea once recap is shipped and the human's drift-resolution workflow is observed.

## Alternatives Considered

- **V1 — Thin verdict re-presenter.** Recap is a `cat`-like merge of `_verify/<sha>.md` and the Feature: per-AC rows with `verdict | spec-snippet | evidence-snippet | drift-flag` where the drift flag is set mechanically (any AC where verdict ≠ pass, or where the verify subagent's justification contains drift keywords like "instead of", "modified to"). Lost because the value of recap is the spec-vs-code judgment, not the layout — keyword heuristics give a false sense of coverage and miss exactly the drift cases where the verify verdict was `pass` but the implementation took a different route than the AC named.
- **V2 — Risk-only recap.** Only emit rows for ACs with non-`pass` verify verdicts or flagged drift; skip clean passes. Lost because `review` and `ship` need a complete per-AC record to gate on, not just the exceptions — a report that omits clean rows can't serve as the honest gate the pipeline was designed around.
- **V3 — Plan-completeness recap (widened scope).** Add "did every Plan task land in a commit?" and "are there commits with `Verifies:` trailers for ACs the Plan didn't enumerate?" Lost because it widens scope past AC-level drift and reaches into Plan-vs-commits territory that's separable; a focused plan-recap Idea can land later if the need is real.
- **V4 — Conversational recap.** Walk through each AC interactively with the user. Lost on attention cost: recap should produce a reviewable artifact in one shot, then let the human decide; an interactive pass is the opposite of the "stage, never commit" discipline the rest of the pipeline uses.
- **V5 — Proposal-stub seeder.** V2's narrator output plus pre-staged `_proposals/<ac-slug>.md` stubs for any `spec-tighter` or `contradiction` drift item. Lost because the `_proposals/` artifact type isn't designed yet — recap shouldn't be the skill that smuggles a new artifact schema into the project. Defer to a follow-on Idea once recap is shipped and we observe how humans actually resolve drift items.

## MVP Scope

A two-week spike that ships skills/recap/SKILL.md plus spec/features/skills/recap/README.md fleshed out. The skill: (1) loads a Feature from spec/features/<slug>/, (2) reads the latest _verify/<sha>.md at HEAD, (3) for each AC walks git log --grep for the Verifies: trailer, (4) dispatches one subagent per AC with AC text + verify verdict + commits, (5) merges per-AC drift verdicts into a Markdown report with a YAML summary block, (6) writes the report to spec/features/<slug>/_recap/<sha>.md, (7) stages the report with git add, (8) exits non-zero on any contradiction. End-to-end dogfood: run recap on this repo's sidekick-capture Feature (the same one verify dogfooded against) and produce a real report.

## Not Doing (and Why)

- Auto-running after verify — out of scope; explicit invocation only (verify already recommends recap via transition-to-recap-only)
- Editing the Feature on the user's behalf — recap flags drift; humans author Feature edits
- Auto-drafting Proposals or seeding _proposals/<slug>.md stubs — separable next Idea once human drift-resolution workflow is observed
- Architectural or stylistic drift detection — AC-level only; style and architecture go to review
- Comparing against the Feature at Plan-approval time — HEAD only; multi-baseline diffing is overfitting before we know the single baseline is wrong
- Pluggable drift detectors or classification schemes — same N=1 overfit trap verify's Idea named; deferred to a dedicated future Idea
- Parallel subagent dispatch — match verify's serial discipline for MVP; revisit if real runtimes hurt
- Plan-completeness checks (every Plan task committed; unplanned trailers) — separable; widens scope past AC-level drift
- specscore.yaml recap: block — premature backend-naming schema
- Auto-promotion of drift items into the Feature's Open Questions section — recap stages a separate artifact; the human decides what to back-port

## Key Assumptions to Validate

| Tier | Assumption | How to validate |
|------|------------|-----------------|
| Must-be-true | An AI subagent given (AC text, verify verdict + justification, commits, diffs) can produce a useful drift classification ∈ {no-drift, spec-tighter-than-code, code-tighter-than-spec, contradiction} often enough to be useful. | Dogfood on this repo's `sidekick-capture` Feature (same one verify dogfooded against); count ACs where the subagent's drift verdict matches a human's read. Target ≥75% agreement on first pass. |
| Must-be-true | The 4-bucket drift taxonomy covers real drift cases without forcing a fifth bucket. | Hand-code the dogfood Feature with a human taxonomy first, then compare to the subagent. If humans reach for a 5th bucket, redesign the taxonomy before shipping. |
| Should-be-true | The verify report's evidence references + the AC's commit trailers are sufficient context for the subagent without recap re-fetching anything verify already saw. | Run with both prompt shapes (with/without re-passing verify's evidence) on dogfood and compare verdict stability. |
| Should-be-true | Recap exit-code semantics — non-zero only on `contradiction`, the other three informational — match how `review` and `ship` will want to gate. | Stub the eventual `review` skill's gate read; confirm it can grep recap's YAML block for `contradiction` count cleanly and that this matches the human's "would I block on this" intuition. |
| Might-be-true | Recap's output is short enough that human reviewers read the body, not just the YAML block. | Measure body length on dogfood; if any AC's narrative exceeds ~500 chars, tighten the subagent's word budget. |
| Might-be-true | The report path `spec/features/<slug>/_recap/<sha>.md` (parallel to `_verify/<sha>.md`) is the right location. | Decide at `specify` time; default to mirroring verify unless a reason emerges. |

## SpecScore Integration

- **New Features this would create:** `spec/features/skills/recap/` (currently a Draft placeholder) gets fully specified from this Idea.
- **Existing Features affected:** `spec/features/skills/verify/` — verify's `transition-to-recap-only` REQ already names recap as the sole permitted downstream skill; no spec change needed, behavior change only when recap ships. `spec/features/skills/review/` — currently a Draft placeholder; review's spec will consume recap's `_recap/<sha>.md` output once both skills exist.
- **Dependencies:** Verify is shipping (`spec/ideas/specstudio-verify-skill.md` Implementing). The `Verifies:` commit-trailer convention must hold (same dependency verify has). The `_verify/<sha>.md` report format (YAML head + body) must be stable enough for recap to parse.

## Open Questions

- Report path: `spec/features/<slug>/_recap/<sha>.md` (parallel to verify) vs. extending verify's report with a drift section? Leaning separate file so each stage produces its own evidence artifact, but worth confirming at `specify` time.
- Exit-code semantics: is `contradiction` the only blocking verdict, or should `code-tighter-than-spec` (implementation more permissive than spec named) also gate? Lean blocking on `contradiction` only for MVP — code-tighter is a real risk but not always a bug; revisit after dogfood.
- Should recap accept an explicit verify-report path argument, or always resolve the latest `_verify/<sha>.md` at HEAD? Lean auto-resolve for MVP; add a `--report` flag if a real workflow needs to recap against an older verify run.
- When the Feature has zero verify report (recap invoked before verify ran), refuse with a recommend-`verify`-first message, or run anyway with `verdict: unknown` rows? Lean refuse — recap without verify is a category error.

---
*This document follows the https://specscore.md/idea-specification*
