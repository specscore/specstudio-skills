# Spec Kit vs. SpecStudio Skills — Skill-Layer Comparison

**Status:** Draft
**Date:** 2026-05-08
**Author:** Alexander Trakhimenok (with Claude)
**Audience:** Internal / self
**Companion docs:**
- [`README.md`](README.md) — the broader product-level (CLI vs. CLI) analysis: Spec Kit vs. SpecScore + Synchestra + Rehearse
- [`extension-integration.md`](extension-integration.md) — manifest-level integration plan
- [`../../ideate-vs-brainstorming-skills-analysis.md`](../../ideate-vs-brainstorming-skills-analysis.md) — the upstream pattern that `specstudio:ideate` / `specstudio:specify` were derived from

---

## TL;DR

Same job — *get from prompt to spec without writing the wrong thing* — solved at three different layers:

| Layer | Project | What it ships |
|---|---|---|
| **CLI + template stack** | `github/spec-kit` | Python `specify` CLI; per-agent install matrix; 7-stage slash-command funnel; prose templates; manifest-driven plugins |
| **Skill set, freeform output** | `obra/superpowers` (`brainstorming`) + `addyosmani/agent-skills` (`idea-refine`, `spec-driven-development`, …) | Markdown skill files invoked via the host runtime's Skill tool; produce prose markdown to `docs/` |
| **Skill set, schema-validated output** | `specscore/specstudio-skills` (`specstudio:ideate`, `specstudio:specify`) | Same Skill-tool invocation surface, but artifacts land in `spec/` and are gated by `specscore lint` |

**SpecStudio skills are the schema-bearing skill-layer expression of the same SDD methodology Spec Kit popularized at the CLI layer.** They share Spec Kit's funnel shape (idea → spec → …) but inherit Superpowers' hard-gate discipline and agent-skills' divergent/convergent ideation, then bind every artifact to a typed SpecScore tree so the output is machine-checkable rather than prose-by-convention.

If Spec Kit is the on-ramp, and Superpowers/agent-skills are the workflow library, **SpecStudio skills are the slot where SDD becomes a typed contract** — the same activities, but their outputs are addressable, lintable, and queryable.

---

## 1. The Three Approaches Side-by-Side

| | Spec Kit | Superpowers + agent-skills | SpecStudio skills |
|---|---|---|---|
| **Distribution surface** | Python CLI (`uv tool install specify-cli`) + per-agent install scripts (30+) | Skill files in plugin manifests; runtime auto-discovers | Skill files in plugin manifest; runtime auto-discovers |
| **Invocation** | Slash commands generated *by the CLI* into the agent's command directory | Skill tool (`Skill` / `skill` / `activate_skill`), plus optional slash aliases | Skill tool, plus `/ideate` and `/specify` aliases |
| **Authoring funnel** | 7 stages: constitution → specify → clarify → plan → tasks → analyze → implement | 2 disjoint pieces: `idea-refine` (agent-skills) for divergence; `brainstorming` (superpowers) for design gating | 2 composed stages: `specstudio:ideate` (divergence + convergence) → `specstudio:specify` (design + gating) |
| **Output location** | `.specify/specs/{N}-{name}/spec.md` (etc.) inside the product repo | `docs/ideas/<name>.md` (agent-skills) and `docs/superpowers/specs/YYYY-MM-DD-<topic>-design.md` (superpowers) | `spec/ideas/<slug>.md` and `spec/features/<slug>/README.md` — typed, lintable |
| **Output format** | Prose markdown with `[NEEDS CLARIFICATION]` string markers | Freeform markdown | SpecScore-typed: title-prefix dispatch key (`# Idea: …` / `# Feature: …`), bold-prefixed body metadata (`**Status:**`, `**Date:**`, …), fixed section schema, promotion graph, status lifecycle |
| **Validation** | None at the format level (agent is expected to follow templates) | None | `specscore lint` rejects placeholders, missing sections, broken promotion links |
| **"Too simple to spec" defense** | Implicit via funnel | Explicit (Superpowers `brainstorming` `<HARD-GATE>`) | Explicit, inherited from Superpowers, plus enforced by lint failure |
| **Divergent ideation** | Not a first-class step | Yes — agent-skills `idea-refine` ships SCAMPER, HMW, JTBD, etc. | Yes — `specstudio:ideate` references `frameworks.md` borrowed from agent-skills |
| **Single-question vs. batch cadence** | Implicit; depends on agent | Strict one-at-a-time (Superpowers); batch (agent-skills) | Contextual: batch in Phase 1 of `ideate`, one-at-a-time in `specify`, codified in `shared/question-cadence.md` |
| **Reviewer subagent** | No (just a prompt to the same agent) | Yes (Superpowers `spec-document-reviewer-prompt.md`) | Yes (`specstudio:specify` only; ports the Superpowers prompt and re-targets the Feature schema) |
| **Implementation handoff** | `/speckit.implement` runs the tasks in the same session | Superpowers terminates `brainstorming` by invoking `writing-plans` | `specstudio:specify` terminates by handing off to `writing-plans`; events emitted for Synchestra |
| **Multi-agent / async** | Single session, single agent | Single session | Skill-level: single session. But artifacts are addressable so Synchestra can dispatch the resulting plans to many agents/sessions/machines |
| **Cross-artifact link integrity** | None | None | `promotes_to`, `supersedes`, `depends_on` resolve as relative paths inside `spec/`; lint catches broken links |
| **Per-agent install matrix** | Required (`specify init --integration <agent>`) | Not required (skills are runtime-agnostic) | Not required (same — runtime-agnostic via Skill tool) |
| **License** | MIT | MIT (both) | Apache-2.0 (skills) + CC BY 4.0 (SpecScore spec text) |

---

## 2. Where SpecStudio Skills Sit Differently from Spec Kit

### 2.1. Layer of expression

Spec Kit is a **CLI that writes slash commands into your agent's command directory**. Its plugin model targets the CLI; the agent is a downstream consumer. To benefit from a Spec Kit extension, you need the CLI installed, the agent set up via `init --integration`, and slash commands generated. That's a real onboarding tax.

SpecStudio skills are **skills that the agent loads natively**. There is no separate CLI for the workflow itself (the only required CLI is `specscore` for lint, and only at write/commit time). The agent reads `specstudio:ideate` and `specstudio:specify` the same way it reads any other skill; no per-agent install matrix; no template generation step.

**Consequence:** Spec Kit's distribution power is huge (~92k stars, GitHub-backed, 30+ agents) — and unavailable to a skill-only project. SpecStudio skills' lift is much lighter — anyone with a Skill-tool-aware runtime gets them by adding the plugin.

### 2.2. Output as contract, not prose

Spec Kit produces prose markdown with conventions. The conventions are useful for humans and pliable for agents, but `[NEEDS CLARIFICATION]` is a string match, not a schema. Two Spec Kit users producing two `spec.md` files cannot in general be diffed, merged, queried, or lint-coupled across a fleet.

SpecStudio skills emit SpecScore-typed artifacts: title-prefix dispatch key (`# Idea: …` / `# Feature: …`), bold-prefixed body metadata (`**Status:**`, `**Date:**`, `**Owner:**`, …), and a fixed section schema. `specscore lint` rejects placeholders, missing required sections, missing required header fields, broken promotion links, and Idea/Feature graph violations. The artifacts are tool-readable: Synchestra agents pick them up by slug-as-ID; Rehearse generates test stubs from the Given/When/Then ACs; CI gates merges that touch features tracing to archived Ideas.

**Consequence:** SpecStudio skills' deliverable is a **typed contract** that survives the conversation. Spec Kit's deliverable is a **shared convention** that needs everyone in the loop to agree.

### 2.3. Funnel topology

Spec Kit's funnel is **flat and linear** (7 stages, each one a slash command). Its WBS is a flat `tasks.md` with `[P]` parallel markers.

SpecStudio's funnel is **hierarchical and compositional**:
- An Idea may promote to *one or more* Features.
- A Feature contains *many* Requirements, each with *many* ACs.
- Plans live alongside Features and reference them by ID.
- Tasks live in plans and may have sub-tasks.

The hierarchy isn't decorative — it's what lets Synchestra do parallel dispatch, what lets `specscore code` answer "what code implements this feature?", and what lets `specscore spec lint` reject a Feature whose source Idea was archived.

**Consequence:** Spec Kit fits flat work. Past a few dozen tasks, its model bends. SpecStudio's model only gets stronger as the WBS deepens.

### 2.4. Gating discipline

Spec Kit funnels: each stage produces an artifact, the next stage consumes it. There is no `<HARD-GATE>` — if you skip `/speckit.clarify`, nothing stops you.

SpecStudio inherits Superpowers' explicit `<HARD-GATE>` and tightens it with lint:
- `specstudio:ideate` won't let you proceed to `specstudio:specify` until the Idea exists, lints clean, and the user approved.
- `specstudio:specify` won't let you proceed to `writing-plans` until the Feature exists, every requirement has ≥1 Given/When/Then AC, lint passes, the reviewer subagent returned "Approved", and the user approved.

**Consequence:** Spec Kit trusts the agent to follow the funnel. SpecStudio refuses to advance until the artifact is structurally correct *and* humanly approved. Different defaults, both defensible — but SpecStudio's stricter gates are what make the typed-contract claim honest.

### 2.5. Authorship of the methodology

Spec Kit's methodology is **invented in-house by GitHub** (with prior art credited). Its templates and slash commands embody one team's view of SDD.

SpecStudio's methodology is **explicitly composed from upstream skills**:
- The divergent/convergent shape and `frameworks.md` come from `addyosmani/agent-skills/idea-refine`.
- The hard-gate, one-question-at-a-time, reviewer-subagent, and visual-companion patterns come from `obra/superpowers/brainstorming`.
- The schema discipline, lint-clean output, and addressable artifacts come from SpecScore.

The merge logic is documented in [`../../ideate-vs-brainstorming-skills-analysis.md`](../../ideate-vs-brainstorming-skills-analysis.md). The decision to keep two skills (rather than one fat skill) is justified there too: divergence and gating are different jobs with different cadences, and conflating them weakens both.

**Consequence:** SpecStudio skills are intentionally a *remix* — that's both the value (best-of pattern compose-up) and the dependency (when upstreams move, we need to track them).

---

## 3. Where the Skill-Layer Comparison Differs from the Product-Level Comparison

The companion [`README.md`](README.md) makes the case to **integrate, not compete** with Spec Kit at the *product* level — by shipping `speckit-specscore` (preset) and `speckit-synchestra` (extension) into Spec Kit's plugin ecosystem.

That argument is unchanged here, but the skill-layer view adds a separate observation:

**Spec Kit's plugin model can carry SpecScore's schema, but it can't carry SpecStudio skills' gating.** Because Spec Kit extensions are *namespaced commands plus before/after hooks*, they can run lint after `/speckit.specify` (good) but cannot replace `/speckit.specify` itself with a skill that interrogates the user one question at a time, dispatches a reviewer subagent, and refuses to advance until the artifact lints. That's outside the extension contract.

So the integration story is asymmetric:

- **Format integration** (`speckit-specscore` preset) — clean, high-leverage, shipping recommended.
- **Execution integration** (`speckit-synchestra` extension) — clean, high-leverage, shipping recommended.
- **Authoring-skill integration** — *Spec Kit and SpecStudio skills don't compose at the authoring step.* A user who wants Spec Kit's distribution and SpecStudio's gating today has to pick one.

This is fine. The two are aimed at different runtimes anyway: Spec Kit's authoring funnel is *the slash command files*, while SpecStudio's authoring funnel is *the Skill tool*. Users who live in agents that surface skills natively (Claude Code, Copilot CLI, Gemini CLI) get more from SpecStudio. Users who want a CLI-driven workflow with per-agent install scripts get more from Spec Kit. The format/execution layers can still compose downstream of either.

---

## 4. Where SpecStudio Skills Sit Differently from Superpowers / agent-skills

Already covered exhaustively in [`../../ideate-vs-brainstorming-skills-analysis.md`](../../ideate-vs-brainstorming-skills-analysis.md). One-paragraph rollup for this doc:

- **vs. agent-skills `idea-refine`:** SpecStudio's `ideate` keeps the 3-phase divergent/convergent structure and the framework library, but makes the artifact mandatory (`spec/ideas/<slug>.md`, lintable) instead of optional, and adds an explicit promotion graph to Features.
- **vs. Superpowers `brainstorming`:** SpecStudio's `specify` keeps the `<HARD-GATE>`, the one-question-at-a-time cadence, the reviewer subagent, and the visual companion (optional), but replaces freeform `docs/superpowers/specs/YYYY-MM-DD-*.md` with a typed `spec/features/<slug>/` tree; date and authorship live in bold-prefixed body metadata, not the filename.
- **vs. agent-skills `spec-driven-development`:** Both push "spec before code"; agent-skills produces prose, SpecStudio produces a SpecScore Feature. Where agent-skills relies on the agent's discipline, SpecStudio backstops with lint and a reviewer subagent.

In short: SpecStudio skills don't *replace* the upstream skills — they restate the same disciplines on top of a typed substrate, then add the substrate-specific affordances (graph, lint, events, addressability) the upstreams can't have because they don't have a substrate.

---

## 5. Open Questions

1. **Should SpecStudio skills explicitly declare the upstreams they descend from in their skill front-matter?** Today the lineage is in research notes only. Making it visible in the skill itself would help onboarders understand why the patterns look the way they do — and would honor the credit.
2. **Does `specstudio:specify` need a Spec-Kit-style `before_implement` hook concept** so projects can plug in custom gating beyond the reviewer subagent? Plausible if the skill grows a plugin model itself; out of scope for the MVP.
3. **Where does `/speckit.constitution` live in the SpecStudio model?** Probably as a new artifact `spec/principles.md` or in the SpecScore project definition — see action item #1 in [`../superpowers-and-agent-skills-analysis.md`](../superpowers-and-agent-skills-analysis.md). The skill that authors it would be a sibling of `ideate` and `specify`, not part of either.
4. **Should there be a CLI front-door for SpecStudio skills**, mirroring Spec Kit's `specify init` per-agent install matrix? `specscore install` is the closest equivalent today, scoped to installing the lint binary. A skill installer with `--agent claude|copilot|gemini|cursor` flags would close the parity gap.
