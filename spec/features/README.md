---
format: https://specscore.md/features-index-specification
---

# SpecStudio Features

> [SpecScore.**Studio**](https://specscore.studio): | [Explore](https://specscore.studio/app/p/github.com/specscore/specstudio-skills/spec/features?op=explore) | [Edit](https://specscore.studio/app/p/github.com/specscore/specstudio-skills/spec/features?op=edit) | [Ask question](https://specscore.studio/app/p/github.com/specscore/specstudio-skills/spec/features?op=ask) | [Request change](https://specscore.studio/app/p/github.com/specscore/specstudio-skills/spec/features?op=request-change) |

Feature specifications for the SpecStudio plugin. This index lists every top-level SpecScore Feature in this repository.

## Contents

| Feature | Status | Description |
|---------|--------|-------------|
| [`skills/`](skills/README.md) | Approved | Umbrella feature for the per-skill sub-features that specify each Claude Code skill's purpose, gates, inputs, outputs, and lifecycle position. |
| [sidekick-capture](sidekick-capture/README.md) | Approved | Phase 0 of the [`sidekick-ideas`](../ideas/sidekick-ideas.md) Idea: the `specstudio:sidekick` skill, shared capture directive, seed artifact format, lint rule, and `sidekick-idea.captured` event. Establishes the write-and-continue capture loop for sideline ideas without any deliberation or auto-promotion. |
| [sidekick-consilium](sidekick-consilium/README.md) | Approved | Phase 1 of the [`sidekick-ideas`](../ideas/sidekick-ideas.md) Idea: the `specstudio:consilium` skill that drains captured *sidekick* (sideline-idea) seeds and produces deterministic verdicts via a 5-stage pipeline (CLI gather → researcher → 9-role parallel expert panel → CLI arbiter → scribe). Per-project configurable roster + gate via `specscore.yaml`. |
| [third-party-integration](third-party-integration/README.md) | Defines the contract for integrating third-party agent skills with SpecScore artifacts via three shapes (Producer, Reviewer, Capability), with explicit non-goal for Shape-3 third-party Producers. |
| [dogfood-test](dogfood-test/README.md) | Deprecated | TODO: Add description. |
| [reviewer-gates](reviewer-gates/README.md) | Approved | Defines the canonical reviewer-gates contract: per-stage reviewer lists scoped under a `gates:` block in `specscore.yaml`, with a `type:` discriminator and type-specific fields per reviewer entry. Pins the schema, dispatch semantics, and verdict contract for the MVP type set (`ai`, `human`), and wires `specstudio:specify` as the first consumer — replacing its built-in reviewer dispatch and User Review Gate with the new typed-gate model. Carves the reviewer parts of `third-party-integration` out into this Feature. |
| [issue-artifact-type](issue-artifact-type/README.md) | Stable | TODO: Add description. |
| [flexible-lifecycle-flows](flexible-lifecycle-flows/README.md) | Approved | TODO: Add description. |
| [score-command](score-command/README.md) | Implementing | The single `specstudio:score` skill from the [`manual-review-and-score-commands`](../ideas/manual-review-and-score-commands.md) Idea: manually re-invokes the configured [`reviewer-gates`](reviewer-gates/README.md) dispatch pipeline against one or more SpecScore artifacts outside the producer-skill exit context. Skips `type: human` entries; supports single-artifact, multi-artifact, and recursive tree-wide invocation. Verdict-only contract here; grade output + `--save` + `--badge` are the next increment of the same command, gated on the `reviewer-gates` grade work. |
| [change-publication-policy](change-publication-policy/README.md) | Draft | SpecStudio skills consume shared publication policy to decide whether to edit only, stage, commit, and push at artifact lifecycle events and command milestones. |
| [implement-execution-topology](implement-execution-topology/README.md) | Approved | Transport-agnostic contract for how implement executes parallel agent work: branch roles, the gated transitions between them, and how a topology is selected. Concrete per-scenario realization lives in sub-Features. |
| [approval-autonomy](approval-autonomy/README.md) | Approved | The implement-autonomy layer: implement fires the event-keyed gate-point events (implementation.pre_commit / pre_push) at per-batch checkpoints so commit-autonomous/push-gated is pure gate config, and owns the non-reviewer execution dynamics — commit cadence, anomaly-halts, explicit re-arm, and the cumulative review fed to the push gate. |
| [cli-detection-convention](cli-detection-convention/README.md) | Approved | One mechanism for skills to detect the specscore CLI — call and branch on exit code — with a per-skill-class response policy, plus a CLI-required artifact-creation mandate for producer skills (no embedded schemas; install-then-retry, no direct-write fallback). |
| [seed-to-idea-promotion](seed-to-idea-promotion/README.md) | Approved | Promote a sidekick seed into a lint-clean Idea via a deterministic specscore idea promote verb (same-repo git mv + body transform + back-link reconcile; cross-repo keep+promoted), with a skill-side consilium-offer handshake for unreviewed manually-picked seeds. |
| [detached-background-implement](detached-background-implement/README.md) | Approved | TODO: Add description. |
| [autopilot](autopilot/README.md) | Approved | A thin orchestrator skill that drives an idea from any pipeline entry point (a cold raw prompt included) to one open MVP pull request, pausing only at a single Idea checkpoint and halting on genuine anomalies. |

### skills

The `skills` feature groups one sub-feature per Claude Code skill in the plugin. Today it covers `ideate`, `specify`, `plan`, `implement`, `verify`, `recap`, `review`, and `ship` — all `Draft` until each is refined via `specstudio:ideate` and promoted via `specstudio:specify`. Implementation maturity (which skills actually ship today) is tracked separately in [`skills/README.md`](../../skills/README.md).

## Open Questions

- None at this time.

---
*This document follows the https://specscore.md/features-index-specification*
