# SpecStudio Features

> [SpecScore.**Studio**](https://specscore.studio): | [Explore](https://specscore.studio/app/p/github.com/synchestra-io/specstudio-skills/spec/features?op=explore) | [Edit](https://specscore.studio/app/p/github.com/synchestra-io/specstudio-skills/spec/features?op=edit) | [Ask question](https://specscore.studio/app/p/github.com/synchestra-io/specstudio-skills/spec/features?op=ask) | [Request change](https://specscore.studio/app/p/github.com/synchestra-io/specstudio-skills/spec/features?op=request-change) |

Feature specifications for the SpecStudio plugin. This index lists every top-level SpecScore Feature in this repository.

## Contents

| Feature | Status | Description |
|---------|--------|-------------|
| [`skills/`](skills/README.md) | Approved | Umbrella feature for the per-skill sub-features that specify each Claude Code skill's purpose, gates, inputs, outputs, and lifecycle position. |
| [sidekick-capture](sidekick-capture/README.md) | Approved | Phase 0 of the [`sidekick-ideas`](../ideas/sidekick-ideas.md) Idea: the `specstudio:sidekick` skill, shared capture directive, seed artifact format, lint rule, and `sidekick-idea.captured` event. Establishes the write-and-continue capture loop for sideline ideas without any deliberation or auto-promotion. |
| [sidekick-consilium](sidekick-consilium/README.md) | Approved | Phase 1 of the [`sidekick-ideas`](../ideas/sidekick-ideas.md) Idea: the `specstudio:consilium` skill that drains captured *sidekick* (sideline-idea) seeds and produces deterministic verdicts via a 5-stage pipeline (CLI gather → researcher → 9-role parallel expert panel → CLI arbiter → scribe). Per-project configurable roster + gate via `specscore.yaml`. |
| [third-party-integration](third-party-integration/README.md) | Defines the contract for integrating third-party agent skills with SpecScore artifacts via three shapes (Producer, Reviewer, Capability), with explicit non-goal for Shape-3 third-party Producers. |
| [dogfood-test](dogfood-test/README.md) | Deprecated | TODO: Add description. |

### skills

The `skills` feature groups one sub-feature per Claude Code skill in the plugin. Today it covers `ideate`, `specify`, `plan`, `implement`, `verify`, `recap`, `review`, and `ship` — all `Draft` until each is refined via `specstudio:ideate` and promoted via `specstudio:specify`. Implementation maturity (which skills actually ship today) is tracked separately in [`skills/README.md`](../../skills/README.md).

## Open Questions

- None at this time.

---
*This document follows the https://specscore.md/features-index-specification*
