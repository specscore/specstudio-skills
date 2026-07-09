# 05 · Stack Integration: GraphSpec, ModelSpec, CodeGrapher, DTQL, OVDB

## The distributed model (founder insight, 2026-07-10 — load-bearing)

The stack's non-obvious differentiator is its **Go-modules-inspired distribution**: knowledge is published *per repo* (spec trees, codegraph snapshots, model files, decision logs — all git-versioned, all URI-addressable), and interlinked by reference, never by central registration. Studio is therefore **an indexer over a decentralized corpus — pkg.go.dev, not a database**:

- Identity = URIs (spec path-slugs, `modelspec:///module.Name`, repo+package+symbol, `specscore.md/...` links already embedded in spec headers across ~19 repos).
- Publishing = pushing to git. No submission step, no central approval, no lock-in.
- Cross-repo links resolve lazily at index time; a dangling reference is a *finding*, not a failure (exactly Go's compile-time missing-module semantics, softened to a report).
- Adoption is incremental per repo — the enterprise wedge: point Studio at three repos, expand when it earns it.
- Consequence for Studio's storage: the fact store is a **rebuildable cache**; the repos remain the system of record. Studio can be deleted and reconstituted — the property that makes it trustworthy enough to centralize *attention* without centralizing *truth*.

## GraphSpec — mostly hidden infrastructure

**Role:** the schema and authoring format for *declared* relationships and domain semantics (entities, relationships, commands, events, policies). **Visibility:** authors see it (it's how you declare); readers should not need to know it exists — its content surfaces as evidence chips ("declared: GraphSpec bookius/booking") and as the Map's declared-edge layer. Generated-vs-manual split: structural relations derivable from code (module uses module) should be *generated into review-able GraphSpec* (proposals, not silent writes); domain semantics (commands/events/policies, ownership, lifecycle vocabulary) stay hand-authored — they encode intent no parser can observe. Studio enriches (proposes), the CLI lints, git records. The current v0.2 pilot discipline (validate the language on bookius/family before populating widely) is correct — Studio's Phase-0 ingestion should consume exactly what the pilots produce and *no more*, pulling GraphSpec forward by consumption rather than by mandate.

## ModelSpec — the semantic bridge, browsed in context

Users meet models **inside products and capabilities** (a "Data" tab: the concepts this thing owns), plus a thin global Models browser for cross-cutting hunting by field name. Model pages show **projection discovery** results (04): declared annotations (`modelspec:///` URIs in source — the CLI's annotation reader is the ingestion path), inferred shape-matches awaiting attestation, and behavioral confirmations from storage-schema readers (dalgo dbschema / OVDB inferred catalogue). Two-way: code links back via annotations; attesting an inferred projection offers the codemod that writes the annotation — every attestation makes the declared layer richer and the inference layer smaller. Generation (Go/TS from ModelSpec) stays out of Studio's v1 scope: Studio *verifies correspondence*, the CLI generates — separation keeps Studio read-mostly and trustworthy.

## CodeGrapher — evidence producer + workflow verbs, never a graph picture

**Pipeline role:** per-repo `codegraph/` INGR snapshots (already committed in 11 sneat repos) are Studio's code-evidence feed. Studio's ingester adds the one capability CodeGrapher lacks today (confirmed in the review's friction report): **cross-repo symbol resolution** — module-path + version joins across snapshots, exactly Go-modules semantics again. Refresh moves from manual to CI (snapshot regenerated on push, like any build artifact).

**Workflow verbs in the UI** (each an answer template, not a visualization): *Impact* ("what churns if `facade` changes" — fan-in walk with blast-radius grouped by product); *Hotspots* (fan-in × recent-churn ranking per repo/capability); *Coverage-of-spec* (AC → Verifies commits → symbols → tests present?); *Dead code* (zero-fan-in exported symbols, cross-repo aware); *Layer check* (declared layering from GraphSpec vs observed import edges — violations become Contradiction items); *Ownership* (symbol → feature registration → product). The graph *picture* exists only inside Map (L5) for the minority who think spatially.

## Multiple graphs — one substrate, many projections

Knowledge/code/product/decision/dependency "graphs" are **lens-filtered projections of the single fact substrate** (03). They merge in storage, never in UX: each Map lens loads one projection's node/edge kinds with its own layout defaults. Merging them visually produces the hairball every graph tool dies of; merging them in storage is what makes cross-graph questions ("which *products* does this *code* hotspot endanger?") one query instead of an export-import ritual.

## DTQL & OVDB — the query and multi-tenant layers

**DTQL**: the honest current state (review-verified): a YAML serialization of the dalgo query AST, implemented in dalgo, served by OVDB's `/dtql` endpoint. That is exactly what Studio needs — **saved questions serialize their retrieval as DTQL** where they hit structured stores. It makes questions portable artifacts (a question library entry = name + DTQL + answer template + citations), and gives dtql.org a true story at last. No independent query language project required.

**OVDB**: Studio's hosted/multi-tenant storage plane when it leaves single-workspace mode — attestation logs, question libraries, fact caches per customer, with OVDB's capability grants as the sharing model (give the auditor read on facts+evidence, nothing else; give an agent write on gotchas only). Self-host = the same artifacts in the customer's inGitDB repos. The stack eats itself, in the good way — and every Studio deployment dogfoods the data plane the company also sells.

## Sneat CLI / SpecStudio skills — the hands

Studio never mutates artifacts directly: state changes route through `specscore` CLI verbs (status transitions, scaffolds, annotations) and the SpecStudio skills pipeline keeps ideate→ship as the authoring workflow. Studio is eyes and memory; the CLI is hands; git is the record. One writing path means one lint gate and no second source of truth.
