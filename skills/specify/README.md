# `specify` skill

The `specstudio:specify` Claude Code skill — turns an approved SpecScore Idea, or a clear buildable intent, into a lint-clean SpecScore Feature with `Given / When / Then` acceptance criteria.

- **Skill manifest:** [`SKILL.md`](./SKILL.md) — the canonical, machine-readable skill definition.
- **Feature specification:** [`spec/features/skills/specify/`](../../spec/features/skills/specify/README.md) — the SpecScore Feature this skill implements.
- **Skills index:** [`../README.md`](../README.md) — all SpecStudio skills with status and lifecycle position.

## What it does

Produces a `spec/features/<slug>/` directory containing the Feature README, requirements, acceptance criteria in G/W/T form, and (optionally) Rehearse test scaffolding. Hard-gated: no plans, code, or implementation-skill invocations until `specscore spec lint` passes and the user explicitly approves. Publication at specified, approved, and update checkpoints follows the shared publication policy.

`ideate` is skippable when the problem and scope are already obvious — `specify` accepts a clear buildable intent directly. If the intent isn't actually clear, the skill pushes back rather than producing a low-quality Feature.

Triggers: `specify`, `/specify`, "spec this out", or the `idea.approved` event.
