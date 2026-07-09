# 01 · UX Philosophy & Contextual Information

## Design around questions, not entities

The entity page is the *fallback*, not the destination. Studio's unit of UX is the **answered question**: a direct answer, its evidence chips, its freshness stamp, and the follow-up questions it unlocks. This inverts the wiki/doc-site model where the user must know which page contains their answer.

### The question benchmark (harvested from real use, Sneat 2026-07)

These were actually asked — by the reviewing agent, by sub-agents mid-task, or by the founder — during one working night. They are the v1 acceptance suite; Studio ships when it answers them instantly with citations:

**Orientation:** What is this ecosystem? What are the products and how mature is each? Which are live? What earns money / is closest to? Who is the user of X?
**Ownership & location:** Which repo owns X? Which repo implements brand Y (trackus→AnyMeter class)? Where is the canonical rule about Z (naming standards had two contradicting copies)? Where do specs for X live?
**Dependency & impact:** What uses contactus? What breaks if `facade` changes (fan-in 104)? Which apps pin which platform version? Is this extension consumed anywhere?
**Truth & freshness:** Is this status current? (shipped products sat at "Approved" for weeks) Is this doc claim still true? (datatug-core README lied) Does this domain actually serve? (investor CTA was dead) When was this fact last verified?
**Why:** Why does this exist? Which decision introduced it? (ADRs scattered across 3 repos) What was decided about X and is it still binding? (Synchestra parked; anonymous-auth rejected)
**Operational:** How does this repo deploy? (answer changed mid-night for 28 repos) What secrets/permissions does this pipeline need? What are the known gotchas here? (pnpm `overrides:` block silently defeating version bumps — discovered twice)
**Lifecycle:** Which acceptance criteria of feature F are implemented/tested? (the `Verifies:` commit-trailer chain) Which specs have no implementation, which implementations no spec?
**Commercial:** Which doors have conversion instruments? What plan-signal is the waitlist showing? Which products are landing-only?

## Cognition: how understanding actually builds

A newcomer (human or agent) builds a mental model in layers, and Studio should serve exactly one layer at a time:

1. **Shape** — "it's one platform with ~30 doors on a shared graph, plus a dev-tools line." One paragraph, one diagram. (The Sneat review needed 7 agents to produce this sentence; Studio should serve it in second one.)
2. **Vocabulary** — the ecosystem's proper nouns and their aliases (sizeus=SizeChart, ext-X vs X, spaceus=tenancy). Without this layer, every search fails. Studio maintains the alias table as first-class data.
3. **Load-bearing structure** — the 10 facts that constrain everything (facade fan-in; public/private CI boundary; contract/shared/internal triad; storage plane options). Studio marks facts as *load-bearing* (high in-degree in the fact graph) and surfaces them early.
4. **Local detail** — only now, and only on demand, the file/symbol/AC level.
5. **Confidence** — the layer no tool serves today: *how do I know this is true and current?* Every answer carries it; users learn to trust the green dots and interrogate the grey ones.

**Anti-goal:** completeness-first displays. A page listing all 142 feature nodes taught the review nothing; the statement "the lifecycle's right side is unused — one feature ever reached Stable" taught everything. Studio prefers the *significant* over the *complete*, with the complete one click behind.

## Progressive disclosure: the five levels

| Level | Surface | Contents | Budget |
|---|---|---|---|
| L1 | Tooltip | one-liner + status pill + freshness dot | instant, ≤1 line |
| L2 | Hover card | the entity's **answer card**: 5–8 load-bearing facts, each with evidence chip; primary relations as chips | ≤ 1 glance (no scrolling) |
| L3 | **Island** (pinned side panel) | tabbed: Facts · Evidence · Related · Activity · Ask | persists while user keeps navigating; stackable (≤3) |
| L4 | Entity page | everything known, grouped by question ("What is it / Where is it / What uses it / Why / How healthy") | scroll ok |
| L5 | Map workspace | graph exploration seeded from the current entity, lens-filtered | expert mode |

Rules: any element at any level can be **promoted** (tooltip→card→island→page) without losing scroll/selection context; islands stack in a right rail and survive navigation — the review's constant pain of "I need the contactus versions *while* reading the debtus page" is the island's reason to exist. Escaping to L4/L5 should feel like a choice, not a requirement; instrument how often users are forced to L4 — it's a design-failure counter.

## Contextual information islands (L2/L3 content spec)

Per entity kind, the hover card / island leads with (in order):

- **Product:** one-liner · maturity + live-check dot · brand↔repo↔domain chips · capability chips it composes · revenue model + conversion instrument state · last meaningful activity · open decisions touching it.
- **Capability:** one-liner · providers (frontend/backend chips) · consumer count with expandable list · version floor spread across consumers (the 0.8→0.22 disease, permanently visible) · canonical standard link.
- **Specification (Idea/Feature):** status + *status-vs-evidence* indicator (spec says Approved, deploys say live → amber drift chip) · AC implementation/test coverage bar (`Verifies:` chain) · source idea / promoted features · owning product.
- **Decision (ADR/seed-decision):** status + age (Proposed >30d = amber) · what it constrains (entities) · superseded-by/supersedes · the one-sentence ruling, verbatim.
- **Repository:** what it implements (products/capabilities — the *reason* it exists) · verdict (core/active/parked/archive) · deploy method + last deploy state · CodeGrapher summary (top fan-in symbols, size) · CI health · known gotchas.
- **Model (ModelSpec entity):** fields · projections discovered (Go DTO / TS interface / Firestore doc) with match-confidence · consuming specs.
- **Domain:** serves? (live HTTP check) · fronts which product · registrar/renewal · GA property.

Every island row is itself hoverable (recursion is the point), and every island has an **Ask** box scoped to the entity ("why does this depend on calendarius?") answered by the AI layer with citations (06).

## Tone & trust affordances

- **Freshness is a visual primitive**: green (verified < 24h, by pipeline), grey (verified < 30d), amber (stale or evidence conflict), red (contradicted by newer evidence). The review's docs-vs-reality chapter becomes a *color*, everywhere, always.
- **Never render a claim without its chip.** If Studio doesn't know why it believes something, it says "unverified claim (README, 2026-05)" — modeling the honesty it wants from its users.
- **Contradictions are content**, not errors: when two sources disagree (the `ext-<id>` vs `<id>-contract` naming incident), Studio shows both with dates and flags the conflict as a first-class item on the home screen — that conflict sat unnoticed for 11 days in files; it should not survive 11 minutes in Studio.
