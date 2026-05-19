# Idea: Artifact Frontmatter Convention — format + status fields

**Status:** Draft
**Date:** 2026-05-19
**Owner:** alexandertrakhimenok
**Promotes To:** —
**Supersedes:** —
**Related Ideas:** —

## Problem Statement

How might we make every SpecScore artifact self-describing (which spec it conforms to) and queryable by third-party tools and databases (current status) without depending on body-markdown parsing?

## Context

Today's SpecScore artifacts express their conformance to a spec in two split places:

- **Format link** lives at the document footer as a free-text line, e.g.
  `*This document follows the https://specscore.md/feature-specification*`.
  It's prose, not structured — easy to miss when skimming, invisible to any
  tool that doesn't fetch and parse the whole document, and inconsistent
  across artifact types (Idea vs Feature vs Plan vs seed).
- **Lifecycle status** lives as a body-Markdown line — `**Status:** Approved`
  for Features and Ideas, and as YAML-frontmatter `status: queued` for
  sidekick-seeds (a deliberate exception today). The body form is
  canonical, but querying it requires Markdown parsing.

Two recent dogfood signals (2026-05-19 session) crystallized the gap:

1. The just-shipped `specscore:change-status` skill normalized CLI-driven
   status transitions for Feature and Idea. But any third-party tool —
   an external dashboard, an ingitdb job, an index/filter pipeline — that
   wants to know "what's the current status of artifact X?" still has to
   load the body and run a Markdown parser. That's a heavy contract for a
   single-field lookup.
2. Sidekick-seeds today *do* carry `status:` in frontmatter (lint-required),
   precisely because the consilium and downstream queue-management code
   needs cheap-to-read frontmatter access. That precedent already exists
   inside the system; it just hasn't been generalized.

Earlier in the same session I argued the opposite — that "SpecScore artifacts
do not expect frontmatter; status is body-only." This Idea is a change of
mind. The body form is still canonical (the CLI / lint own it); but a
mirrored frontmatter copy serves a genuinely different audience (third-party
queries, indexing, in-database filtering) at near-zero cost.

## Recommended Direction

Every SpecScore artifact MUST carry two YAML-frontmatter fields:

- **`format:`** — URL to the SpecScore specification for this artifact type
  (e.g. `https://specscore.md/feature-specification`,
  `https://specscore.md/idea-specification`,
  `https://specscore.md/plan-specification`,
  `https://specscore.md/sidekick-seed-specification`). **Mandatory for all
  documents.** Replaces the existing footer line
  `*This document follows the …*`, which is removed.
- **`status:`** — string mirror of the body `**Status:**` field, in the same
  token form the body uses (`Draft`, `Approved`, `Implementing`, `Stable`,
  `Deprecated`, `Archived`, `Queued`, `Completed`, …). **Mandatory for any
  document type that defines a Status.** Documents without a status concept
  (e.g. a README index page) omit it.

Body `**Status:**` remains the **canonical truth**. Frontmatter `status:` is
a derived mirror, maintained in sync by CLI tooling: every successful
`specscore feature change-status` / `specscore idea change-status` invocation
rewrites both the body line and the frontmatter line as part of the atomic
status rewrite. `specscore spec lint` gains a rule that flags any frontmatter
↔ body Status drift as an error, with `--fix` choosing the body value as
the source of truth.

The `format:` field is essentially static per artifact type — set at create
time by `specscore feature new` / `specscore idea new` / `specscore task new`
/ the sidekick capture skill, then never touched.

## Alternatives Considered

- **Body Status only (status-quo for Feature/Idea).** Rejected: every
  third-party reader needs a full Markdown parser to extract one string.
  ingitdb-style operations and SQL-like filtering pipelines pay a steep
  per-document parsing cost that scales badly across a multi-thousand-doc
  corpus.

- **Frontmatter Status only (no body line).** Rejected: human readers
  expect Status visible at the top of the rendered document; rendering
  pipelines that hide frontmatter (GitHub's Markdown renderer, IDE
  previews, `specscore.studio`'s artifact view) would hide the most
  important lifecycle signal. Frontmatter-only also conflicts with the
  established Feature / Idea / Plan body-meta convention.

- **Keep the footer line for format; add only frontmatter `status:`.**
  Rejected: the footer line has the same machine-parseability problem as
  body Status — buried in prose, no schema. If we're already paying the
  cost of a frontmatter migration for `status:`, generalizing to a
  `format:` mirror at the same time is a free win and removes an
  inconsistent surface.

## MVP Scope

One job: **every artifact in this repo carries the two frontmatter fields,
and `specscore spec lint` enforces them.** Concretely —

1. The `specscore-cli` lint rule set gains:
   - `frontmatter-format-required` — every Feature / Idea / Plan / seed
     MUST have a `format:` key in frontmatter pointing at the correct
     spec URL for its type.
   - `frontmatter-status-mirrors-body` — for artifact types with a
     `**Status:**` body line, frontmatter `status:` MUST exist and match
     (case-insensitive, whitespace-trimmed). `--fix` rewrites
     frontmatter from body.
   - The legacy footer-line check is removed.
2. `specscore feature change-status` and `specscore idea change-status`
   atomically rewrite both the body line and the frontmatter mirror; on
   any failure mid-flight, both are rolled back.
3. `specscore feature new` / `specscore idea new` / `specscore task new`
   emit the two frontmatter fields in the scaffold.
4. Existing artifacts in this repo (and the two sibling repos
   `synchestra-io/specscore` and `synchestra-io/ai-plugin-specscore`) are
   migrated in one commit per repo via a one-shot script: read body
   Status, write frontmatter status, derive format from artifact type,
   strip the footer line.

Out of scope for MVP: a `format:` namespace-resolution layer (where
projects could override the spec URL per type), and any change to the
sidekick-seed format beyond renaming the existing `status:` key (it's
already there).

## Not Doing (and Why)

- Adding more frontmatter fields beyond `format:` and `status:` — keeps the principle "only what third-party tooling genuinely needs cheap access to."
- Letting projects override `format:` URLs per type — possible later via specscore.yaml, but every artifact pointing at the same canonical specscore.md URL is simpler and sufficient for v1.
- Touching the sidekick-seed format beyond ensuring `format:` is added — seeds already carry frontmatter status; the question of whether they also gain a body `**Status:**` line is a separate decision tracked in Outstanding Questions.

## Key Assumptions to Validate

| Tier | Assumption | How to validate |
|------|------------|-----------------|
| Must-be-true | Frontmatter `status:` can be kept in lockstep with body `**Status:**` via CLI + lint without race conditions in practice | Implement the dual-write inside `change-status`; run a 100-transition stress loop; confirm no drift survives `lint --fix`. |
| Must-be-true | Third-party tooling (ingitdb, generic indexers, downstream dashboards) materially prefers YAML-frontmatter parsing over Markdown body parsing | Prototype a tiny indexer that ingests this repo's artifacts; measure parse time and code complexity with frontmatter-only vs body-parsing. |
| Should-be-true | The canonical `format:` URL for each artifact type is stable enough to bake into every artifact | Check `specscore.md`'s URL history for the four spec endpoints over the last 12 months; require any rename to ship a redirect. |
| Should-be-true | Migration of existing artifacts can be done by deterministic script without human review per file | Write the migration script; dry-run against this repo; diff for spot-check; require zero non-mechanical edits. |
| Might-be-true | Sidekick-seeds should ALSO gain a body `**Status:**` line (in addition to keeping the frontmatter mirror), for human-readability symmetry with Feature / Idea | Open question; defer to seed-format Feature work. |


## SpecScore Integration

- **New Features this would create:**
  - `artifact-frontmatter` (in `synchestra-io/specscore`) — defines the
    two mandatory fields, their grammar, and the body-vs-frontmatter
    source-of-truth contract.
  - Two new lint-rule entries (probably under an existing
    `specscore-cli/spec/features/cli/spec/lint/` Feature) —
    `frontmatter-format-required` and `frontmatter-status-mirrors-body`.
- **Existing Features affected:**
  - `cli/feature/change-status` and `cli/idea/change-status` — must
    atomically rewrite frontmatter alongside body; rollback applies to
    both.
  - `cli/feature/new`, `cli/idea/new`, `cli/task/new` — scaffold both
    fields at create time.
  - Feature / Idea / Plan / seed specifications on `specscore.md` — drop
    the footer-line convention; add the two frontmatter fields as
    required.
  - `specstudio:sidekick` — emit `format:` in new seed frontmatter (the
    `status:` field is already emitted).
- **Dependencies:**
  - Coordinated change in `synchestra-io/specscore-cli` for lint rules
    and verb behavior.
  - Coordinated change in `synchestra-io/specscore` for the spec text on
    `specscore.md`.
  - One-shot migration commit per consuming repo.

## Outstanding Questions

- **Sidekick-seed body `**Status:**` symmetry.** Today seeds have
  frontmatter `status:` but no body `**Status:**` line. After this
  change, do seeds gain a body line too (for human readability and
  parity with Feature / Idea), or do they remain frontmatter-only as an
  exception? Track in a follow-on Feature for the sidekick-seed format
  itself.
- **Format-URL grammar.** Bare URLs (`https://specscore.md/feature-specification`) vs typed keys mapped via a registry (`format: feature` → URL via `specscore.yaml`). Bare URLs are simpler; typed keys allow private spec mirrors. Recommend bare URLs for MVP; revisit if anyone asks for the registry.
- **What counts as "having a status"?** Features, Ideas, Plans, seeds,
  Tasks all have a Status concept today. READMEs / index pages do not.
  The lint rule must distinguish them by `format:` value, not by file
  location. Confirm at Feature-spec time.
- **Migration ordering across repos.** The CLI lint rule landing before
  the repo migrations would temporarily break lint on every existing
  repo. Recommended sequence: ship the CLI rule disabled behind a flag,
  migrate all repos, then flip the rule on by default. Detail at
  implementation time.

---
*This document follows the https://specscore.md/idea-specification*
