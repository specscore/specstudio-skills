# Feature: Verify Skill

> [SpecScore.**Studio**](https://specscore.studio): | [Explore](https://specscore.studio/app/p/github.com/synchestra-io/specstudio-skills/spec/features/skills/verify?op=explore) | [Edit](https://specscore.studio/app/p/github.com/synchestra-io/specstudio-skills/spec/features/skills/verify?op=edit) | [Ask question](https://specscore.studio/app/p/github.com/synchestra-io/specstudio-skills/spec/features/skills/verify?op=ask) | [Request change](https://specscore.studio/app/p/github.com/synchestra-io/specstudio-skills/spec/features/skills/verify?op=request-change) |

**Status:** Draft

## Summary

The `verify` skill is intended to run the Rehearse test scenarios attached to a Feature and report per-AC pass/fail coverage. The skill is not yet defined — no Idea has been written, and the scope below is a placeholder to be refined via `specstudio:ideate`.

## Problem

A SpecScore Feature is a contract; Rehearse scenarios are the executable proof that the contract is satisfied. Today, running those scenarios and mapping their results back to the Feature's AC IDs is a manual exercise — there is no skill that consumes a Feature, runs its scenarios, and produces a structured per-AC report.

Without that step, `recap`, `review`, and `ship` cannot be honestly gated on test results, because nothing produces the standard report they would gate on.

## Behavior

The skill is *intended* to: load a Feature's `_tests/` scenarios (or the equivalent Rehearse layout), execute them, and emit a structured report listing each AC ID with its pass/fail status, evidence (test output, screenshots, etc.), and any unmapped scenarios or unmapped ACs. The report should be machine-readable enough that downstream skills (`recap`, `review`, `ship`) can consume it as a gate.

This is intent, not contract. The actual scope, output format, gate semantics, and Rehearse coupling need to be defined through `specstudio:ideate`.

## Acceptance Criteria

Not defined yet.

## Outstanding Questions

- Acceptance criteria not yet defined for this feature.
- What is the canonical report format — JSON, Markdown, both?
- How does `verify` handle ACs with no associated Rehearse scenario — error, warn, or info?
- How does `verify` handle scenarios that don't reference a specific AC ID?
- Does `verify` re-run all scenarios every time, or use a cache keyed on code/spec changes?
- Should `verify` be capable of running on a single AC in isolation, or only on full Features?
- How does `verify` interact with existing test runners (jest, pytest, etc.) — wrap them, replace them, or sit alongside them?

---
*This document follows the https://specscore.md/feature-specification*
