# Competitors & Adjacent Tools

> [SpecScore.**Studio**](https://specscore.studio): | [Explore](https://specscore.studio/app/p/github.com/specscore/specstudio-skills/spec/research/competitors?op=explore) | [Edit](https://specscore.studio/app/p/github.com/specscore/specstudio-skills/spec/research/competitors?op=edit) | [Ask question](https://specscore.studio/app/p/github.com/specscore/specstudio-skills/spec/research/competitors?op=ask) | [Request change](https://specscore.studio/app/p/github.com/specscore/specstudio-skills/spec/research/competitors?op=request-change) |

How **SpecStudio skills** (`specstudio:ideate`, `specstudio:specify`, …) sit alongside other tools in the Spec-Driven Development (SDD) and AI-engineering-discipline space.

These notes are research, not contracts. The comparison subject is the *skill set itself* — the design choices behind each skill, where they came from, what they enforce.

---

## The Landscape in One Picture

```
                  ┌────────────────────────────────────────┐
                  │   github/spec-kit                      │
                  │   CLI + slash-command funnel + plugin  │
                  │   API. Prose markdown output.          │
                  │   ~92k★, MIT, GitHub-backed.           │
                  └────────────────────────────────────────┘
                                    ▲
                                    │ different layer
                                    │ (CLI vs Skill tool)
                                    │
   ┌──────────────────────┐         │         ┌──────────────────────┐
   │ obra/superpowers     │         │         │ addyosmani/          │
   │ Workflow skills:     │         │         │ agent-skills         │
   │ brainstorming, TDD,  │◀────────┼────────▶│ Discipline skills:   │
   │ debugging, plans, …  │   share │  share  │ idea-refine, spec-   │
   │ Hard-gate discipline.│   tool  │  tool   │ driven-dev, review,  │
   │ MIT.                 │ surface │surface  │ ship, …  MIT.        │
   └──────────────────────┘         │         └──────────────────────┘
              │                     │                     │
              │ pattern             │                     │ pattern
              │ source              │                     │ source
              ▼                     ▼                     ▼
                  ┌────────────────────────────────────────┐
                  │   specscore/specstudio-skills      │
                  │   ideate + specify (so far).           │
                  │   Same Skill-tool surface as upstreams.│
                  │   Output: SpecScore-typed artifacts in │
                  │   spec/ideas/, spec/features/.         │
                  │   Lint-gated. Apache-2.0 + CC BY 4.0.  │
                  └────────────────────────────────────────┘
```

**The core claim:** SpecStudio skills are the **schema-bearing skill-layer expression of SDD**. They borrow Superpowers' gating discipline, agent-skills' divergent/convergent ideation, and Spec Kit's funnel topology — then bind every artifact to a typed SpecScore tree so the output is machine-checkable rather than prose-by-convention.

---

## At a Glance

| Tool | Layer | Output | Validation | License |
|---|---|---|---|---|
| **github/spec-kit** | Python CLI + per-agent install matrix; 7-stage slash-command funnel | Prose markdown in `.specify/specs/{N}-{name}/` | None at the format level (`[NEEDS CLARIFICATION]` is a string, not a schema) | MIT |
| **obra/superpowers** | Skill set (Claude Code / Copilot CLI / Gemini CLI / Codex / OpenCode) | Freeform markdown in `docs/superpowers/specs/YYYY-MM-DD-<topic>-design.md` | None; relies on `<HARD-GATE>` + reviewer subagent + user approval | MIT |
| **addyosmani/agent-skills** | Skill set (Claude Code) + reference checklists + agent personas | Freeform markdown in `docs/ideas/<name>.md`, etc. | None; relies on per-skill checklists and anti-rationalization tables | MIT |
| **specscore/specstudio-skills** | Skill set (Claude Code / Copilot CLI / Gemini CLI) backed by `specscore` CLI | Typed artifacts in `spec/ideas/<slug>.md`, `spec/features/<slug>/` — bold-prefixed body metadata (`**Status:**`, `**Date:**`, …) per the SpecScore convention, plus a fixed section schema | `specscore lint` + reviewer subagent + user approval | Apache-2.0 (skills) + CC BY 4.0 (SpecScore spec text) |

---

## How SpecStudio Skills Are Different

### vs. **github/spec-kit**

- **Different runtime layer.** Spec Kit is a CLI that *generates* slash commands into the agent's command directory; SpecStudio skills *are* the skills the agent loads natively. No per-agent install matrix; no template generation step.
- **Typed contracts vs. shared conventions.** Spec Kit emits prose markdown with conventions; SpecStudio emits SpecScore-typed artifacts (title-prefix dispatch key, bold-prefixed body metadata, fixed section schema) that lint clean against `specscore lint` and survive the conversation as machine-readable contracts.
- **Hierarchical WBS.** Spec Kit's `tasks.md` is flat with `[P]` parallel markers; SpecStudio artifacts compose into Idea → Feature(s) → Requirement(s) → AC(s) → Plan(s) → Task(s) graphs.
- **Stronger gates.** Spec Kit funnels but doesn't refuse — skip a stage and nothing stops you. SpecStudio's `<HARD-GATE>` plus lint refuses to advance until the artifact is structurally correct *and* humanly approved.
- **Different distribution arithmetic.** Spec Kit has GitHub's distribution (~92k stars, 30+ agent integrations, 100+ community extensions); SpecStudio is pre-1.0. The product-level recommendation in [`github-spec-kit/README.md`](github-spec-kit/README.md) is to *integrate, not compete* — ship `speckit-specscore` as a preset and `speckit-synchestra` as an extension.

Full skill-layer detail: [`github-spec-kit/vs-specstudio-skills.md`](github-spec-kit/vs-specstudio-skills.md).

### vs. **obra/superpowers** (`brainstorming`)

- **Same gating philosophy, typed output.** SpecStudio's `specify` inherits the `<HARD-GATE>`, one-question-at-a-time cadence, reviewer subagent, and optional visual companion — but writes to `spec/features/<slug>/` (typed, lintable, addressable) instead of `docs/superpowers/specs/YYYY-MM-DD-<topic>-design.md` (prose, file-named-by-date).
- **Body metadata beats filename conventions.** Date and authorship live in `**Date:**` / `**Owner:**` lines under the title, not in the filename. Slugs become stable IDs that promotion links can resolve.
- **Lint replaces vibes.** Superpowers' "spec self-review" is a prompt that asks the agent to check its own work; SpecStudio backstops the same review with `specscore lint` so placeholders, missing sections, and broken graph edges become machine-detected failures.

Full detail: [`../ideate-vs-brainstorming-skills-analysis.md`](../ideate-vs-brainstorming-skills-analysis.md).

### vs. **addyosmani/agent-skills** (`idea-refine`, `spec-driven-development`, …)

- **Same divergent/convergent shape, mandatory artifact.** SpecStudio's `ideate` keeps the 3-phase structure and the framework library (SCAMPER, HMW, JTBD, …) — but makes the Idea artifact mandatory and lint-clean instead of an optional save.
- **Promotion graph.** agent-skills' `docs/ideas/<name>.md` is a one-pager dead-end; SpecStudio Ideas have a `promotes_to` field that Synchestra populates when Features are born from them, so the lineage stays queryable.
- **Reference checklists and personas as future work.** agent-skills' OWASP / WCAG / Core Web Vitals reference docs and `code-reviewer` / `security-auditor` / `test-engineer` personas are real assets we don't yet match; see action items in [`superpowers-and-agent-skills-analysis.md`](superpowers-and-agent-skills-analysis.md).

Full detail: [`superpowers-and-agent-skills-analysis.md`](superpowers-and-agent-skills-analysis.md).

---

## Why Three Comparisons, Not One

Spec Kit, Superpowers, and agent-skills are **not the same kind of project**, even though their surface areas overlap:

- **Spec Kit** is a **methodology toolkit** with a CLI and a plugin API. Its competitive surface is *distribution and ecosystem*.
- **Superpowers** is a **process-discipline skill set**. Its competitive surface is *gating rigor and workflow shape*.
- **agent-skills** is a **standards-and-checklists skill set**. Its competitive surface is *what gets checked* (security, performance, accessibility) and *what gets produced at each SDLC phase*.
- **SpecStudio skills** is a **typed-substrate skill set**. Its competitive surface is *output as contract* — making the artifact a machine-checkable thing rather than a markdown convention.

Each pair-wise comparison surfaces different decisions. Folding them together would obscure the reasoning. They live in their own files; this README is the entry point.

---

## Index

| Competitor | What it is | Comparison docs |
|---|---|---|
| **github/spec-kit** | GitHub's official Python CLI for SDD; 7-stage slash-command funnel; plugin API; ~92k★, MIT | [`github-spec-kit/README.md`](github-spec-kit/README.md) (product-level), [`github-spec-kit/vs-specstudio-skills.md`](github-spec-kit/vs-specstudio-skills.md) (skill-layer), [`github-spec-kit/extension-integration.md`](github-spec-kit/extension-integration.md) (manifest-level) |
| **obra/superpowers** (`brainstorming`) | Pre-implementation gating skill with `<HARD-GATE>`, one-question-at-a-time, reviewer subagent, visual companion | [`superpowers-and-agent-skills-analysis.md`](superpowers-and-agent-skills-analysis.md) (joint dossier with agent-skills); [`../ideate-vs-brainstorming-skills-analysis.md`](../ideate-vs-brainstorming-skills-analysis.md) (skill-layer detail vs. `specstudio:ideate`/`specstudio:specify`) |
| **addyosmani/agent-skills** | 21 skills across 6 SDLC phases, anti-rationalization tables, reference checklists, agent personas | [`superpowers-and-agent-skills-analysis.md`](superpowers-and-agent-skills-analysis.md) (joint dossier with Superpowers) |

---

## Other Tools to Research

Prominent adjacent tools that haven't yet been analysed in depth. Each has at least one design dimension worth a closer look. Roughly grouped by how directly they overlap with SpecStudio skills.

### Direct SDD overlap (priority for future research)

- **Tessl** ([tessl.io](https://www.tessl.io/)). Explicit "spec-first" development platform with a hosted workflow and a strong "AI Native Dev" brand. Closest direct competitor at the *methodology* level. Worth a deep read on what they call a spec, how their loop closes, and where they rely on hosted infrastructure vs. local files.
- **AWS Kiro** ([kiro.dev](https://kiro.dev/)). Agentic IDE with first-class *specs* and *hooks*. Specs decompose into requirements, design, and tasks (echoes SDD funnel); hooks fire on file events. Vendor-locked to AWS but conceptually close to what SpecStudio + Synchestra hooks aim for.
- **TaskMaster AI** ([github.com/eyaltoledano/claude-task-master](https://github.com/eyaltoledano/claude-task-master)). PRD-driven task generator widely used inside Cursor/Claude Code/Cline. Maps a PRD to a task list and tracks progress. Single-agent, flat-task model — the kind of flow `specstudio:specify` + Synchestra is trying to make typed and multi-agent.

### Convention-file ecosystems (different model, same intent)

- **Cursor Rules** (`.cursorrules`, `.cursor/rules/*.mdc`). Project-scoped rules that fire automatically based on file globs. Distribution surface is huge (Cursor's user base). The rules model is *passive guidance*, not *invocable skills* — interesting axis to compare.
- **OpenAI Codex CLI / `AGENTS.md`** ([github.com/openai/codex](https://github.com/openai/codex)). Lightweight markdown convention for agent instructions; widely adopted across multiple agent tools as a portable equivalent to `CLAUDE.md`. Worth tracking as the lowest-common-denominator format.
- **Aider conventions** (`CONVENTIONS.md`) ([aider.chat](https://aider.chat/)). Minimalist single-file approach — no skill model, no funnel, just a conventions doc the agent always reads. Counterpoint to the multi-skill model; useful for "what's the smallest thing that works?" arguments.
- **Cline / Roo Code workflows** (`.clinerules/`, `.roo/rules/`). Workflow-style prompt files invoked from agentic IDEs. Closer in spirit to skills than to rules — worth comparing against Superpowers / agent-skills / SpecStudio skills as a separate distribution channel.
- **Continue.dev** ([continue.dev](https://continue.dev/)). Open-source AI assistant with custom slash commands, rules, and docs ingestion. Has its own skill-shaped surface area; runs across IDEs.
- **Sourcegraph Cody** ([sourcegraph.com/cody](https://sourcegraph.com/cody)). Has prompts, context fetching, and OpenCtx — context-engineering rather than spec-authoring, but adjacent.

### First-party skill marketplaces

- **Anthropic experimental skills** ([github.com/anthropics/skills](https://github.com/anthropics/skills) and related). First-party Claude Skills repository — sets norms for what Claude Code's Skill tool expects. SpecStudio skills depend on this surface staying stable; worth a periodic compatibility review.
- **GitHub Copilot CLI plugins** ([github.com/github/gh-copilot](https://github.com/github/gh-copilot)). Skill-shaped surface inside `gh-copilot`; less mature than Anthropic's, but the same skill model applies.
- **Block / Goose recipes** ([block.github.io/goose](https://block.github.io/goose/)). Goose's "recipes" are skill-shaped (markdown + frontmatter, invoked by name). Different runtime, similar mental model — useful precedent for cross-runtime portability.

### Adjacent — methodology and PM-side, not skill-shaped

- **Linear / Notion / Jira AI features**. Requirements-management products are bolting on AI; their spec-equivalents are PRDs in proprietary stores. Out of scope as direct competitors but worth tracking for "where does the spec live?" arguments.
- **Replit Agent**, **Bolt.new**, **v0.dev**, **Lovable**. AI app builders that conflate spec, plan, and implementation. Different audience (no-code-ish), but the "spec is the prompt" framing is a real user expectation we'll keep brushing against.

### Research priorities

If picking the next two to deep-dive, the clearest yield is:

1. **Tessl** — methodology-level competitor. The right place to test our positioning claims.
2. **Kiro** — closest to the *substrate + hooks* shape Synchestra is building toward.

The convention-file ecosystems (Cursor rules, AGENTS.md, etc.) matter more for *distribution* than for *design choices*; their analysis can wait until SpecStudio skills are ready to ship companion install paths for those runtimes.

---

## Adding a New Competitor

When a new tool enters the SDD or agent-discipline space:

1. Confirm the comparison subject is the **skill set** — design choices, gating, output format, runtime surface. Tools whose overlap is purely product-level (pricing, distribution, business model) don't belong in this tree.
2. Create `spec/research/competitors/<slug>/` with a `README.md` overview.
3. If the comparison has a CLI/product-level dimension *and* a skill-layer dimension, split them into separate files (`README.md` for product-level, `vs-specstudio-skills.md` for skill-layer) — see [`github-spec-kit/`](github-spec-kit/) as the pattern.
4. Cite sources at the top of each doc with fetch date.
5. Add the new tool to the *Index* table above and to the *At a Glance* table.

## Conventions

- **Date every analysis.** Competitor docs rot fast — the date at the top tells the reader how stale the comparison is.
- **Quote, don't paraphrase, key claims.** When characterizing a competitor's design choice, link the source so the original is verifiable.
- **No speculation without labels.** If a capability is inferred from marketing rather than shipped product, say so explicitly.
- **No marketing language about ourselves.** This tree is for thinking, not selling.

## Open Questions

None at this time.

---
*This document follows the https://specscore.md/feature-specification*
