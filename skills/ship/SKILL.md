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
- The Feature's latest verify report is missing or not all-green → recommend `specstudio:verify <feature-slug>`. Write nothing; exit non-zero. (AC: `refuses-when-verify-not-green`)
- The Feature's latest recap report is missing or contains any contradiction → recommend `specstudio:recap <feature-slug>`. Write nothing; exit non-zero. (AC: `refuses-on-recap-contradiction`)

## Pre-Flight

Pre-flight refusals exit immediately, write nothing, and dispatch no reviewer and no delegate.

1. **Resolve input.** The skill MUST accept exactly one positional argument — the Feature slug — and resolve it to `spec/features/<feature-slug>/README.md`. Zero arguments or more than one argument is a usage error: print the correct usage, write nothing, and exit non-zero. Ship coordinates exactly one Feature; it never operates on a set of Features in one run. (AC: `rejects-non-feature-input`)

2. **Status guard.** Read the resolved Feature's body-metadata `**Status:**` line. Refuse unless it is exactly `Implementing`. On refusal, print the current Status value and recommend the appropriate prior step (e.g. `specstudio:implement` to reach `Implementing`, or note that an already-`Stable` Feature has nothing to ship). The lifecycle transition ship performs is `Implementing → Stable`, which is reachable only from `Implementing`; any other status is refused here before any further work. (AC: `refuses-non-implementing-status`)

### Machine gates

These are hard gates ship enforces itself by reading committed artifacts — it does not delegate them to a reviewer. They mirror how `specstudio:verify` and `specstudio:recap` resolve their reports. Both run after the status guard and before any reviewer or delegate is dispatched.

3. **Verify-green gate.** Resolve the latest `spec/features/<feature-slug>/_verify/<sha>.md` report reachable at HEAD — prefer the report whose `<sha>` matches `git rev-parse --short HEAD`, otherwise the report whose embedded YAML `revision:` is most recent in branch history (the same resolution `specstudio:recap` uses). Parse its top-of-file YAML summary block. Refuse unless **every** AC verdict is `pass` (zero `fail`, zero `error`, zero `unmapped`). If `_verify/` is absent or contains no report reachable at HEAD, refuse. On any refusal, name the failing or missing condition, recommend `specstudio:verify <feature-slug>`, write nothing, and exit non-zero. (AC: `refuses-when-verify-not-green`)

4. **Recap-no-contradiction gate.** Resolve the latest `spec/features/<feature-slug>/_recap/<sha>.md` report reachable at HEAD using the same resolution rule as step 3. Parse its YAML summary's `drift:` list. Refuse unless it contains **zero** `contradiction` verdicts. If `_recap/` is absent or contains no report reachable at HEAD, refuse. On any refusal, name the contradiction or missing-recap condition, recommend `specstudio:recap <feature-slug>`, write nothing, and exit non-zero. Recap is a **strictly mandatory** upstream gate for ship — there is no waiver path in the MVP. (AC: `refuses-on-recap-contradiction`)

## Checklist

Create a task for each and complete in order:

1. **Pre-flight input + status** (Pre-Flight steps 1–2): resolve the single-Feature input, then the status guard. Refusals exit immediately; do not proceed.
2. **Pre-flight machine gates** (Pre-Flight steps 3–4): verify-green, then recap-no-contradiction. Both are hard gates; any refusal exits immediately.
3. **Reviewer gate** (`## Reviewer Gate`): fire `ship.pre_dispatch`, load and run `gates.ship.pre_dispatch` reviewers (AND-composed). Halt on `Issues Found`; proceed only on release.

<!-- Subsequent checklist steps (deploy dispatch, lifecycle transition, event
emission) are added by later tasks in the ship plan. -->

## Reviewer Gate

After every pre-flight gate passes, ship fires the `ship.pre_dispatch` gate-point event and evaluates the reviewer gate keyed on it — `gates.ship.pre_dispatch` in `specscore.yaml` — via the shared reviewer-gates [loader](../shared/reviewer-gates/loader.md) and [runner](../shared/reviewer-gates/runner.md). This is the judgment-and-human go/no-go checkpoint before any deploy. Ship carries **no** hardcoded baseline reviewer; the reviewer list comes exclusively from the gate config.

1. **Fire and load.** Fire `ship.pre_dispatch` (single-fire per run; see [events.md](../shared/events.md)). Load and validate `gates.ship.pre_dispatch.reviewers` via the loader with event key `ship.pre_dispatch`. If the gate is unconfigured or invalid, the loader refuses per its contract — surface its error verbatim and halt; do not dispatch a delegate, do not transition status.
2. **Run.** Dispatch the validated reviewer list via the runner: serial, in declared list order, AND-composed. The gate releases only when **every** entry returns `Approved`. A `type: human` entry is the operator's go/no-go. Ship introduces **no new reviewer type** — it uses only the types reviewer-gates already defines (`ai`, `human`, and the broader `deterministic` / `auto-approve`).
3. **Outcome.** On release (`Approved`), proceed to deploy dispatch. On `Issues Found`, halt: surface the failing reviewer's `Blocker` findings, dispatch no delegate, and transition no status. (AC: `gate-releases-only-on-all-approved`)

## References

- [Feature: Ship Skill](../../spec/features/skills/ship/README.md) — the SpecScore Feature this skill implements.
- [Plan: Ship Skill](../../spec/plans/ship.md) — the six-task plan this skill realizes.
- [verify SKILL.md](../verify/SKILL.md), [recap SKILL.md](../recap/SKILL.md) — sibling pipeline skills whose pre-flight and report-resolution conventions ship mirrors.
