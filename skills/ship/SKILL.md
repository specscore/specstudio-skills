---
name: ship
description: |
  Terminal skill of the SpecStudio pipeline. Enforces spec-aware release
  gates for a single Feature, dispatches the deploy to a project-configured
  delegate skill, and on the delegate's explicit success transitions the
  Feature Implementing -> Stable and emits ship.completed. Ship gates,
  records, and performs a single dispatch — it never executes or
  orchestrates a deploy itself (no deploy mechanics, sequencing, retry,
  rollback, canary, flag-flips, scheduling, or multi-feature coordination).
  Trigger: "ship", "/ship", "specstudio:ship".
aliases: [ship]
---

# Ship

Close the SpecStudio pipeline for a single Feature: enforce the spec-aware
release gates, hand the deploy off to a project-configured delegate skill, and
record the outcome. Ship **gates, records, and dispatches once** — it never
deploys, sequences, retries, rolls back, or orchestrates. The high-blast-radius
deploy execution lives in a tool the project already trusts.

Implements the [Ship Skill Feature](../../spec/features/skills/ship/README.md).

## When to Use

- A single Feature at `spec/features/<feature-slug>/README.md` is `**Status:** Implementing`, its ACs have all been verified green, its latest recap shows no contradictions, and the user wants to release it.
- The user wants to run the spec-aware release gates for one Feature and (when a delegate is configured) dispatch the deploy.

**Refuse and redirect when:**

- The invocation does not name exactly one Feature slug → print a usage error and exit non-zero. Ship operates on one Feature per invocation. (AC: `rejects-non-feature-input`)
- The Feature's `**Status:**` is not `Implementing` → print the current Status and recommend the appropriate prior step (`Implementing` is the only status from which `Stable` is reachable). Write nothing; exit non-zero. (AC: `refuses-non-implementing-status`)

## Pre-Flight

Pre-flight refusals exit immediately, write nothing, and dispatch no reviewer and no delegate.

1. **Resolve input.** The skill MUST accept exactly one positional argument — the Feature slug — and resolve it to `spec/features/<feature-slug>/README.md`. Zero arguments or more than one argument is a usage error: print the correct usage, write nothing, and exit non-zero. Ship coordinates exactly one Feature; it never operates on a set of Features in one run. (AC: `rejects-non-feature-input`)

2. **Status guard.** Read the resolved Feature's body-metadata `**Status:**` line. Refuse unless it is exactly `Implementing`. On refusal, print the current Status value and recommend the appropriate prior step (e.g. `specstudio:implement` to reach `Implementing`, or note that an already-`Stable` Feature has nothing to ship). The lifecycle transition ship performs is `Implementing → Stable`, which is reachable only from `Implementing`; any other status is refused here before any further work. (AC: `refuses-non-implementing-status`)

## Checklist

Create a task for each and complete in order:

1. **Pre-flight** (steps above): resolve the single-Feature input, then the status guard. Refusals exit immediately; do not proceed.

<!-- Subsequent checklist steps (pre-flight machine gates, reviewer gate, deploy
dispatch, lifecycle transition, event emission) are added by later tasks in the
ship plan. -->

## References

- [Feature: Ship Skill](../../spec/features/skills/ship/README.md) — the SpecScore Feature this skill implements.
- [Plan: Ship Skill](../../spec/plans/ship.md) — the six-task plan this skill realizes.
- [verify SKILL.md](../verify/SKILL.md), [recap SKILL.md](../recap/SKILL.md) — sibling pipeline skills whose pre-flight and report-resolution conventions ship mirrors.
