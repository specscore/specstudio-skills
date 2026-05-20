---
type: sidekick-seed
slug: idea-skills-should-pick-destination-repo-not-default-to-cwd
captured_at: 2026-05-20T16:42:56Z
captured_by: user
captured_during: null
trigger: explicit
status: queued
synchestra_task: null
---

# Idea skills should pick destination repo, not default to cwd

## Observation

In multi-repo SpecScore workspaces (e.g. `specscore/specscore`, `specscore/specscore-cli`, `specscore/specstudio-skills`, `specscore/ai-plugin-specscore`, …), Idea-generation skills — `specstudio:ideate`, `specstudio:sidekick`, and `agent-skills:idea-refine` — currently write the new Idea/seed to `spec/ideas/` of whatever repo `cwd` happens to be. There is no inference, no prompt, no audit of where the content's natural home is.

## Concrete dogfood evidence

The `artifact-frontmatter-convention` Idea was captured into `specscore/specstudio-skills` during the 2026-05-19 session (commit `c4114cb`) even though the Idea is plainly about a SpecScore-wide convention that belongs in `specscore/specscore` alongside the spec text. It sat misfiled for a day before being relocated (commits `160ae03` in skills, `7e32851` in specscore on 2026-05-20). The relocate also required rewriting `synchestra-io/*` org references and disambiguating "this repo" wording — pure rework caused by the wrong initial destination.

## Suggested behavior

1. **Infer destination from content.** The skill should heuristically map the Idea's scope to a candidate repo:
   - Convention or spec text → `specscore/specscore`
   - CLI behavior → `specscore/specscore-cli`
   - A specific skill's behavior → that skill's host repo (this repo for specstudio skills)
   - Plugin/marketplace concerns → `specscore/ai-plugin-specscore`
   - Adopter/consumer specifics → the consumer repo
2. **Ask when ambiguous.** When two or more repos are plausible homes, the skill MUST surface the options and ask the user before writing. Do not silently default to cwd.
3. **Confirm on first capture per session.** Even when one repo seems obvious, surface the chosen destination in the success line so the user can catch a misfile immediately rather than days later.

## Affected skills

- `specstudio:sidekick` (this repo)
- `specstudio:ideate` (this repo)
- `agent-skills:idea-refine` (agent-skills plugin)

Multi-repo destination resolution is a cross-skill concern; consider extracting it into a `shared/` helper rather than duplicating per skill.

## Out of scope for this seed

- The exact heuristic ruleset (will need design).
- How to discover sibling repos on disk (env var? config in `specscore.yaml`? scan parent dir?).
- Whether the same problem applies to Feature/Plan/Task skills (likely yes — separate seeds).

---
*This document follows the https://specscore.md/sidekick-seed-specification*
