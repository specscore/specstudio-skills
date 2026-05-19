# Feature: Implement Skill

> [SpecScore.**Studio**](https://specscore.studio): | [Explore](https://specscore.studio/app/p/github.com/synchestra-io/specstudio-skills/spec/features/skills/implement?op=explore) | [Edit](https://specscore.studio/app/p/github.com/synchestra-io/specstudio-skills/spec/features/skills/implement?op=edit) | [Ask question](https://specscore.studio/app/p/github.com/synchestra-io/specstudio-skills/spec/features/skills/implement?op=ask) | [Request change](https://specscore.studio/app/p/github.com/synchestra-io/specstudio-skills/spec/features/skills/implement?op=request-change) |

**Status:** Draft

## Summary

The `implement` skill is intended to consume a lint-clean Plan and produce source code one task at a time, with each commit traceable to the AC IDs the task claims to satisfy. The skill is not yet defined — no Idea has been written, and the scope below is a placeholder to be refined via `specstudio:ideate`.

The name `implement` is chosen deliberately over the earlier working name `build`. In software, "build" connotes compile/package steps on already-written source (`npm run build`, build artifacts, build pipelines). What this skill does is *write* the source — i.e., implement it.

## Problem

`specstudio:plan` produces a Plan whose tasks reference specific AC IDs. The next step — actually writing the code that satisfies those ACs — is currently done with general-purpose implementation skills (`agent-skills:incremental-implementation`, `superpowers:test-driven-development`) that have no awareness of the AC linkage. As a result, commits don't trace back to acceptance criteria, partial implementations are hard to detect, and the spec↔code coherence loop breaks at the implementation step.

An `implement` skill with explicit AC awareness is meant to close that loop, but its precise contract has not been designed.

## Behavior

The skill is *intended* to: consume one task from a Plan at a time, produce a focused commit (or series of commits) that satisfies the task's referenced ACs, run the corresponding Rehearse scenarios for those ACs, and update the Plan's task status. Each commit message should reference the satisfied AC IDs to preserve the traceability chain.

This is intent, not contract. The actual scope, gates, inputs/outputs, and AC-mapping rules need to be defined through `specstudio:ideate`.

## Acceptance Criteria

Not defined yet.

## Outstanding Questions

- Acceptance criteria not yet defined for this feature.
- Does `implement` write code directly, or does it dispatch to a sub-agent / Synchestra runner that writes the code?
- What's the gate condition for a task being "done" — Rehearse scenarios pass, code review approved, or both?
- How does `implement` handle a task whose AC is ambiguous in execution — push back to the user, or attempt and let `verify` catch it?
- Does `implement` operate on one task at a time strictly, or can it batch related tasks?
- How does `implement` interact with the existing `agent-skills:incremental-implementation` and `superpowers:test-driven-development` skills — replace them, wrap them, or coexist?
- Should `implement` enforce that every commit references at least one AC ID?

---
*This document follows the https://specscore.md/feature-specification*
