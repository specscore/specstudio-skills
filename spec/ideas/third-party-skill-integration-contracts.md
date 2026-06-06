---
format: https://specscore.md/idea-specification
status: Specified
---

# Idea: Third-Party Skill Integration Contracts

**Status:** Specified
**Date:** 2026-05-08
**Owner:** alexander.trakhimenok
**Promotes To:** third-party-integration
**Supersedes:** —
**Related Ideas:** —

## Problem Statement

How might we let SpecStudio adopters use third-party agent skills (brainstorming, reviewers, visual companions) alongside SpecStudio without compromising the SpecScore artifact lifecycle?

## Context

This Idea was triggered by two observations during a session on 2026-05-08. (1) When a user invokes Superpowers' `brainstorming` inside a SpecScore-managed project, the agent heuristically detects the SpecScore convention and asks the user which format to follow — surfacing an unstated boundary that should be explicit. (2) The existing `specstudio:specify` Feature already specifies a `reviewer-extension-hook` that allows additional reviewer subagents to plug in, but the registration mechanism for those reviewers is an Outstanding Question. Both signals point to the same gap: SpecStudio has no formalized contract for how third-party skills integrate with SpecScore artifacts. Prior art lives in spec/research/ideate-vs-brainstorming-skills-analysis.md (§6 single-skill vs. paired-skill decision; §7.2 Synchestra alignment requirements; §11.1 visual companion options including reuse of upstream Superpowers). The conversation that produced this Idea identified three shapes of third-party skill — Producer (writes artifacts), Reviewer (returns verdicts), Capability (emits assets/runtime) — with very different integration costs.

## Recommended Direction

The contract lives as a canonical SpecScore Feature at `spec/features/third-party-integration/` and pins three operational mechanisms — one per shape.

**Producer shape — instruction injection.** SpecStudio publishes a canonical instructions block (a platform-neutral snippet) that adopters paste into their repo's agent instructions file — `CLAUDE.md`, `AGENTS.md`, `GEMINI.md`, or equivalent. The snippet tells any third-party skill operating in the repo to use `spec/` paths, canonical SpecScore body-metadata, and `Given / When / Then` acceptance criteria, and to treat its output as a **draft handoff** — when a Producer-shape skill (Superpowers `brainstorming`, addyosmani `idea-refine`, etc.) finishes, the user runs `specstudio:ideate` or `specstudio:specify` to canonicalize. Producer output is never a finished SpecScore artifact, only draft input. SpecStudio does not maintain wrappers around individual third-party skills — adapter wrappers were rejected as too tightly coupled to upstream lifecycles.

**Reviewer shape — registration hook.** The existing `reviewer-extension-hook` Outstanding Question in `specstudio:specify` is closed by this Feature. Reviewers register through one mechanism — chosen at spec time from the candidates already listed (project-level config file, plugin manifest entries, convention-based directory discovery) — ship a subagent prompt, and return `Approved` or `Issues Found` per their own blocker/advisory taxonomy. Composition across all registered reviewers is AND: any single `Issues Found` blocks the User Review Gate.

**Capability shape — tool invocation contract.** Capabilities (visual companions, diagram renderers, accessibility auditors) expose an invocation contract: output path injection (`--out spec/features/<slug>/assets/`), completion signal, supported asset types. SpecStudio invokes them as subprocesses during `specstudio:specify`; their output lands as static assets under `spec/features/<slug>/assets/`. They never write Features.

**Why a Feature, not a `docs/` design doc.** Promoting this to a SpecScore Feature does three things at once: (1) eats SpecStudio's own dogfood — the integration boundary is itself a SpecScore artifact, lintable, versionable, machine-addressable; (2) gives third-party skill authors a stable target to write against, with explicit MUST / SHOULD / MUST-NOT teeth that prose docs lack; (3) lets `specscore spec lint` enforce contract conformance over time. A markdown doc in `docs/` was rejected because design docs without normative teeth drift, and the Shape-3 non-goal needs MUST-NOT language to stop Producer-as-Feature-writer requests from sneaking back in.

## Alternatives Considered

- **Plain markdown design doc** in `docs/contracts/third-party-skills.md`. Lost because design docs without normative teeth drift over time; the boundary needs explicit MUST-NOT language to stop Shape-3 producer requests from sneaking back in, and `specscore spec lint` won't enforce it on free-floating prose. Also fails the eat-your-own-dogfood test — SpecStudio asking adopters to use SpecScore Features while keeping its own integration boundary in unstructured `docs/` is incoherent.
- **Contract + first integration shipped together** (e.g. bundle the contract Feature with a working `specstudio:brainstorm` adapter or a wired-up Superpowers visual companion). Lost because it conflates strategy (what's allowed) with execution (one specific integration). Each adapter deserves its own downstream Feature with its own scope, lifecycle events, and reviewer pass; bundling them now blurs the gate this Idea is meant to draw.
- **Aggressive enable-all posture** — treat every third-party skill as potentially first-class, build adapters as needed including Shape-3 producers. Lost because Shape-3 Producer integration requires a post-processor that translates upstream output to canonical SpecScore form on every upstream version change; the maintenance burden roughly equals rebuilding the discipline natively (which is what `ideate` and `specify` already do).
- **Defensive posture** — document that SpecStudio doesn't accept any third-party skill integration, only canonical producers. Lost because it ignores real signal: the existing `reviewer-extension-hook` already concedes that some shapes (reviewers) should integrate, and the §11.1 visual-companion analysis already identified reuse-upstream as a viable Capability path. A blanket refusal would be retroactively forbidding what we've already partially specified.

## MVP Scope

A single approved SpecScore Feature at `spec/features/third-party-integration/` that (a) defines the Producer/Reviewer/Capability taxonomy with one normative paragraph per shape, (b) specifies the reviewer-extension-hook registration mechanism (closing the existing Outstanding Question in `specify`), (c) specifies the capability-tool contract (output path injection, completion signal, supported asset types), and (d) declares Shape-3 third-party producers as an explicit non-goal. No code, no manifest schema, no working adapter — those are downstream Features promoted from this one. Timeboxed to a single focused `specstudio:specify` session.

## Not Doing (and Why)

- Adapter skills wrapping specific third-party skills (e.g. specstudio:brainstorm) — each adapter is its own downstream Feature once the contract is approved.
- Manifest schema for third-party skill self-description — premature; revisit once at least one real third-party reviewer ships and concrete needs emerge.
- Implementation of any specific third-party integration (visual companion, reviewer registry, etc.) — strategy first, execution downstream.
- Native Synchestra-orchestrated skill governance — the contract may eventually generalize to in-org skills, but that's out of MVP scope; first ship the third-party boundary.
- A three-tier conformance hierarchy (Linux kernel ABI lens) — interesting but speculative; the two-tier model (sanctioned shapes vs. forbidden Shape-3 producers) is sufficient for MVP.

## Key Assumptions to Validate

| Tier | Assumption | How to validate |
|------|------------|-----------------|
| Must-be-true | The Producer/Reviewer/Capability taxonomy actually covers every real third-party skill SpecStudio adopters would want to integrate — not just brainstorming, reviewers, and visual companions. | Audit the top ~10 published skill packs (Superpowers, addyosmani/agent-skills, claude-plugins-official, etc.) against the three shapes during the `specstudio:specify` session. Any skill that doesn't cleanly fit one shape is evidence the taxonomy is incomplete; either expand it or document the gap as a deliberate non-goal. |
| Must-be-true | "No Shape-3 third-party producers" is acceptable to SpecStudio adopters as a hard line, not just expedient for SpecStudio maintainers. | Phrase the rule explicitly as a MUST-NOT in the Feature draft and surface it as the most controversial item at the User Review Gate. If reviewers push back ("but I want my brainstorming output to be a real Feature"), document the friction and decide whether the line holds or moves. |
| Must-be-true | The reviewer-extension-hook registration mechanism has a clean concrete answer reachable in one `specify` session — without one, the Feature this Idea promotes to can't be built, only described. | Sketch the three candidate mechanisms (project config file, plugin manifest entry, convention-based directory discovery) in the Feature's Outstanding Questions before writing the registration REQ. Pick one with a one-line rationale; if all three feel premature, surface that as a dependency and time-box. |
| Must-be-true | A pasted instruction block in `CLAUDE.md` / `AGENTS.md` / equivalent is robust enough that competent agents follow it under Producer-shape invocation (e.g. running Superpowers `brainstorming` produces ~SpecScore-shaped output the agent then voluntarily hands off to `specstudio:ideate` / `specstudio:specify`). If agents routinely ignore the snippet or partially comply, the Producer mechanism collapses and Producers must be rejected outright instead. | Author a draft snippet, run Superpowers `brainstorming` against three real prompts inside a SpecScore-managed repo with the snippet installed, and audit each output against the canonical SpecScore schema (path, body-metadata, AC format, handoff prompt presence). Pass criterion: ≥80% conformance on artifact shape and 100% on the handoff signal. Below threshold means the snippet design needs work or the mechanism is unworkable. |
| Should-be-true | The Capability contract (output path injection, completion signal, supported asset types) is generalizable enough to cover both visual-companion-class tools and future capabilities (e.g. diagram renderers, screenshot grabbers, accessibility checkers). | Stress-test the contract against three concrete capability examples during the `specify` session — Superpowers visual companion, a hypothetical Mermaid diagram renderer, a hypothetical Lighthouse-class auditor. If the contract needs per-capability special cases, it's not a contract — refactor or narrow scope. |
| Should-be-true | The contract is stable enough that third-party skill authors can target it without churn, but flexible enough that SpecScore can evolve. | Define a versioning rule for the contract itself (e.g. "contract changes go through `specify` revise-in-place; backward-incompatible changes require a `supersedes` and a deprecation window"). Test by simulating one breaking change in scope. |
| Might-be-true | This contract eventually generalizes to native Synchestra-orchestrated skills (in-org skills, not just third-party), and the same Producer/Reviewer/Capability taxonomy applies there. | Out of MVP scope. Revisit after at least one real third-party reviewer or capability has integrated successfully and we have empirical signal on whether the contract carries to in-org consumers without modification. |
| Might-be-true | Other Synchestra-aware skill packs will adopt this contract pattern (declaring their own integration boundaries as SpecScore Features) once SpecStudio ships one. | Out of MVP scope. Worth tracking but don't optimize for. The validation is observational — does anyone copy the pattern? |


## SpecScore Integration

- **New Features this would create:** `spec/features/third-party-integration/` — the canonical integration contract Feature itself. Likely downstream Features (out of MVP scope, listed for traceability): one Feature per shipped Capability adapter (e.g. `spec/features/visual-companion-adapter/`), one Feature per registered Reviewer, plus a possible `spec/features/skill-manifest/` once enough adapters exist to need a shared schema.
- **Existing Features affected:** [`spec/features/skills/specify/`](../features/skills/specify/README.md) — closes its `reviewer-extension-hook` registration Outstanding Question by referencing the new contract. [`spec/features/skills/ideate/`](../features/skills/ideate/README.md) — gains an explicit Source-Idea / Producer-non-goal cross-reference making clear that `ideate` is the only sanctioned Idea producer.
- **Dependencies:** [`spec/research/ideate-vs-brainstorming-skills-analysis.md`](../research/ideate-vs-brainstorming-skills-analysis.md) §6, §7.2, and §11.1 — the analysis that surfaced the three-shape taxonomy and the §11.1 visual-companion options. No blocking in-flight work; this Idea can be specified independently.
- **Consumed by:** [`spec/ideas/specstudio-init-skill.md`](specstudio-init-skill.md) — the proposed `specstudio:init` skill consumes the canonical Producer-shape instruction snippet defined by this Feature, installing it into the appropriate platform agent-instructions file (`CLAUDE.md` / `AGENTS.md` / `GEMINI.md`). The two Ideas are independent — this one defines the snippet text and the integration rules; the init Idea defines how a project is bootstrapped end-to-end including the snippet install step.

## Open Questions

- **Reviewer registration mechanism — pick one of three at spec time.** Candidates: (a) project-level config file (e.g. `.specscore/reviewers.yaml`); (b) plugin manifest entries declared by the third-party skill pack itself; (c) convention-based directory discovery (e.g. scan `spec/reviewers/<name>/` for prompts). The decision needs to be made during `specstudio:specify`, not deferred again.
- **Does the contract apply to non-Claude agents (Cursor, Copilot, Gemini CLI, Codex)?** Each platform handles skills differently; the contract may need platform-conditional clauses or may explicitly scope to Claude-compatible runtimes for MVP.
- **Contract versioning rule.** When SpecStudio later refines the contract, how do third-party skill authors discover and adapt? Default proposal (revise-in-place; `supersedes` only on breaking change with a deprecation window) needs to be made explicit in the Feature.
- **Does "Capability" need sub-shapes?** Visual companions, diagram renderers, and accessibility auditors may share a runtime-tool shape but emit very different artifact types. Decide whether one Capability contract covers all or whether sub-shapes are needed.
- **Where do reviewer prompts physically live in the repo?** Inside the `specstudio:specify` skill (`skills/specify/references/reviewers/<name>.md`)? In a sibling `skills/reviewers/` directory? In `spec/reviewers/`? Tied to the registration mechanism choice.
- **Where does the canonical Producer-shape instruction snippet live?** Candidates: (a) inside the `third-party-integration` Feature directory at `spec/features/third-party-integration/snippet.md`; (b) as a separate file under `skills/shared/agent-instructions-snippet.md`; (c) generated on demand by a `specscore` CLI subcommand (e.g. `specscore snippet --target claude`). Option (a) makes it lintable as a Feature asset; (c) makes it dynamic per platform. Decide at spec time.
- **How are snippet revisions communicated to adopters who already pasted an older version?** Candidates: (a) embed a version comment in the snippet itself and let an `specstudio:init --update` flow detect drift; (b) make the snippet a single line that points to a hosted URL the agent fetches (rejected if it requires network); (c) leave it manual — adopters re-paste on contract revisions. Tightly related to the contract-versioning rule already surfaced as a Should-be-true assumption.

---
*This document follows the https://specscore.md/idea-specification*
