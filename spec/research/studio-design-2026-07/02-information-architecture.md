# 02 · Information & Navigation Architecture, Screen Hierarchy

## What deserves first-class navigation

Tested against the question benchmark (01): a noun earns top-level navigation only if real questions *start* from it. From the Sneat night:

| Noun | Questions start here? | Verdict |
|---|---|---|
| **Ecosystem (home)** | "what is this? what's broken? what happened?" | ✅ the hub |
| **Products** | constantly ("which door…", "is X live…") | ✅ |
| **Capabilities** | constantly ("what uses contacts", "version floor") | ✅ |
| **Decisions** | "why/was it decided/is it binding" — asked across 3 repos tonight | ✅ (the scattered-ADR fix) |
| **Models** | "what is a Happening / where does this type live" | ✅ (thin at first) |
| **Map** | "show me the shape" | ✅ (L5 workspace) |
| **Activity** | "what changed since I last looked" — every agent's first question | ✅ |
| **Search/Ask** | the universal entry | ✅ (omnipresent bar, not a page) |
| Repositories | questions *end* here ("…and where is that?") | ❌ demoted — reachable via any entity's evidence, has L4 pages, no top-nav slot |
| Customer segments, verticals, domains, technologies | filter/lens material | ❌ facets on Products, not destinations |
| Specifications | belong to products/capabilities | ❌ surfaced within their owners + search; a global spec list is a report, not a nav root |
| Tasks/plans | workflow, not knowledge | ❌ stays in the SpecStudio skills pipeline; Studio links to it |

Seven top-level destinations. Everything else is reachable in ≤2 hops via islands or search.

## The IA in one picture

```
                    ┌─ Ask/Search (omnipresent) ─┐
Home ──┬── Products ──┬── product page ──┬── specs · capabilities used · repos(evidence)
       │              └── facets: vertical/segment/maturity/revenue
       ├── Capabilities ── capability page ── providers · consumers · standards · models
       ├── Decisions ── unified registry (all repos) ── decision page ── constrains…
       ├── Models ── ModelSpec browser ── model page ── projections in code
       ├── Map ── lens-filtered graph workspace (L5)
       └── Activity ── event stream: deploys · status changes · decisions · drift alerts
Repos, domains, specs, symbols: L4 pages, reached by evidence chips, islands, search.
```

## Navigation mechanics

- **Breadcrumbs are question-paths, not folder-paths**: `contactus (capability) → consumers → debtus → version pins` records *how you got here*, restorable and shareable. A share of that trail is a share of an *investigation*, the unit of collaboration tonight's agents lacked.
- **Islands over navigation**: hover-promote to a pinned island instead of navigating whenever the user's question is about *the relationship*, not the target. Navigation count is a failure metric.
- **Lenses** (role presets, switchable anywhere): *Contributor* (default), *Architect* (adds coupling/hotspot data to every card), *Founder* (adds maturity/revenue/conversion columns), *Diligence* (adds evidence density + claims-vs-verified), *Agent* (JSON — same IA served via MCP, see 06). Lenses reorder and re-prioritize; they never hide the evidence layer.
- **Aliases resolve everywhere**: typing/linking `SizeChart`, `sizeus`, or `sizechart.app` reaches one entity. The alias table is editable data (it was the single most re-derived fact in the Sneat night).

## Screen hierarchy (deliverable 17)

```
1 Home
  1.1 Health strip: freshness %, open contradictions, drift alerts (status≠evidence), CI/deploy reds
  1.2 Shape card: the one-paragraph ecosystem summary + mini-map (generated, attested)
  1.3 Activity stream (filterable by lens)
  1.4 My questions (saved, re-verified; answer-changed badges)
  1.5 Ask bar
2 Products (index: cards w/ maturity·live·revenue chips; facets)
  2.1 Product page: What/Why (authored) · Status (generated) · Composition (capabilities)
      · Surfaces (domains, apps — live-checked) · Specs & ACs (coverage bars)
      · Decisions touching it · Commercial (lens-gated) · Evidence & repos · Activity
3 Capabilities (index: provider + consumer-count + version-spread columns)
  3.1 Capability page: contract/impl providers · consumer matrix (with pins!) · standards
      · models · hotspots (CodeGrapher) · open decisions
4 Decisions (registry: all sources, status+age; amber >30d proposed — the aging rule as UI)
  4.1 Decision page: ruling verbatim · constrains · evidence of compliance/violation · history
5 Models (by module; search by field name)
  5.1 Model page: fields · projections (Go/TS/Firestore… with confidence) · consumers
6 Map (L5): seeded by any entity; lens filters; save-as-view
7 Activity (global stream + per-entity tabs)
8 L4 leaf pages (no top nav): Repository · Domain · Spec/Feature · Symbol · Person/Agent
9 System: Sources & pipelines (ingestion health) · Alias table · Question library admin
```

## What appears first / what stays hidden

First: the health strip and shape card — *state* before *structure*. Hidden until asked: file trees, raw graphs, full catalogues, lint details, historical versions. Never hidden: freshness dots and evidence chips — they are the product.

## Multi-ecosystem note

The IA above is per-ecosystem (per org/workspace). Studio's shell adds an ecosystem switcher; nothing else changes. Cross-ecosystem federation (public standards like SpecScore itself being *consumed* by Sneat) appears as external entities with their own evidence class ("upstream").
