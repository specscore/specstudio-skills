# Path Conventions: `spec/` vs. `docs/`

**Status:** Canonical
**Applies to:** All SpecScore / Synchestra skills in this plugin

## The Rule

| Tree | Purpose | Audience | Format | Tooling |
|---|---|---|---|---|
| `spec/` | Machine-addressable SpecScore artifacts (ideas, features, requirements, acceptance criteria, plans) | Tooling (`specscore spec lint`, Rehearse, Synchestra agents) and contributors editing specs | Typed — title-prefix dispatch (`# Idea: …` / `# Feature: …` / `# Plan: …`), bold-prefixed body metadata (`**Status:**`, `**Date:**`, `**Owner:**`, …), fixed section schema | Lintable, schema-validated, stable IDs (slug = filename), bidirectional links |
| `docs/` | User-facing prose | Human readers of the project (adopters, reviewers, contributors) | Freeform markdown | Rendered as-is; no schema constraints |

## Canonical Paths

| Artifact | Path | Filename |
|---|---|---|
| Idea | `spec/ideas/<slug>.md` | single file |
| Feature | `spec/features/<slug>/README.md` | directory |
| Plan | `spec/plans/<slug>/README.md` (+ task files) | directory |
| Feature visual asset (mockup, diagram) | `spec/features/<slug>/assets/*` | any |
| Rehearse scenario (optional) | `spec/features/<slug>/_tests/<scenario>.md` | per scenario |
| User-facing guide / tutorial | `docs/<topic>.md` | single file |
| ADR / design journal | `docs/adr/<YYYY-MM-DD>-<slug>.md` | single file |
| Internal ideation notes (pre-artifact) | `docs/ideas/<slug>.md` | **AVOID** — promote to `spec/ideas/` |

## Rules

1. **Dates go in body metadata, not filenames.** `spec/ideas/payment-fraud-signals.md` with `**Date:** 2026-04-14` line in the body — not `2026-04-14-payment-fraud-signals.md`.
2. **Slugs are stable IDs.** The canonical id is the filename without `.md`. There is no separate `id` field. Do not rename once promoted/approved. If a new scope invalidates existing acceptance criteria, create a successor with a new slug and set `**Supersedes:**` in the successor's body metadata.
3. **Revise Features in place.** Git history is the record of evolution. `**Supersedes:**` is reserved for wholesale replacement, not iteration.
4. **Promotion links resolve inside `spec/`.** `**Promotes To:** checkout-v2` resolves to `spec/features/checkout-v2/README.md` — a single tree move does not cross a boundary.
5. **Never put typed artifacts in `docs/`.** If it has a schema, it belongs in `spec/`. If someone proposes `docs/features/`, that's a mistake.
6. **Never put freeform prose in `spec/`.** Guides, tutorials, README-style narration → `docs/`. A `spec/features/foo/NOTES.md` that's just prose is a smell.
7. **Skills auto-bootstrap their canonical sub-tree when missing.** When `specstudio:ideate` is invoked in a project without `spec/ideas/`, the skill creates the directory and a lint-clean `spec/ideas/README.md` index before writing the first Idea. The same rule applies to `specstudio:specify` for `spec/features/` and to `specstudio:plan` for `spec/plans/`. Bootstrapping MUST be explicit — the skill tells the user it created the path; never silent.
8. **Skills publish created files through policy.** When a skill creates or first-writes any file under `spec/`, it MUST add that path to the current publication manifest and apply the shared [publication-policy protocol](publication-policy.md) at the relevant event or milestone. Staging, committing, and pushing are policy actions with manifest and branch safety; they are not hard-coded by path convention.

## Overrides of Upstream Skills

When porting logic from upstream skills, the following path conventions are **deliberately changed**:

| Upstream | Upstream path | SpecScore path | Reason |
|---|---|---|---|
| `addyosmani/agent-skills` `idea-refine` | `docs/ideas/<name>.md` (optional save) | `spec/ideas/<slug>.md` (mandatory, lintable) | Ideas are typed, not prose; unsaved ideation is waste |
| `obra/superpowers` `brainstorming` | `docs/superpowers/specs/YYYY-MM-DD-<topic>-design.md` | `spec/features/<slug>/README.md` | Dates belong in body metadata; slugs are stable IDs; Features have sub-artifacts |
| `obra/superpowers` `brainstorming` visual companion | `.superpowers/brainstorm/<session-id>/…` | `spec/features/<slug>/assets/` (when persisted) | Assets live with their Feature, not in a scratch dir |

## Enforcement

- `specscore spec lint` refuses SpecScore artifacts outside `spec/`.
- `specscore spec lint` warns on prose-shaped files inside `spec/` (missing required title prefix, missing required body-metadata lines, missing required sections).
- CI integration: PRs creating new top-level directories under `spec/` require review.
