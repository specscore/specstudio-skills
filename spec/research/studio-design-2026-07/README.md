# SpecScore Studio — Product Design (2026-07)

**Author:** Claude (Chief Product Architect / Knowledge Architect / DX Lead) · **Dataset:** the Sneat ecosystem (~190 repos, 63 domains, 30 products) · **Method:** the 2026-07 ecosystem review (`sneat-co/backstage/spec/research/ecosystem-review-2026-07/`) plus one full execution night in which the author and ~20 AI agents *were* the target users — every friction logged here was actually hit, with a timestamp.

## 1 · Product vision

**SpecScore Studio is the answer layer for software ecosystems.**

Not a documentation site (docs become projections of it), not an IDE (it never edits code), not a graph explorer (graphs are its evidence, not its UX). Studio is the place where any question about an ecosystem — asked by a human or an AI agent — gets an **instant, cited, freshness-stamped answer**, and where knowledge produced while working (decisions, gotchas, session outcomes) lands as navigable facts instead of chat ephemera.

The one-line pitch, in the voice of the person who needs it: *"I just joined / I just woke up as an agent with empty context. Make me dangerous in ten minutes, and never let me act on a stale fact without knowing it's stale."*

### Why this and not a better wiki

The Sneat review found the failure mode of every artifact-centric system: **uncompleted transitions**. Code moved, brands renamed, statuses froze at "Approved" while products shipped, two standards contradicted each other for eleven days, 97 links rotted. Nobody was negligent — artifact maintenance simply loses to shipping, always. The conclusion is architectural, not moral: *truth must be observed, not transcribed.* Studio's substrate is generated and verified continuously; hand-written text is reserved for the one thing observation can't produce — intent and reasoning.

### The four primitives (founder-approved 2026-07-10)

1. **Facts, not pages.** Every statement is a subject–predicate–object atom with evidence and a verified-at stamp.
2. **Questions as first-class citizens.** Navigation, search and the home screen are organized around answering questions, not displaying entities. Saved questions have continuously re-verified answers.
3. **Agents as co-users who write back.** Studio serves context to agents (MCP) and receives knowledge from them — quarantined until attested. AI output becomes navigable knowledge.
4. **Repositories demoted to evidence.** First-class nouns are Products, Capabilities, Decisions, Models. A repo is *where*, never *what*.

### Success metric (one number)

**Time-to-confident-answer** for a defined benchmark of 50 real questions (harvested from the Sneat review; see 01), measured for (a) a new human contributor, (b) a fresh AI agent with empty context. Studio v1 must beat "agent with grep" by 10× on cost and 5× on latency, with citations. If it can't, it isn't worth its complexity — see 12-critical-review.

## 2 · Final reflection (the decades framing)

In a decade, most code will be written and most questions asked by agents. The scarce resources will be **trust** (is this fact current? who says so?) and **shared memory** (what did the last thousand agent-sessions learn?). Version control solved shared memory for code; nothing has solved it for *understanding*. That is the seat Studio takes: the system of record for what an engineering organization knows — with the same durability guarantees git gave to source. Sneat, with its 190 repos and one human, is not an edge case; it is an early picture of every organization's ratio of knowledge to rememberers.

## Contents

| File | Deliverables |
|---|---|
| [01-ux-philosophy.md](01-ux-philosophy.md) | 2 UX philosophy · 14 contextual information · the question benchmark |
| [02-information-architecture.md](02-information-architecture.md) | 3 IA · 4 navigation · 17 screen hierarchy |
| [03-entity-and-evidence-model.md](03-entity-and-evidence-model.md) | 5 knowledge architecture · 6 entity model · 8 evidence model |
| [04-traceability.md](04-traceability.md) | 7 traceability + semantic traceability |
| [05-stack-integration.md](05-stack-integration.md) | 9 GraphSpec · 10 ModelSpec · 11 CodeGrapher · multiple graphs · DTQL/OVDB |
| [06-ai-model.md](06-ai-model.md) | 12 AI interaction model |
| [07-search.md](07-search.md) | 13 search architecture |
| [08-docs-strategy.md](08-docs-strategy.md) | 15 generated documentation strategy |
| [09-wireframes.md](09-wireframes.md) | 16 low-fi wireframes |
| [10-architecture.md](10-architecture.md) | Studio's own software architecture |
| [11-roadmap.md](11-roadmap.md) | 18 roadmap + commercial perspective |
| [12-critical-review.md](12-critical-review.md) | 19 critical review of this proposal |

## Open Questions

- Which two design-partner organizations (beyond Sneat) rebuild the question benchmark for Phase 1 — and how different is their corpus from Sneat's unusually well-artifacted one?
- Fact-store schema versioning: how do adapter upgrades migrate cached facts (rebuild-all is the v1 answer; when does it stop being cheap)?
- Agent reputation: what failure data must accumulate before write-scopes widen beyond gotchas/ledgers/proposals?
- Does the existing Angular specscore-studio-app get repointed (as 10-architecture assumes) or rewritten — decision needs a spike against the fact-store rendering model.
