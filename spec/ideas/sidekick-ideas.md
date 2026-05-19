# Idea: Sidekick Ideas

**Status:** Implementing
**Date:** 2026-05-18
**Owner:** alexandertrakhimenok
**Promotes To:** sidekick-capture
**Supersedes:** —
**Related Ideas:** —

## Problem Statement

How might we capture promising side-ideas during focused work without derailing the host task, and triage them through a deterministic expert panel that can auto-promote unanimous low-cost wins?

## Context

While running focused work in `specstudio:ideate`, `specstudio:specify`, or `agent-skills:build`, the host agent regularly notices tangential improvement ideas — refactors, missing tests, adjacent features, UX wins — that are out of scope for the current task. Today these get dropped: either the agent derails to chase them, the user is interrupted to triage them, or they are forgotten. None of those outcomes is good.

This repo already provides the artifact substrate for capture (`spec/ideas/` + `specscore:idea` scaffolder + lint), the queue substrate (`synchestra:task` lifecycle + events), and the prioritization layer (`synchestra:whats-next`). What is missing is (a) a discipline for the host agent to *write-and-keep-going*, (b) a deterministic deliberation step that turns captured seeds into actionable verdicts, and (c) a strict, auditable gate that auto-promotes the strongest seeds into Feature specs and plans without ever auto-generating code.

Prior art: multi-agent debate frameworks (CAMEL, ChatDev, MetaGPT, AutoGen GroupChat) cover deliberation but lack the opportunistic-capture front end; AutoGPT-style task spawning covers mid-flow capture but lacks expert review and a human-safe gate; IDE "suggestion" sidebars (Cursor, Copilot Workspace) are single-model and do not deliberate. The novel combination here is *opportunistic capture + deterministic expert panel + auto-promotion ceiling that stops short of code*.

## Recommended Direction

A three-layer system where **capture is woven into existing skills**, **Synchestra owns the queue and verdict as system-of-record**, and **a worker runtime drains the queue and applies a strict deterministic gate**. Each layer is independently useful; together they form the full loop.

**Layer 1 — Capture (woven into host skills).** A shared directive in `skills/shared/` instructs `ideate`, `specify`, and `build` to watch for out-of-scope improvements. The host agent supports both *heuristic capture* (it spots cues like "would be nice if…", "another approach is…") and *explicit capture* (`/sidekick <one-liner>`). Either path writes a lightweight seed at `spec/ideas/seeds/<slug>.md` and emits a `sidekick-idea.captured` Synchestra event. The host agent then keeps going on its original task — the discipline of *write-and-continue* is the whole point.

**Layer 2 — Synchestra orchestration (queue + audit trail).** The captured event creates a `consilium-review` task in the Synchestra task board, idempotent on the content-hash of the seed's one-liner. The seed artifact and the verdict-bearing task together are the durable record. `synchestra:whats-next` can already see and prioritize these. Verdicts persist across sessions, machines, and teammates.

**Layer 3 — Consilium worker (deliberation + strict gate).** A new `synchestra:consilium` skill drains pending tasks through a five-stage pipeline:

1. **CLI gather (deterministic).** The Synchestra CLI runs graph queries — `synchestra:feature` for related features, `synchestra:code` for code-to-spec refs, recent git log over relevant paths, prior-seed lookup over the dedupe window — and assembles a raw context bundle.
2. **Researcher agent (one LLM call).** The researcher reads the raw bundle and produces a **briefing pack**: a structured *fact-only* document (no scoring, no judgments) of related artifacts, prior decisions, and relevant code locations. The briefing is attached to the seed and becomes part of the audit trail. Its job is to ensure all experts reason over the same evidence and to eliminate duplicated repo-crawling across the panel.
3. **Expert panel (9 parallel role agents).** A fixed default panel — 3 Builders (Engineer, Architect, QA), 3 Customers (PM, UX, Marketing), 3 Adversaries (YAGNI Cop, Skeptic, optional Security/Ops) — each receives the seed + the briefing pack as their shared base context and returns a structured YAML vote. Every expert retains read-access tools and may do role-specific deeper research when needed: the briefing is a floor, not a ceiling. This is what prevents the panel from pre-converging on a researcher blind spot, while still capturing the token win for the (common) case where the briefing is sufficient.
4. **CLI arbiter (deterministic).** The Synchestra CLI validates votes against schema, applies the adversary-veto rule and the builder+customer-consensus rule, sets the verdict, transitions task status, and emits the `sidekick-idea.reviewed` event.
5. **Scribe agent (one LLM call, advisory only).** The scribe synthesizes a *Panel summary* paragraph in the appropriate flavor — "why the panel converged", "why the panel rejected", or "where the panel split" — and the CLI appends it to the seed. The scribe cannot change the verdict; by contract the CLI ignores any verdict field it emits.

Each expert returns one of four verdicts: `should-implement`, `should-not-implement`, `no-opinion`, or `abstain` — the latter for "not in my wheelhouse." Abstain carries a confidence: a *high-confidence* abstain ("clearly not my domain") excludes the expert from the consensus denominator; a *low-confidence* abstain ("I can't tell whether this matters to me") signals panel confusion and caps the overall verdict at `needs-human-review`. This preserves the safety property of a constant-panel design (Marketing still *reads* every seed and can catch an "internal refactor" that secretly touches the public API) while giving pure-technical ideas a path through the gate without forcing customer-side experts to vote on things outside their domain.

When the strict gate passes (`all non-abstain Builders + all non-abstain Customers approve + no adversary veto + no low-confidence abstain + at least one non-abstain Customer voice (or explicit builder acknowledgement of zero customer surface) + median confidence ≥ medium across non-abstain votes + cost & complexity ≤ 🟡`), the worker auto-invokes `specstudio:specify` and `specstudio:plan` to draft the Feature spec and implementation plan — and stops. Code is never written autonomously.

The reason the CLI-vs-LLM split is non-negotiable at *both* ends: a gating system whose verdict is produced by an LLM is non-deterministic, non-auditable, and tends to rubber-stamp the majority; and a research step that mixes facts with judgment quietly pre-converges every expert that reads it. Deterministic rules over typed inputs are reproducible, unit-testable, and tunable per-project via `synchestra.yaml`. LLMs are good at synthesis (the researcher and scribe jobs); they are bad at gating (the arbiter job) and they are bad at fact-gathering when judgment leaks in. The pipeline is symmetric on purpose: CLI on the rails at both ends, one LLM synthesis call bracketing each end of the panel.

## Alternatives Considered

**Hook-only, no Synchestra.** Capture into `spec/ideas/seeds/` and run the panel via a Claude Code `Stop` hook with no Synchestra task type. *Lost because:* verdicts end up scattered across loose markdown, the queue is implicit, dedupe and prioritization have to be re-implemented locally, and the system only works inside a Claude Code session. Ships in a weekend but rebuilds a task queue badly.

**Synchestra-only, no hook ergonomics.** Capture + queue + manual `/consilium` invocation. *Not rejected — adopted as Phase 1.* The hook is correctly a Phase-3 ergonomic layer on top of a durable core, not part of the MVP.

**LLM arbiter agent.** A single arbiter reads all 9 votes and produces the verdict. *Lost because:* non-deterministic verdict gating poisons everything that depends on it (auto-promote, audit trail, per-project tuning, unit tests). Arbiter LLMs also reliably echo the majority — they add tokens without adding judgment. Kept the LLM for the synthesis job (the scribe) where it is genuinely good, and gave the deterministic gate to the CLI.

**Auto-build a prototype on unanimous approval.** The original framing of the idea. *Lost because:* even unanimous AI agreement is still AI agreement. Token cost, repo cleanup risk, and the chance of half-baked code landing without a human ever seeing the Feature are all real. The Phase-2 ceiling stops at *Feature spec draft + implementation plan*, both of which a human reviews before `build` runs.

**Numerical 1–9 scoring.** Looks precise, isn't. *Lost because:* unanchored numeric scales aren't calibrated across roles or runs. Replaced with a 3-step ordinal (🟢/🟡/🔴) plus a required *Confidence* level (low/med/high), a four-option *Verdict* (`should-implement` / `should-not-implement` / `no-opinion` / `abstain`), and a one-sentence *Strongest argument* per expert.

**Dynamic panel composition per idea.** A pre-panel classifier picks which experts review each seed based on inferred relevance (e.g., skip Marketing on internal refactors). *Lost because:* the cases where Marketing would be wrongly skipped are exactly the cases the panel needs to catch — an "internal refactor" that secretly changes a public API contract is the precise failure mode that constant-panel-with-abstain prevents. Relevance is expressed by experts themselves via `abstain`, not by an upstream classifier; the panel composition stays fixed per project.

## MVP Scope

The MVP nails one job: **a user mid-flow in `specstudio:specify` (or any host skill) can drop a seed without losing focus, manually invoke `/consilium` later, and read a deterministic verdict plus a useful Panel summary on the seed.** Auto-promotion and hook ergonomics are out of MVP scope.

Concretely, the MVP is Phases 0 + 1 of the design:

- **Phase 0 — Capture.** Shared capture directive woven into `ideate`, `specify`, and `build`. Seed artifact format defined and lint-validated. Both heuristic and explicit (`/sidekick`) capture paths working. Seeds pile up usefully in `spec/ideas/seeds/` even with no panel running — the system is a notebook before it is a court.
- **Phase 1 — Manual consilium.** `consilium-review` task type added to Synchestra with idempotent dedupe. `synchestra:consilium` skill that, when invoked, claims one or more pending tasks and runs the full five-stage pipeline: CLI gather → researcher (briefing pack) → 9-role panel in parallel → CLI arbiter → scribe. Writes the briefing, verdict, and Panel summary back to the seed and task. No auto-promote yet — the verdict is the deliverable.

MVP success is measured by running the consilium on 5–10 real seeds captured during normal work and confirming: (a) the host agents did not derail during capture, (b) the panel produced verdicts that the user would have made themselves with the same information, (c) the Panel summary is the part of the seed the user actually reads.

## Not Doing (and Why)

- Live multi-agent debate UI — agents return YAML in parallel, no chat theatre
- LLM arbiter that decides the verdict — verdict gating must stay deterministic
- Autonomous code generation on auto-promote — Phase 2 ceiling stops at draft Feature + plan
- Cross-project consilium memory — each project has its own queue and verdicts
- Per-role model selection — all roles share one model in MVP
- Dynamic panel composition per idea — every expert always reads every seed; relevance is expressed via `abstain`, not by an upstream classifier skipping experts

## Key Assumptions to Validate

| Tier | Assumption | How to validate |
|------|------------|-----------------|
| Must-be-true | Host agents can capture sideline ideas without derailing their primary task. The discipline of "write seed and continue" survives in practice. | Run `ideate` and `specify` sessions with the capture directive enabled; inspect transcripts and outputs against runs without it. Primary outputs must be indistinguishable; seeds must appear at moments a human reviewer agrees are reasonable. |
| Must-be-true | Adversarial roles (Skeptic, YAGNI Cop) genuinely dissent rather than echo majority sentiment when the seed has real weaknesses. | Construct a calibration set of 10 seeds with known weaknesses (out-of-scope, low ROI, premature, duplicate). Adversaries must surface those weaknesses in ≥ 80% of cases with high confidence. If they don't, role prompts are inadequate. |
| Must-be-true | Deterministic verdict gating in the CLI is reproducible and unit-testable. | Snapshot tests over a fixture set of 9-vote inputs. Same inputs must always produce same verdict, status transition, and emitted event. |
| Should-be-true | The full pipeline (researcher + 9-role panel + scribe) produces a useful verdict within a reasonable token budget (target: ≤ 30K tokens per review; rough plan: ~5K researcher + 9 × ~2K experts + ~1K scribe ≈ 24K). | Measure actual token cost across 20 reviews; if median exceeds budget, tighten the briefing pack template, trim default roster, or shorten role prompts. |
| Should-be-true | The researcher producing a shared *fact-only* briefing meaningfully reduces panel-wide token use without pre-converging the verdict. | A/B comparison on a fixture seed set: panel with vs. without briefing. Measure (a) total tokens consumed; (b) verdict agreement rate. Briefing wins if (a) drops materially while (b) does not drift toward false consensus. If the briefing-fed panel converges *too* much (loss of dissent in adversary roles), the researcher is over-summarizing. |
| Should-be-true | The unanimous-Builders + unanimous-Customers + no-adversary-veto gate is strict enough that Phase-2 auto-promote produces Features the user would have approved manually. | Run Phase 1 in shadow mode for 20+ verdicts; for each `should-implement` verdict, ask the user post-hoc whether they would have approved auto-promotion. Target: ≥ 95% agreement before flipping `auto_promote: true`. |
| Should-be-true | The scribe agent's *Panel summary* paragraph is the part of the seed users actually read and trust. | After 10 manual consilium runs, ask the user which fields they referenced when deciding what to do with the seed. If the summary is skipped in favor of raw vote tables, the scribe is failing. |
| Should-be-true | `abstain` is used as a discriminating signal — i.e., experts abstain on out-of-domain seeds (high confidence) and engage on in-domain ones — rather than becoming a lazy default to dodge committing to a verdict. | Tag each abstain in a calibration set of 15 seeds: was it correct (verifiably out-of-domain) or lazy (within domain but expert ducked)? Lazy rate must stay under 10%, or role prompts need tightening. Pair: at least 60% of seeds must have ≥ 1 substantive (non-abstain) Customer vote — too many all-Customer-abstain seeds signals the panel is under-engaging. |
| Might-be-true | Heuristic capture (no explicit `/sidekick`) produces a signal-to-noise ratio better than 1 useful seed per 5 captures. | Tag each heuristic-captured seed as `useful` / `noise` after one week of work. Below threshold, fall back to explicit-only capture in defaults. |
| Might-be-true | A 30-day content-hash dedupe window catches re-captures without false-positive collapsing of distinct ideas. | Inspect collisions across the first 30 days; tune window if either too-coarse or too-permissive. |


## SpecScore Integration

- **New Features this would create:**
  - `skills/shared/sidekick-capture` — the capture directive woven into host skills (heuristic + explicit `/sidekick` paths, write-and-continue discipline).
  - `spec/ideas/seeds/` artifact format — lightweight seed schema, lint rule, frontmatter contract.
  - `synchestra/task-types/consilium-review` — the queue resource type, idempotency rule, lifecycle hooks.
  - `skills/consilium` — the worker skill orchestrating the five-stage pipeline (CLI gather → researcher → panel → CLI arbiter → scribe). Roster config, parallel role-agent fan-out, briefing/verdict/summary write-back. Roles live as markdown under `skills/consilium/roles/<role>.md`.
  - `skills/consilium/researcher` — the pre-panel researcher: reads the CLI-gathered raw bundle, emits a fact-only briefing pack. Prompt template lives alongside the skill. Scribe and researcher prompts share a *no-judgment* contract.
  - `cli/consilium/gather` — deterministic context-bundle assembly: calls `synchestra:feature`, `synchestra:code`, git log over relevant paths, and dedupe-window seed lookup. Output schema is versioned so the researcher prompt is stable.
  - `cli/consilium/verdict` — the deterministic CLI arbiter (schema validation + veto rule + consensus rule + verdict emission). Lives in the SpecScore CLI or a sibling, with unit-test fixtures.
  - `skills/consilium/auto-promote` (Phase 2) — the strict-gate promotion chain that invokes `specstudio:specify` and `specstudio:plan`.
  - `hooks/consilium-drain` (Phase 3) — Claude Code `Stop` hook or scheduled `loop` that auto-invokes `synchestra:consilium`.
- **Existing Features affected:**
  - `skills/ideate`, `skills/specify` (specstudio) — gain the capture directive; primary outputs unchanged.
  - `agent-skills:build` and similar host skills — same capture directive added.
  - `synchestra:whats-next` — must learn to surface `consilium-review` tasks and seeds with `needs-human-review` verdicts.
  - `specscore spec lint` — extended with the seed-schema lint rule.
- **Dependencies:**
  - Synchestra task system and event bus (already present).
  - `specscore` CLI (idea/feature/lint already present; `consilium verdict` is new).
  - Claude Code `Agent` tool with `run_in_background` and parallel-tool-use support (already available).

## Outstanding Questions

- **Seed → Idea promotion path.** Do approved seeds flow `seed → spec/ideas/<slug>.md (via specstudio:ideate) → Feature`, or `seed → Feature` directly when the verdict is strong enough? The first preserves the refinement step; the second is faster. Likely answer: strong-verdict seeds skip the Idea step; weak-but-interesting seeds promote into Ideas for further ideation.
- **Roster override surface.** Per-project `specscore.yaml` is decided (the arbiter's CLI owns the config — see Phase 1 sub-Idea `sidekick-consilium`). Open: do we also allow per-invocation overrides (`/consilium --without security,ops`)? Pro: useful for cost control on small seeds. Con: undermines the "fixed default" guarantee.
- **Veto and abstain confidence thresholds.** Two related thresholds need calibration once the consilium is running. (a) Adversary veto: current rule is *high*-confidence `should-not-implement` triggers the block — should *medium* also block, or only flag? Conservative is medium-blocks; aggressive is high-only. (b) Abstain handling: current rule is *high*-confidence abstain excludes from denominator and *low*-confidence abstain caps verdict at `needs-human-review`. Medium-confidence abstain is currently treated as low (cautious default); validate whether that's too conservative in practice.
- **Model choice for adversaries.** Should adversarial roles run on a different model than builders/customers to break correlation? MVP says no (one model); validation may flip this.
- **Owner attribution.** Seeds captured heuristically by an agent — is `owner` the user running the session or the host skill that captured? Current draft assumes the user; debatable.
- **Notification surface.** Phase 3 hook drains the queue; how does the user *learn* a verdict is ready without polling? Synchestra event → terminal message? PushNotification? Out of MVP, but design space worth naming.
- **Lint-sync rule for source-artifact back-link drift.** The Phase 0 `sidekick-capture` Feature ships *immediate-write* of back-links (the skill updates the source artifact's `## Sidekick Seeds Generated` section at capture time) so reviewers see the list without waiting. Deferred: a `specscore spec lint --fix` rule that reconciles drift between `spec/ideas/seeds/` and source-artifact back-link sections (same pattern as ideas-index sync and `Promotes To` reverse-link sync). Cross-repo: extends the `specscore` CLI. Failed back-link writes during capture are reported as warnings; the lint sync would heal them automatically on the next pass.
- **Briefing-pack ceiling.** What is the right size cap for the researcher's briefing? Too short loses useful context; too long defeats the token win and risks experts treating it as authoritative rather than as a floor. First guess: ≤ 1,500 tokens of structured facts. Validate empirically.

---
*This document follows the https://specscore.md/idea-specification*
