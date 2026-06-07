---
captured_by: user
status: queued
---

# Idea skills should pick destination repo, not default to cwd

## Observation

In multi-repo SpecScore workspaces, Idea-generation skills — `specstudio:ideate`, `specstudio:sidekick`, and `agent-skills:idea-refine` — write to `spec/ideas/` of whatever repo `cwd` happens to be. No inference, no prompt, no audit of where the content's natural home is.

## Concrete dogfood evidence

The `artifact-frontmatter-convention` Idea was captured into `specscore/specstudio-skills` (commit `c4114cb`, 2026-05-19) even though it belongs in `specscore/specscore`. Relocated 2026-05-20 (commits `160ae03` in skills, `7e32851` in specscore). The relocate also required rewriting `specscore/*` org references and disambiguating "this repo" wording — pure rework caused by the wrong initial destination.

## Promoted

Full Idea at [`../idea-skills-destination-resolution.md`](../idea-skills-destination-resolution.md). Recommended Direction: silent-when-unambiguous inference (sibling-dir scan + keyword scoring) + a companion `specstudio:relocate-idea` recovery skill + opt-in mismatch logging to improve the inference heuristic over time.

---
*This document follows the https://specscore.md/sidekick-seed-specification*
