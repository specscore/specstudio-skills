# Feature: Ship Skill

> [SpecScore.**Studio**](https://specscore.studio): | [Explore](https://specscore.studio/app/github.com/synchestra-io/specstudio-skills/spec/features/skills/ship?op=explore) | [Edit](https://specscore.studio/app/github.com/synchestra-io/specstudio-skills/spec/features/skills/ship?op=edit) | [Ask question](https://specscore.studio/app/github.com/synchestra-io/specstudio-skills/spec/features/skills/ship?op=ask) | [Request change](https://specscore.studio/app/github.com/synchestra-io/specstudio-skills/spec/features/skills/ship?op=request-change) |

**Status:** Draft

## Summary

The `ship` skill is intended to run the pre-launch checklist for a Feature, gated on `verify` and `review` having passed, and to coordinate the deploy or release. The skill is not yet defined — no Idea has been written, and the scope below is a placeholder to be refined via `specstudio:ideate`.

## Problem

The handoff from "code merged" to "feature actually shipped" is where deploy-time mistakes happen: missing migration, missing config, missing monitoring, missing rollback plan. Generic shipping skills (`agent-skills:shipping-and-launch`, `gh-deploy`) cover the mechanics but not the spec-aware checks: did every AC's `verify` report come back green? Did every `review` finding get resolved or explicitly waived? Was a `recap` produced and accepted?

A spec-aware `ship` skill exists to enforce those gates before any deploy action is taken.

## Behavior

The skill is *intended* to: load the Feature, the most recent `verify` report, the `review` findings, and the `recap` artifact, then run a structured pre-launch checklist. Hard gates: all ACs verified green, all blocker-severity review findings resolved, recap accepted. Soft gates: monitoring, rollback plan, feature-flag status. On all gates passing, the skill proceeds with (or hands off to) the deploy mechanics.

This is intent, not contract. The actual scope, gate definitions, and deploy-coordination semantics need to be defined through `specstudio:ideate`.

## Acceptance Criteria

Not defined yet.

## Open Questions

- Acceptance criteria not yet defined for this feature.
- Does `ship` actually deploy, or only run the checklist and hand off to a separate deploy tool (e.g., `gh-deploy`)?
- How does `ship` handle Features that ship behind a feature flag — is the flag flip the "ship" event, or is it a separate post-ship action?
- What is the Feature's status transition triggered by `ship` — `In Progress` → `Stable`, or does that happen elsewhere?
- How does `ship` handle multi-Feature releases where some Features are green and others are not?
- Should `ship` enforce a recap, or treat it as optional?
- How does `ship` interact with rollback — does it track its own rollback plan, or rely on the deploy tool's?

---
*This document follows the https://specscore.md/feature-specification*
