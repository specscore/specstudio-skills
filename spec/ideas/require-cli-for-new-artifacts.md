# Idea: Require the specscore CLI for New Artifacts — Drop Fallbacks and Embedded Templates

**Status:** Draft
**Date:** 2026-06-05
**Owner:** alexander.trakhimenok
**Promotes To:** —
**Supersedes:** —
**Related Ideas:** extends:cli-detection-convention

## Problem Statement

How might we make the specscore CLI the single source of truth for creating new artifacts, so producer skills stop carrying duplicated schema templates and divergent direct-write fallbacks?

## Context

Triggered by a maintainer questioning why producer skills (ideate, sidekick, specify, plan, init) each ship a full authoritative schema block AND a direct-write fallback that re-implements what 'specscore idea new' already does. The schema is duplicated between every skill and the CLI, so the two can silently drift. The current shared/cli-detection.md convention deliberately classes ideate and sidekick as Optional (CLI preferred, direct-write fallback on exit 127). This idea revises that convention for the artifact-CREATION path.

## Recommended Direction

Promote new-artifact creation in every producer skill from Optional-class to CLI-required. The CLI scaffold becomes the single source of truth for artifact structure; skills delete their embedded authoritative schema blocks and direct-write fallbacks, retaining only instructions for how to FILL the scaffold's comment-prompt sections via Edit. On exit 127 (CLI absent) a producer stops, points the user at /specscore:install, and offers install-then-retry rather than hand-writing a divergent file. This kills template/CLI drift, shortens every producer skill, and guarantees a known-good lint baseline.

The template-sourcing problem moves down a layer, into the CLI. The CLI fetches the canonical scaffold template from specscore.md (one published source of truth, updatable without shipping a new binary), and falls back to a copy embedded in the binary when there is no network. Skills stay oblivious to either path — they only ever call `specscore <noun> new`. This keeps the offline case working (the embedded template), keeps the canonical template centrally managed (the web copy), and keeps the skills free of any template knowledge at all.

The canonical templates are addressable at a stable per-type URL — `https://specscore.md/new/idea`, `https://specscore.md/new/feature`, and so on — which serves two roles at once: it is where the CLI fetches, and a human-clickable reference. Each producer skill MAY carry a one-line read-only pointer to its artifact's template URL in place of the deleted inline schema, so an author can see the canonical shape on demand without the skill holding a copy that drifts. The pointer is reference-only: artifact creation still runs exclusively through `specscore <noun> new`; the skill never fetches the URL and writes the file itself — that would re-introduce the rejected skills-download path.

## Alternatives Considered

- **Keep the Optional class with a real direct-write fallback (status quo).** Rejected. Two creation paths means two things to maintain, and the embedded schema duplicates the CLI's own output — they drift the moment either side changes. The fallback only "helps" the rare CLI-absent user, and it helps them by producing a *divergent* artifact that lint may later reject anyway.
- **Require the CLI but keep the schema blocks as non-authoritative reference.** Rejected for MVP. A "reference" copy of the schema is still a copy: it will drift, and authors will trust it because it is right there in the skill. If authors need to see the shape, they run the scaffold once — the generated file *is* the reference, and it cannot lie about what the CLI produces.
- **Auto-generate the schema blocks into each skill from the CLI at build time (codegen).** Keeps templates in sync without deleting them, so authors keep an inline reference. Rejected for MVP as heavier machinery (a build step in the skills repo) than the problem warrants; revisit only if removing the blocks entirely proves to hurt authoring. Captured in Open Questions.
- **Skills download canonical templates from specscore.md directly (no CLI).** Rejected as a *skills-level* design, though it inspired the chosen approach. Fetching a bare template into the skill leaves the skill re-implementing everything the CLI does *around* the template — updating `spec/ideas/README.md`, running lint-fix — so the fallback logic this idea deletes simply returns minus the schema. It also adds a per-creation network dependency and a new drift axis (web template vs. the installed CLI's lint rules). The same web-template benefit is captured *inside* the CLI instead (see Recommended Direction): the CLI downloads the template and embeds an offline fallback, so skills stay template-free and the index/lint-fix logic stays in one place.
- **Install-then-fall-back: try to install, and if install fails, hand-write the file anyway.** Rejected. It re-introduces the divergent-artifact path through the back door and defeats the "guaranteed known-good baseline" goal. A hard stop with a clear install path is more honest than a silent degraded result.

## MVP Scope

A convention update plus representative conversion: revise shared/cli-detection.md to add a 'CLI-required for artifact creation, install-then-retry on 127' rule, then apply it to ideate (Ideas), sidekick (seeds), and specify (Features) — deleting their fallback + schema sections and wiring install-then-retry. Done when the three skills lint clean and a maintainer agrees no section the skills must fill is lost by removing the embedded schemas. plan and init follow once CLI scaffold verbs for Plans and project bootstrap are confirmed.

## Not Doing (and Why)

- Removing the direct-write fallback from EDIT paths — this idea only governs NEW artifact creation; editing existing artifacts via Edit stays
- Rewriting all producer skills in one pass — MVP converts three representatives; plan/init follow
- Adding new CLI scaffold verbs — this idea assumes/depends on them existing; the CLI work is a separate dependency
- Changing exit-code semantics — relies on existing 127-absent and exit-8-too-old behavior from cli-detection.md
- Having the skill fetch the template URL and write the file — the per-type URL is a read-only reference for humans; creation always goes through the CLI

## Key Assumptions to Validate

| Tier | Assumption | How to validate |
|------|------------|-----------------|
| Must-be-true | A CLI scaffold verb exists (or is committed) for every artifact type a producer creates — Idea (`idea new`, confirmed), seed (sidekick), Feature (specify), Plan (plan), project bootstrap (init) | Enumerate each producer's created artifact; for each, confirm a `specscore <noun> new`-style verb exists and emits a lint-clean skeleton. Where none exists, this idea blocks on CLI work |
| Must-be-true | The CLI scaffold's section set is a strict superset of what the skill must fill, so deleting the embedded schema loses no information the author needs | Diff each skill's authoritative schema block against the CLI scaffold output; confirm every section the skill instructs the author to fill appears as a comment-prompt in the scaffold |
| Must-be-true | "install-then-retry" is executable inside a single skill/agent session via `/specscore:install` without leaving partial artifact state | Run a producer in a CLI-absent sandbox; confirm it stops cleanly (no file written), installs, and resumes the same scaffold call to completion |
| Should-be-true | Forcing the CLI does not strand offline / air-gapped users, because the binary carries an embedded template fallback once installed (only the *install* step needs network, not each creation) | Install in a sandbox, drop the network, create an artifact; confirm the embedded-template path produces a lint-clean file |
| Should-be-true | The web-published template and the CLI's embedded fallback (and the CLI's own lint rules) stay compatible, so a newer web template never produces a file an installed CLI then rejects | Define a versioning/compatibility contract between specscore.md templates and CLI versions; test creating with an old CLI against a newer published template |
| Should-be-true | `init` can require the CLI despite being the project-bootstrap entry point, without making first-run worse than today's AI-agent fallback | Walk the cold-start path: brand-new repo, no CLI; confirm install-then-retry is no worse than the current fallback bootstrap |
| Might-be-true | Authors will not miss an inline schema reference once it is removed, because running the scaffold once — or opening the per-type template URL — shows the shape | Convert the three MVP skills; ask a maintainer to author an artifact with the converted skills and report whether the missing inline schema hurt |


## SpecScore Integration

- **New Features this would create:** Likely none new — this revises the existing `cli-detection-convention` (the *Optional* class's fallback disappears for artifact creation) and edits the producer skills. A new shared rule may live in `skills/shared/cli-detection.md` rather than as its own Feature. Confirm at spec time.
- **Existing Features affected:** `cli-detection-convention` (response table gains a "CLI-required for creation, install-then-retry on 127" rule); producer skills `ideate`, `sidekick`, `specify`, `plan`, `init` (fallback + embedded schema sections deleted; install-then-retry wired in).
- **Dependencies:** (1) CLI scaffold verbs for every artifact type the producers create (Idea confirmed via `idea new`; seed/Feature/Plan/bootstrap must be confirmed or built in the `specscore` CLI). (2) **CLI template-sourcing** — the CLI fetches the canonical scaffold template from specscore.md with an embedded offline fallback, plus a versioning contract between published templates and CLI versions. Both are cross-repo work in `specscore` / `specscore.md` and gate the full rollout. Aligns with the existing `shared/*.md` convention (`cli-detection.md`, `path-conventions.md`).

## Open Questions

- Where on specscore.md do canonical templates live, and what is the fetch contract? **Proposed:** a stable per-type URL `https://specscore.md/new/<artifact-type>` (e.g. `/new/idea`, `/new/feature`) used both as the CLI fetch endpoint and a human reference link. Still open: caching, content negotiation (raw template vs. rendered page), versioning, auth (likely none).
- What is the compatibility/versioning rule between a web-published template and an installed CLI's lint rules, so a newer template never yields a file the local CLI rejects? Does the CLI pin to a template version matching its own?
- Does `init` requiring the CLI create an unacceptable cold-start (project bootstrap before any tooling exists), or is install-then-retry good enough there?
- Should the CLI template-sourcing work be tracked as a `specscore` / `specscore.md` sidekick seed (as `cli-detection-convention` did for its CLI dependency)?

---
*This document follows the https://specscore.md/idea-specification*
