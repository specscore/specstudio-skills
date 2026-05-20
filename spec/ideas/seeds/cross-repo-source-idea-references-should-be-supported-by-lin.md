---
type: sidekick-seed
slug: cross-repo-source-idea-references-should-be-supported-by-lin
captured_at: 2026-05-20T18:19:40Z
captured_by: user
captured_during: null
trigger: explicit
status: queued
synchestra_task: null
---

# Cross-repo Source-Idea references should be supported by lint

## Observation

`specscore spec lint` rule `idea-feature-cross-reference` requires a Feature's `**Source Ideas:** <slug>` value to resolve to a local `spec/ideas/<slug>.md`. Cross-repo references — Feature in repo A sourced from an Idea in repo B — are unsupported.

## Concrete dogfood evidence

2026-05-20 session: specifying `cli/idea/relocate` in `specscore-cli`, sourced from `idea-skills-destination-resolution` Idea in `specstudio-skills`. Lint rejected the cross-repo slug. Workaround: set `**Source Ideas:** —` and document the link in the Interaction section via full GitHub URL — losing the structural body-metadata link.

## Suggested direction

Allow two Source Ideas forms:
- Local slug (current): resolves in same repo.
- Cross-repo URL: `https://github.com/<org>/<repo>/blob/<branch>/spec/ideas/<slug>.md`, resolved best-effort (lint warns if unreachable; never errors).

## Affected surfaces

- `specscore-cli` lint rule `idea-feature-cross-reference`
- `specscore/specscore` Source Ideas convention text
- Possibly `specstudio:specify` (cross-repo destination-aware writing)

---
*This document follows the https://specscore.md/sidekick-seed-specification*
