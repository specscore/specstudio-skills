# `ideate` skill

The `specstudio:ideate` Claude Code skill — refines raw, vague concepts into lint-clean SpecScore Idea artifacts through structured divergent and convergent thinking.

- **Skill manifest:** [`SKILL.md`](./SKILL.md) — the canonical, machine-readable skill definition.
- **Feature specification:** [`spec/features/skills/ideate/`](../../spec/features/skills/ideate/README.md) — the SpecScore Feature this skill implements.
- **Skills index:** [`../README.md`](../README.md) — all SpecStudio skills with status and lifecycle position.

## What it does

Produces a `spec/ideas/<slug>.md` containing Problem Statement, Recommended Direction, Alternatives Considered, MVP Scope, Not Doing, Key Assumptions, and Open Questions. Hard-gated: does not invoke `specify` or any implementation skill until the Idea is lint-clean and user-approved. Publication at draft, approval, and update checkpoints follows the shared publication policy.

Triggers: `ideate`, `/ideate`, "refine this idea", "stress-test this".
