# Idea: Sidekick Consilium

**Status:** Implementing
**Date:** 2026-05-18
**Owner:** alexandertrakhimenok
**Promotes To:** sidekick-consilium
**Supersedes:** —
**Related Ideas:** extends:sidekick-ideas

## Problem Statement

How might we drain captured sidekick seeds into deterministic, auditable verdicts — surfacing real disagreement, respecting domain relevance through abstain, and staying within token budget — without ever letting LLM judgment leak into the gate?

## Context

Phase 0 (`sidekick-capture`, Implementing) ships an idea-capture pipeline that durably records sideline ideas during host-skill work. Seeds queue up in `spec/ideas/seeds/` but Phase 0 stops at write-and-continue — there is no triage. Phase 1 is the consilium worker that drains the queue and produces deterministic verdicts so the queue stops being an unreviewed inbox. The parent [`sidekick-ideas`](sidekick-ideas.md) Idea already established the 5-stage pipeline architecture (CLI gather → researcher → 9-role panel → CLI arbiter → scribe) and the abstain-with-confidence semantics; this sub-Idea pins down the implementation-seam decisions specific to Phase 1.

## Recommended Direction

Ship Phase 1 as a `specstudio:consilium` skill that drains all queued seeds per invocation, runs the 5-stage pipeline per seed under a 25K-token budget (5K researcher + 9 × ~2K experts + 1K scribe), and writes the verdict to the orchestrator `consilium-review` task as the structured source of truth, with the scribe's prose summary mirrored into the seed for human discoverability. The deterministic arbiter — the rule engine that turns 9 typed votes into a verdict — lives as a new `specscore consilium verdict` subcommand in the cross-repo `specscore/specscore-cli` from day one, so the verdict logic gets unit-test fixtures and reuse across SpecScore projects.

Verdict storage is intentionally multi-consumer: the orchestrator task carries the full structured payload (all 9 votes with confidence, the briefing pack, the arbiter's rule trace, the content_hash) that machine consumers (`specscore:whats-next`, future Phase 2 auto-promote) query directly; the seed gets only the scribe's one-paragraph summary so a human browsing `spec/ideas/seeds/` sees the verdict alongside the captured one-liner without scrolling through 9 vote blocks.

Invocation is single-mode and simple: `/consilium` drains every queued task, emitting one verdict per seed. No per-slug invocation, no `--limit` knob, no concurrency beyond the parallel role-agent fan-out within a single seed's review. Calibration thresholds and ergonomic knobs are explicit Outstanding Questions to revisit after the first 20 calibration verdicts.

**Roster and verdict-gate are both per-project configurable** via `specscore.yaml` under a `consilium:` top-level block. Phase 1 ships two sub-blocks:

- `consilium.roster` — projects use the 9-role default out of the box, but the block can (a) `exclude` any default role by name and (b) `custom` add roles by pointing at a markdown definition file the project owns. Each custom role declares its group (`builders | customers | adversaries`) so the verdict-gate denominators recompute correctly.

- `consilium.gate` — the rule knobs the arbiter applies to turn 9 typed votes into a verdict. Configurable: `adversary_veto_confidence` (the confidence floor at which an adversary `should-not-implement` becomes a hard block), `cost_ceiling` and `complexity_ceiling` (🟢-only vs 🟢-or-🟡), `min_median_confidence` (the median confidence across non-abstain votes required for `should-implement`), and `require_all_builders` / `require_all_customers` (unanimous vs. supermajority thresholds). Defaults match the parent Idea's strict baseline; projects with different risk tolerance can loosen or tighten.

Validation lives in the `specscore consilium verdict` subcommand: the roster must produce at least one member per group after exclude/add, custom-role names must not collide with defaults, the total panel size must stay under a documented cap (recommended ≤ 12 roles total), and gate knobs must fall within their documented ranges. Per-invocation overrides (`/consilium --without security`) remain out of scope for Phase 1 — they're a parent-Idea OQ to revisit after calibration if users hit the friction. Phase 2 will add `consilium.auto_promote` to the same block (action knobs: which actions fire on `should-implement`, dry-run toggle, etc.) — Phase 1's schema is designed so this extension is non-disruptive.

## Alternatives Considered

**Inline verdict in the seed file (V1).** Append a full `## Consilium Verdict` section to `spec/ideas/seeds/<slug>.md` containing all 9 votes + the verdict + the scribe summary. Single source of truth in one file. *Lost because:* the seed file mutates with every review (and with every Phase 0 back-link append), losing the "seed is what the captor wrote" property; humans browsing seeds wade through 9 vote blocks before reaching the summary; machine consumers (`whats-next`, Phase 2) have to parse markdown instead of structured data. The multi-consumer answer the user gave makes the structured-task source-of-truth the right choice.

**Sibling verdict file (V2).** Write the verdict to `spec/ideas/seeds/<slug>.verdict.md` and leave the seed immutable. *Lost because:* doubles the file count per idea, makes the lint rule's job harder (now two artifact types per directory), and still leaves humans needing to context-switch between two files to understand a seed's state. The "task + seed-summary mirror" of V3 keeps each surface tuned to its consumer.

**3-role minimal panel.** One Builder + one Customer + one Adversary, ~12K tokens per review. *Lost because:* the parent Idea explicitly chose 9 roles for adversarial diversity and the safety property that the constant panel reads every seed. A 3-role panel can't surface intra-group disagreement (e.g., Engineer disagreeing with Architect on coupling) and gives the single Adversary disproportionate veto weight.

**Inline bash arbiter in the skill (V6).** Implement the verdict gate as deterministic bash (yq/jq) inside the `specstudio:consilium` skill; no cross-repo work. *Lost because:* the user chose V4 — the gate logic belongs in `specscore-cli` long-term, deserves unit-test fixtures, and benefits from being a reusable subcommand. The cross-repo coordination cost is accepted as the price of doing it right from day one rather than refactoring later.

## MVP Scope

A user with N queued seeds in `spec/ideas/seeds/` runs `/consilium` once, sees N verdicts in roughly N × ~30s, and reads scribe summaries appended to each seed. The CLI arbiter is in place. The panel uses the 9-role default roster, but a project that has authored `consilium.roster.exclude` and/or `consilium.roster.custom` in its `specscore.yaml` sees the configured roster honored (validated by the CLI). No auto-promote, no hooks, no notifications, no per-invocation overrides. Ship behind a quality gate of ≥ 95% post-hoc human agreement on a 20-verdict calibration set run against the default roster.

## Not Doing (and Why)

- Auto-promotion to Feature spec or plan — Phase 2 territory; verdict is the deliverable here
- Hook ergonomics (Stop hook, loop-based drain) — Phase 3
- Per-role model selection — all roles share one model in MVP; revisit if calibration shows adversaries correlate too much
- Per-invocation roster overrides (`/consilium --without security`) — parent-Idea OQ; revisit after calibration if users hit the friction
- Notification surface (Slack/email/webhook on verdict) — Phase 3 territory
- Dynamic panel composition per seed — already rejected in parent Idea in favor of abstain-with-confidence
- Cross-project consilium memory — each project has its own queue and verdicts; no shared learning
- Live multi-agent debate UI — agents return YAML in parallel; no chat theatre

## Key Assumptions to Validate

| Tier | Assumption | How to validate |
|------|------------|-----------------|
| Must-be-true | The 9-role panel + abstain produces verdicts the user would have made themselves with the same information. | Run the consilium on a 20-seed calibration set (5 known-strong, 5 known-weak, 5 known-out-of-domain, 5 known-ambiguous). After each verdict, capture the user's post-hoc opinion. Target: ≥ 95% agreement on `should-implement` and `should-not-implement` verdicts; tolerate disagreement on `needs-human-review` (those are correct by design). |
| Must-be-true | The deterministic arbiter is reproducible: same 9 votes always produce the same verdict, status transition, and emitted event. | Snapshot tests in `specscore-cli` over a fixture set of vote inputs covering each gate-rule branch (adversary veto, builder/customer consensus, abstain denominators, all-customer-abstain edge case). CI gates on snapshot-stable outputs. |
| Must-be-true | Adversary roles (Skeptic, YAGNI Cop, optional Security/Ops) genuinely dissent on weak seeds rather than echoing majority sentiment. | On the 20-seed calibration set, expect adversaries to flag the 5 known-weak seeds with high confidence ≥ 80% of the time. If they don't, role prompts are inadequate and need re-authoring before Phase 1 ships. |
| Should-be-true | The 25K-token-per-seed budget produces useful verdicts. | Measure actual token cost across the 20 calibration verdicts. If median exceeds 25K, tighten the briefing pack template or trim role prompts; if median is ≪ 25K, the budget has slack for future deeper briefings. |
| Should-be-true | The researcher's briefing pack meaningfully reduces panel-wide tokens without pre-converging the verdict. | A/B comparison on a 5-seed subset: panel with vs. without briefing. Measure (a) total tokens consumed, (b) verdict agreement rate. Briefing wins if (a) drops materially while (b) does not drift toward false consensus. If the briefing-fed panel converges *too* much, the researcher is over-summarizing — shorten the briefing template or expand to fact-only outline form. |
| Should-be-true | `abstain` is used as a discriminating signal — experts abstain on genuinely out-of-domain seeds (high confidence) and engage on in-domain ones — rather than becoming a lazy default. | On the 5-known-out-of-domain seeds in the calibration set, expect ≥ 1 high-confidence customer abstain. On the 15 in-domain seeds, expect lazy-abstain rate < 10%. Track per-role to spot specific roles ducking. |
| Should-be-true | The scribe's *Panel summary* paragraph (mirrored into the seed) is the field humans actually reference when deciding what to do with a verdict. | After the 20-seed calibration set, survey: "When you decided what to do with each seed, which field did you read first / read most?" If `## Consilium Verdict` summary is the answer < 50% of the time, the scribe is failing and needs a prompt rework. |
| Should-be-true | `specscore/specscore-cli` can ship the `specscore consilium verdict` subcommand in 2–4 weeks of cross-repo coordination after this Idea promotes to a Feature. | Open a parallel issue / Feature in `specscore-cli` at promotion time. Track via the companion-plan pattern established for `seed-lint-rule` (issue #6). |
| Should-be-true | Custom-role markdown definitions can conform to the same role-output YAML schema the default roles use, with only the role's prompt body varying. | A test project authors a custom role (e.g., a domain-specific "Accessibility" role), runs `/consilium` against a calibration seed, verifies the custom role's vote parses identically to a default role's vote. If the schema requires per-role specialization, escalate to a richer role-contract spec before Phase 1 ships. |
| Should-be-true | The roster-validation rule (≥ 1 member per group after exclude/add; ≤ 12 total) is the right ceiling to keep panel cost bounded without blocking reasonable customization. | Track configured rosters across the first 3 adopter projects: median size, frequency of `exclude` use, frequency of `custom` use. Adjust the cap if it becomes a friction point. |
| Should-be-true | The 20-verdict calibration set (≥ 95% post-hoc human agreement) is meaningful when run against the *default* roster + *default* gate knobs, even though projects can override both. | The calibration target is baseline-only. Projects that override roster or gate knobs accept that their own calibration is their responsibility; document this explicitly in the Feature spec. If users routinely override before calibrating, the baseline calibration loses signal — track override-before-calibration as an anti-pattern. |
| Should-be-true | The gate knobs (adversary veto confidence, cost/complexity ceiling, median confidence, builder/customer consensus) are the right set — not too few, not too many. | Survey adopter projects after 3 months: which knobs did they touch, which did they want but didn't have, which were noise. Trim or extend the knob set additively based on signal. |
| Might-be-true | A 20-verdict calibration set is enough to validate the gate before declaring Phase 1 ready. | If the first 20 reviews surface a gate-rule edge case not covered (e.g., a verdict pattern that feels wrong but the rules approve), extend the calibration to 40. Don't ship Phase 2 (auto-promote) until calibration is solid. |
| Might-be-true | Per-project roster overrides (`/consilium --without security`) won't be needed for the first 3 months of use. | Track friction signal in calibration sessions: does any user manually want to skip a role for a given seed? If > 3 such requests in the first 20 verdicts, surface the override design earlier. |
| Might-be-true | The "all customers abstain" edge case will be common enough on technical refactor seeds that the "explicit builder acknowledgement" clause needs concrete operationalization (vs. theoretical edge case). | On the 20-seed calibration, count how often all 3 customers abstain. If ≥ 25% of seeds trigger this path, the builder-acknowledgement YAML format becomes load-bearing and deserves its own AC in the Feature spec. |


## SpecScore Integration

- **New Features this would create:**
  - `sidekick-consilium` (in this repo) — the `specstudio:consilium` skill that orchestrates the 5-stage pipeline per seed, fans out the role agents in parallel (size determined by the project's roster), invokes the researcher and scribe, and writes the verdict back to the orchestrator task and the seed's `## Consilium Verdict` section. Includes the per-project roster configuration schema (`consilium.roster` in `specscore.yaml`) and the custom-role markdown contract.
  - `consilium-review` Synchestra task type (cross-repo: [`specscore/synchestra`](https://github.com/specscore/synchestra)) — the queue resource. Created at drain time (lazily) keyed by seed `content_hash` for idempotency. Stores the full structured verdict payload as the source of truth, including which roster was active at review time (for reproducibility when the roster changes later).
  - `specscore-consilium-verdict-subcommand` (cross-repo: [`specscore/specscore-cli`](https://github.com/specscore/specscore-cli)) — the deterministic CLI arbiter. Input: a YAML votes file + the active roster spec. Output: a verdict + transition + event payload. Snapshot-tested over the gate-rule branches. Also validates the roster (≥ 1 per group, ≤ 12 total, no name collisions, custom-role files exist and parse).
  - Researcher and scribe role definitions ship as part of the consilium skill's references (`skills/consilium/roles/researcher.md`, `skills/consilium/roles/scribe.md`); the 9 default expert roles ship at `skills/consilium/roles/<role>.md`. Projects with `consilium.roster.custom` entries point at their own role definitions (typically under `.specscore/roles/<role>.md` or similar). Editing markdown reconfigures the panel without code changes.
- **Existing Features affected:**
  - `sidekick-capture` ([Implementing](../features/sidekick-capture/README.md)) — no change. The consilium consumes `sidekick-idea.captured` events emitted by that Feature.
  - `specscore:whats-next` (in `orchestrator` repo) — extends to surface `consilium-review` tasks and seeds with `needs-human-review` verdicts at the top of the prioritization report.
  - `skills/shared/events.md` (in this repo) — extends with the `sidekick-idea.reviewed` event emitted by the consilium when a verdict is finalized.
- **Dependencies:**
  - Cross-repo dependency on `specscore/specscore-cli` for the `consilium verdict` subcommand. Companion plan stub at `spec/plans/sidekick-consilium-arbiter-companion.md` (to be authored at Feature-promotion time, modeled on the `sidekick-capture-lint-rule-companion.md` pattern).
  - Cross-repo dependency on `specscore/synchestra` for the `consilium-review` task type. Same companion-plan pattern.
  - Soft dependency on Phase 0 (`sidekick-capture`) actually being deployed in target projects — seeds need to exist before the consilium can drain them.

## Open Questions

- **Custom-role definition contract.** A project's `consilium.roster.custom` entry points at a markdown file. What MUST that file contain? Tentative shape: YAML frontmatter (`name`, `group`, `output_schema_version`) + a prompt body + an example vote. Resolve at Feature spec time; should align with the default-role file format so the contract is uniform.
- **Custom-role security and sandboxing.** Loading a custom role from a path doesn't execute anything (markdown is read into the agent prompt as text), but a malicious or poorly-written custom prompt could (a) leak data, (b) systematically vote `should-implement` to game auto-promote, or (c) impersonate a default role. Phase 1 mitigations: roster validation + name-collision detection. Phase 2+ mitigation: signing/attestation. Resolve the signing question only if multi-tenant scenarios materialize.
- **Roster snapshotting at review time.** When the roster changes between two reviews of the same content_hash, do we re-review? Tentative answer: yes — the verdict task stores the roster snapshot used to produce it; a roster change invalidates prior verdicts and re-enqueues affected seeds. Validate during calibration.
- **Group-balance constraint.** Current rule: ≥ 1 member per group after exclude/add. Open: should adversaries have a *minimum* of 2 (so a single Adversary's veto isn't disproportionately weighted)? Tentative: no — the parent Idea's "single Adversary veto blocks auto-promote" rule already gives adversaries asymmetric power, and adding a minimum-2 floor is over-engineering until evidence demands it.
- **`specscore.yaml` schema authority.** This Idea introduces a `consilium:` top-level block in `specscore.yaml` containing `roster` and `gate` sub-blocks (with `auto_promote` reserved for Phase 2). Does `specscore/specscore-cli` own that schema, or does each plugin contribute its own block via a registered extension point? Resolve at Feature spec time; preference is for plugins to own their blocks under a top-level plugin namespace, with specscore-cli providing the registration mechanism.
- **Exact gate-knob ranges and discreteness.** Each knob has a documented range (e.g., `adversary_veto_confidence: low | medium | high`). Open: should knobs be discrete enums (proposed) or continuous numeric thresholds (e.g., `0.0–1.0`)? Discrete enums match the 3-step ordinal used elsewhere in this Idea (🟢/🟡/🔴, low/med/high). Continuous would allow finer calibration but breaks the ordinal model. Default to discrete; revisit if calibration data shows the granularity is too coarse.
- **Gate-knob baseline calibration vs. override timing.** Phase 1 ships gate knobs but the 20-verdict calibration target applies only to the *default* gate. If a project overrides gate knobs *before* running calibration, the 95% post-hoc agreement gate doesn't apply to them — they own their own calibration. Should the skill emit a warning when invoked against an overridden gate that hasn't had its own calibration run? Probably yes (advisory only, not blocking). Resolve at Feature spec time.
- **Token-budget calibration.** The 25K target is held from the parent Idea's arithmetic. After the first 20 verdicts, validate: is the median actually 24K-ish, or higher? If higher, the briefing template needs tightening.
- **Verdict re-emission on seed mutation.** If a seed file is hand-edited after a verdict has been written, is the verdict still valid? Tentative answer: no — the verdict's `content_hash` becomes stale relative to the seed's body, the task is marked `stale`, and the seed is re-enqueued. Validate with a fixture during Feature implementation.
- **Concurrency / single-writer assumption.** Phase 1 assumes a single `/consilium` run at a time. If two are kicked off concurrently against the same project, what prevents double-review of the same task? Tentative: the orchestrator task's claim semantics (task transitions `queued → claimed` atomically; second claimer sees `claimed` and skips). Verify the orchestrator task lifecycle actually provides this guarantee.

---
*This document follows the https://specscore.md/idea-specification*
