# SpecStudio Plans

> [SpecScore.**Studio**](https://specscore.studio): | [Explore](https://specscore.studio/app/p/github.com/specscore/specstudio-skills/spec/plans?op=explore) | [Edit](https://specscore.studio/app/p/github.com/specscore/specstudio-skills/spec/plans?op=edit) | [Ask question](https://specscore.studio/app/p/github.com/specscore/specstudio-skills/spec/plans?op=ask) | [Request change](https://specscore.studio/app/p/github.com/specscore/specstudio-skills/spec/plans?op=request-change) |

Implementation plans for Approved SpecScore Features. Each plan breaks a Feature's REQs and ACs into ordered, verifiable tasks that an engineer (human or agent) can execute task-by-task.

Plans here are authored by `superpowers:writing-plans` today and will be authored by `specstudio:plan` once that skill ships ([Approved Idea](../ideas/specstudio-plan-skill.md)).

## Contents

| Plan | Feature | Notes |
|---|---|---|
| [sidekick-capture](sidekick-capture.md) | [sidekick-capture](../features/sidekick-capture/README.md) | Phase 0 of the sidekick-ideas Idea: capture infrastructure. 9 tasks. |
| [sidekick-capture-lint-rule-companion](sidekick-capture-lint-rule-companion.md) | — | Cross-repo companion stub: the `sidekick-seed` lint rule lives in `specscore/specscore-cli`. |
| [sidekick-consilium](sidekick-consilium.md) | [sidekick-consilium](../features/sidekick-consilium/README.md) | Phase 1 of the sidekick-ideas Idea: the consilium worker. 11 tasks. Cross-repo dependencies tracked via companion stubs (arbiter + task type) authored in Task 1. |
| [sidekick-consilium-arbiter-companion](sidekick-consilium-arbiter-companion.md) | — | Cross-repo companion stub: the `specscore consilium verdict` subcommand lives in `specscore/specscore-cli`. |
| [sidekick-consilium-task-companion](sidekick-consilium-task-companion.md) | — | Cross-repo companion stub: the `consilium-review` task type lives in `specscore/synchestra`. |
| [reviewer-gates](reviewer-gates.md) | [reviewer-gates](../features/reviewer-gates/README.md) | Typed per-stage reviewer gates: schema + load-time validator + runner + `specstudio:specify` wiring + carve-out of legacy reviewer parts from `third-party-integration` and `specify`. 7 tasks covering 16 ACs, 0 deferred. |
| [issue-artifact-type](issue-artifact-type.md) | [issue-artifact-type](../features/issue-artifact-type/README.md) | Introduces the `issue` top-level artifact (parallel to Ideas/Features/Plans), `I-` lint rule namespace, dual-location (root + Feature-scoped), four-state lifecycle. 11 tasks covering 33 ACs, 0 deferred. First plan from `specstudio:plan` skill. |

## Open Questions

- None at this time.

---
*This document follows the https://specscore.md/plans-index-specification*
