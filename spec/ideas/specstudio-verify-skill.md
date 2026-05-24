# Idea: SpecStudio verify skill

**Status:** Implementing
**Date:** 2026-05-22
**Owner:** alex
**Promotes To:** skills/verify
**Supersedes:** —
**Related Ideas:** —

## Problem Statement

How might we give every approved Feature a machine-checkable per-AC verdict so recap/review/ship have something honest to gate on, without locking SpecStudio to any one test framework?

## Context

verify is the next roadmap slot in the SpecStudio pipeline (init → ideate → specify → plan → implement → **verify** → recap → review → ship). The existing draft Feature at spec/features/skills/verify/README.md names the intent but defers scope to this Idea. implement already hands off to specstudio:verify via transition-to-verify and produces a Verifies: <feature-slug>#ac:<ac-slug> trailer on every commit — that trailer is the AC↔code linkage verify will consume. Today there is no Rehearse runtime in this repo: _tests/ directories hold Markdown G/W/T scenarios with no executor. The pipeline cannot close end-to-end until something produces per-AC verdicts.

## Recommended Direction

Ship a minimum-viable verify skill that closes the end-to-end pipeline with a single built-in AI-subagent verifier. The skill consumes an approved Feature, walks git log for each AC's Verifies: trailer to gather the commits + diffs that claim to satisfy it, dispatches a subagent per AC with (AC text, commit list, diffs), and renders a Markdown report at spec/features/<slug>/_verify/<sha>.md with a grep-friendly YAML summary block at the top listing each AC ID with verdict ∈ {pass, fail, unmapped, error}. The skill exits non-zero on any fail. Pluggable verifier backends are explicitly out of scope for this Idea — they belong to a future dedicated Idea designed against a real second backend (pytest adapter, Rehearse runtime, superpowers TDD). Designing the contract with N=1 implementation is a known trap. Defer.

## Alternatives Considered

- **V1 — Reviewer-shape contract + one verifier.** Define a stable JSONL stdin/stdout contract for verifiers in `skills/shared/` and ship one builtin that honors it. Lost because the contract would be overfit to the AI-subagent shape with N=1 implementation; a real second backend (pytest adapter, Rehearse runtime) is needed to design the contract honestly. Reopens as a dedicated Idea later.
- **V2 — Verdict bus (sink, not runner).** `verify` is a sink that aggregates verdicts emitted to `.specscore/verify-results.jsonl` by any external runner (CI, pytest, manual reviewer). Lost on MVP scope — execution-timing decoupling is real value but solves a problem we don't have yet, and adds a freshness/staleness model that doubles surface area.
- **V3 — Consilium-for-ACs.** Reuse the consilium 5-stage pipeline with a panel voting per AC. Lost on cost and weight — too expensive for a routine "did I land the feature" check, but viable as a *non-default* verifier once pluggability lands.
- **V4 — Whole-repo scoping (no trailer dependency).** Verifier sees the full working tree, no git-log walk. Lost because it discards the AC↔commit linkage `implement` already enforces and produces noisier verdicts. The trailer convention is the discipline; verify should reward it.

## MVP Scope

A two-week spike that ships skills/verify/SKILL.md plus spec/features/skills/verify/README.md fleshed out. The skill: (1) loads a Feature from spec/features/<slug>/, (2) walks git log --grep for each AC's Verifies: trailer, (3) dispatches one subagent per AC with AC text + commits + diffs, (4) merges per-AC verdicts into a Markdown report with a YAML summary block, (5) writes the report to spec/features/<slug>/_verify/<sha>.md, (6) stages the report with git add, (7) exits non-zero on any fail. No specscore.yaml verify: block. No JSONL contract in shared/. No pluggability. End-to-end dogfood: run verify on this repo's own sidekick-capture Feature and produce a real report.

## Not Doing (and Why)

- Pluggable verifier backends — deferred to a dedicated future Idea once a real second backend is on the table (designing the contract with N=1 implementation overfits to the AI-subagent shape)
- specscore.yaml verify: block — not added in MVP; would prematurely commit to a backend-naming schema
- JSONL stdin/stdout contract in shared/ — same overfit risk as above
- Rehearse executor — there is no Rehearse runtime in this repo today; _tests/ Markdown scenarios are read as evidence text by the subagent, not executed
- Whole-repo input scoping — trailer-driven scoping is tighter and leverages discipline implement already enforces; hand-edits without trailers being invisible to verify is correct behavior
- Single-AC isolated runs — MVP runs the full Feature; per-AC mode lands in a follow-on Idea if a real workflow needs it
- Drift detection against implement's claimed ACs — useful but separable; lands when pluggability lands

## Key Assumptions to Validate

| Tier | Assumption | How to validate |
|------|------------|-----------------|
| Must-be-true | An AI subagent given `(AC text, commits, diffs)` can produce a reliable `pass/fail/unmapped/error` verdict often enough to be useful. | Dogfood on this repo's `sidekick-capture` Feature: count ACs where the subagent's verdict matches a human's read. Target ≥80% agreement on first pass. |
| Must-be-true | `implement`'s `Verifies: <feature-slug>#ac:<ac-slug>` trailer convention is followed in practice for any Feature we'd verify. | Audit current commit history for Features in this repo (`sidekick-capture`, `sidekick-consilium`, `init`); confirm trailers exist and parse. If trailer hygiene is broken upstream, fix `implement` before `verify` ships. |
| Should-be-true | A Markdown report with a top-of-file YAML summary block is sufficient for `recap`/`review`/`ship` to gate on without a separate JSON file. | Stub `recap` to grep the YAML block; confirm it can extract per-AC verdicts cleanly. |
| Should-be-true | Two-week MVP timebox is realistic for one subagent-dispatching skill with no contract design and no pluggability. | Compare scope to `specstudio:implement` (which shipped) — both dispatch subagents per unit (task vs AC); `verify` has less orchestration overhead (no batches, no dependency graph). |
| Might-be-true | ACs without any matching `Verifies:` trailer should be reported as `unmapped` rather than `fail`. | Decide at spec time; defer to user-review of the Feature. |
| Might-be-true | The report path `spec/features/<slug>/_verify/<sha>.md` is the right location vs. `.specscore/verify/<feature>/<sha>.md` (tracked vs. untracked). | Decide at spec time. Tracked makes reports reviewable in PRs; untracked keeps churn out of git history. |


## SpecScore Integration

- **New Features this would create:** `spec/features/skills/verify/` (currently a Draft placeholder) gets fully specified from this Idea.
- **Existing Features affected:** `spec/features/skills/implement/` — the existing `transition-to-verify` REQ becomes a real handoff once `verify` ships. No spec change needed; behavior change only.
- **Dependencies:** `implement` shipped (✅). The `Verifies:` commit-trailer convention must hold in practice.

## Open Questions

- Report location: tracked (`spec/features/<slug>/_verify/<sha>.md`) vs. untracked (`.specscore/verify/<feature>/<sha>.md`)? Tracked makes reports PR-reviewable; untracked keeps git history quiet. Decide at `specify` time.
- ACs with no matching `Verifies:` trailer: report as `unmapped` (info) or `fail` (gating)? Leaning toward `unmapped` so `verify` is honest about scope, but `ship` should treat `unmapped` as blocking.
- What does the skill do when run on a Feature whose source code has no commits yet (just spec)? Likely: every AC is `unmapped`; report is still generated; exit non-zero. Confirm at `specify` time.
- Subagent dispatch: serial vs. parallel per AC? Implement's parallel batch model is a reasonable precedent, but `verify` has no inter-AC dependencies — full parallel is the obvious default. Decide at `plan` time.

---
*This document follows the https://specscore.md/idea-specification*
