# 04 · Traceability Architecture

## Anchors, not chains

The prompt's 24-hop chain (Vision→…→Documentation) is the right *aspiration* and the wrong *implementation target*. Demanding every hop everywhere produces either busywork or fiction. Instead: **trace anchors** — a small set of hops that are (a) cheap to capture at the moment of work, (b) mechanically verifiable later. Everything between anchors is derived or on-demand.

### The anchor set (v1) — each already has a capture mechanism in the stack

| Hop | Capture mechanism (exists today) | Evidence class |
|---|---|---|
| Idea ↔ Feature | `Source Ideas` / `Promotes To` frontmatter, CLI-synced + linted | declared |
| Feature → AC | `#ac:` identity scheme | declared |
| AC → implementation commit | **`Verifies:` commit trailer** (specstudio:implement writes it; verify skill walks it) | declared→verified (CI) |
| Commit → symbols/files | git + CodeGrapher | derived |
| Symbol ↔ symbol (uses/calls/imports) | CodeGrapher edges | derived |
| Spec/code ↔ decision | decision refs in spec frontmatter; `specscore:` source annotations | declared |
| Model ↔ code type | `modelspec:///` URI in annotations + projection discovery (05) | declared + inferred→attested |
| Repo → product/capability | feature/extension registrations (e.g. `Extension()` config, `provide<X>Internal()`) parsed | derived |
| Domain → product → deployment | registry + wrangler/workflow parses + live probe | derived + verified |
| AC → test | test files referenced by Verifies-commits; naming/registration conventions | derived (weak) → attested |

**Derived hops fill the gaps**: Vision→Idea is a narrative link (authored, one per idea, cheap); Requirement→ACs is internal to SpecScore already; API endpoint/Command/Permission/UI-component hops come from framework-aware extractors (the Sneat extension model is unusually parseable: routes, extension configs, and contract tokens are all conventional — an extractor per convention, added incrementally).

## The trace view (UX)

A **Trace panel** on any AC, feature, symbol, or model: renders the anchor path in both directions with per-hop evidence chips, and *visibly marks the gaps* ("no test evidence for ac:3 — last Verifies commit 2026-06-30"). Gap-marking is the feature: fake completeness is the failure mode of every traceability tool. The specstudio `verify`/`recap` skills' per-AC verdict reports ingest directly as trace evidence.

## Semantic traceability (concept → implementations)

The cross-language chain (ModelSpec → Go DTO → TS interface → Firestore doc → SQL schema) works as **projection discovery**:

1. ModelSpec entity is the concept anchor (URI).
2. **Declared projections**: code annotated with the URI (`specscore:`/`modelspec:` comment annotations — the CLI's `code` command already reads this format).
3. **Inferred projections**: matcher over CodeGrapher symbol tables — name similarity + field-shape similarity (field names/types across Go structs, TS interfaces, dbmodels) → class-6 facts in the proposal tray; one-click attest promotes them (and optionally *writes the annotation back into source* via a codemod, moving the link from attested to declared permanently).
4. **Behavioral confirmation**: Firestore/SQL schema readers (dalgo's dbschema, OVDB's inferred catalogue — both exist) match stored shapes to the model.

Same machinery serves Feature→registration→router→UI: the "projection" concept generalizes to *any concept realized in multiple conventional places*.

## Business-rule traceability (AC → if-statement)

Honest position: linking an AC to the exact `if` is capture-at-write-time or nothing. The mechanism exists — `Verifies:` commits scoped small — so the *practice* (small, AC-scoped commits, already how specstudio:implement works) is the technology. Studio's role: show AC→commit→diff hunks, and let a reviewer attest hunk-level anchors when precision matters (validation rules, money paths). Do not build AST-level rule mining in v1; mark it explicitly as research.

## Where each link kind lives

- Declared links: in the artifacts (spec frontmatter, annotations, trailers) — **git-versioned, tool-synced**; Studio ingests, never owns them.
- Derived links: recomputed by pipelines into the fact store; disposable cache.
- Attested links: the one thing Studio itself must durably store (per-ecosystem attestation log — itself an inGitDB collection, so it stays git-portable).
- Inferred links: ephemeral until attested; expire if ignored 90 days.
