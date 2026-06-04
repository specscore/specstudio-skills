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
- The Feature's latest recap report is missing or contains any contradiction **and the recap gate is enforced** (the default — `recap.required_for_ship` is absent or `true`) → recommend `specstudio:recap <feature-slug>`. Write nothing; exit non-zero. (AC: `refuses-on-recap-contradiction`) When `recap.required_for_ship` is `false`, this refusal does not apply — the recap gate is skipped (AC: `proceeds-when-recap-waived`).

## Pre-Flight

Pre-flight refusals exit immediately, write nothing, and dispatch no reviewer and no delegate.

1. **Resolve input.** The skill MUST accept exactly one positional argument — the Feature slug — and resolve it to `spec/features/<feature-slug>/README.md`. Zero arguments or more than one argument is a usage error: print the correct usage, write nothing, and exit non-zero. Ship coordinates exactly one Feature; it never operates on a set of Features in one run. (AC: `rejects-non-feature-input`)

2. **Status guard.** Read the resolved Feature's body-metadata `**Status:**` line. Refuse unless it is exactly `Implementing`. On refusal, print the current Status value and recommend the appropriate prior step (e.g. `specstudio:implement` to reach `Implementing`, or note that an already-`Stable` Feature has nothing to ship). The lifecycle transition ship performs is `Implementing → Stable`, which is reachable only from `Implementing`; any other status is refused here before any further work. (AC: `refuses-non-implementing-status`)

### Machine gates

These are hard gates ship enforces itself by reading committed artifacts — it does not delegate them to a reviewer. They mirror how `specstudio:verify` and `specstudio:recap` resolve their reports. Both run after the status guard and before any reviewer or delegate is dispatched.

3. **Verify-green gate.** Resolve the latest `spec/features/<feature-slug>/_verify/<sha>.md` report reachable at HEAD — prefer the report whose `<sha>` matches `git rev-parse --short HEAD`, otherwise the report whose embedded YAML `revision:` is most recent in branch history (the same resolution `specstudio:recap` uses). Parse its top-of-file YAML summary block. Refuse unless **every** AC verdict is `pass` (zero `fail`, zero `error`, zero `unmapped`). If `_verify/` is absent or contains no report reachable at HEAD, refuse. On any refusal, name the failing or missing condition, recommend `specstudio:verify <feature-slug>`, write nothing, and exit non-zero. (AC: `refuses-when-verify-not-green`)

4. **Recap-no-contradiction gate (config-gated).** First read `recap.required_for_ship` from `specscore.yaml` (default `true` when the `recap:` block or the key is absent). The waiver is read from config **only** — never inferred from project size, token budget, or any heuristic.
   - **Enforced (`true`, the default):** Resolve the latest `spec/features/<feature-slug>/_recap/<sha>.md` report reachable at HEAD using the same resolution rule as step 3. Parse its YAML summary's `drift:` list. Refuse unless it contains **zero** `contradiction` verdicts. If `_recap/` is absent or contains no report reachable at HEAD, refuse. On any refusal, name the contradiction or missing-recap condition, recommend `specstudio:recap <feature-slug>`, write nothing, and exit non-zero. (AC: `refuses-on-recap-contradiction`)
   - **Waived (`false`):** Skip this gate entirely — perform **no** recap-presence check and **no** contradiction check — and proceed to the reviewer gate. The waiver MUST be disclosed in pre-flight output (see step 5 of the checklist / `## Recap-Gate Waiver` below); it is never silent. (AC: `proceeds-when-recap-waived`)

   The waiver applies **only** to the recap gate. The verify-green gate (step 3) is unaffected and remains mandatory in all cases.

## Checklist

Create a task for each and complete in order:

1. **Pre-flight input + status** (Pre-Flight steps 1–2): resolve the single-Feature input, then the status guard. Refusals exit immediately; do not proceed.
2. **Pre-flight machine gates** (Pre-Flight steps 3–4): verify-green, then recap-no-contradiction. Both are hard gates; any refusal exits immediately.
3. **Reviewer gate** (`## Reviewer Gate`): fire `ship.pre_dispatch`, load and run `gates.ship.pre_dispatch` reviewers (AND-composed). Halt on `Issues Found`; proceed only on release.
4. **Deploy dispatch** (`## Deploy Dispatch`): if `ship.delegate` is configured, dispatch it once; otherwise hand back. Act on the delegate's outcome only.
5. **Transition and emit** (`## Lifecycle Transition and Event`): on explicit delegate success, transition `Implementing → Stable`, apply publication policy, and emit `ship.completed` exactly once.

## Reviewer Gate

After every pre-flight gate passes, ship fires the `ship.pre_dispatch` gate-point event and evaluates the reviewer gate keyed on it — `gates.ship.pre_dispatch` in `specscore.yaml` — via the shared reviewer-gates [loader](../shared/reviewer-gates/loader.md) and [runner](../shared/reviewer-gates/runner.md). This is the judgment-and-human go/no-go checkpoint before any deploy. Ship carries **no** hardcoded baseline reviewer; the reviewer list comes exclusively from the gate config.

1. **Fire and load.** Fire `ship.pre_dispatch` (single-fire per run; see [events.md](../shared/events.md)). Load and validate `gates.ship.pre_dispatch.reviewers` via the loader with event key `ship.pre_dispatch`. If the gate is unconfigured or invalid, the loader refuses per its contract — surface its error verbatim and halt; do not dispatch a delegate, do not transition status.
2. **Run.** Dispatch the validated reviewer list via the runner: serial, in declared list order, AND-composed. The gate releases only when **every** entry returns `Approved`. A `type: human` entry is the operator's go/no-go. Ship introduces **no new reviewer type** — it uses only the types reviewer-gates already defines (`ai`, `human`, and the broader `deterministic` / `auto-approve`).
3. **Outcome.** On release (`Approved`), proceed to deploy dispatch. On `Issues Found`, halt: surface the failing reviewer's `Blocker` findings, dispatch no delegate, and transition no status. (AC: `gate-releases-only-on-all-approved`)

## Deploy Dispatch

When the reviewer gate releases, ship hands the deploy to a project-configured delegate. Ship dispatches **once** and reacts to a single outcome — it never sequences, retries, or orchestrates.

### Config schema (`ship:` in `specscore.yaml`)

```yaml
ship:
  delegate:
    skill: <skill-name>     # the deploy skill ship invokes (e.g. gh-deploy)
    args: <string>          # opaque args handed to the delegate verbatim
```

`delegate.skill` and `delegate.args` are the **only** recognized fields. Ship MUST NOT honor — and the `ship:` schema MUST NOT grow — any field expressing sequencing, retry, rollback, canary, feature-flag, or scheduling behavior; those belong to the delegate, never to ship (the boundary the `bars-execution-and-orchestration` AC enforces).

### Dispatch behavior

1. **Configured delegate → single dispatch.** When `ship.delegate` is present, invoke the named delegate skill **exactly once**, passing `delegate.args`. Ship does not sequence multiple delegates and does not retry. (AC: `dispatches-single-configured-delegate`)
2. **No delegate → hand back.** When `ship.delegate` is absent, summarize the gate results, state explicitly that no deploy delegate is configured, transition no status, emit no `ship.completed`, and exit without attempting a deploy. Ship never guesses how to deploy. (AC: `hands-back-when-no-delegate`)
3. **Explicit success only.** Treat **only** the delegate's explicit success signal as success. On delegate failure, an ambiguous result, or no clear success signal, leave the Feature in `Implementing`, do not retry, surface the delegate's outcome verbatim, and emit no `ship.completed`. (AC: `no-transition-on-delegate-failure`)

## Lifecycle Transition and Event

Runs **only** on the delegate's explicit success (never on refusal or hand-back).

1. **Transition.** Transition the Feature `Implementing → Stable` by invoking `specscore feature change-status <feature-slug> --to Stable`. Ship owns this status write; `Implementing → Stable` is the only transition ship performs. (AC: `transitions-to-stable-on-success`)
2. **Publication + event.** Build the checkpoint manifest, apply [publication-policy.md](../shared/publication-policy.md) for the `ship.completed` checkpoint, then emit `ship.completed` **exactly once** with `publication_result` per [events.md](../shared/events.md). The event carries `feature_slug`, `revision`, the dispatched `delegate`, `from_status: Implementing` / `to_status: Stable`, and `recap_status` (`enforced` | `waived`) recording whether the recap gate ran this run. No `ship.completed` is emitted on any refusal or on the no-delegate hand-back. (AC: `emits-ship-completed-once`, `records-recap-waiver`)

## Recap-Gate Waiver

Recap is a mandatory upstream gate for ship **by default**. A project that accepts shipping without a fresh drift verdict (e.g. a solo or low-stakes project) can waive it explicitly via a top-level `recap:` block in `specscore.yaml`:

```yaml
recap:
  required_for_ship: true   # default; ship refuses without a contradiction-free recap report
  # required_for_ship: false → ship skips the recap gate entirely (see Pre-Flight step 4)
```

- `required_for_ship` is a boolean; its **default is `true`** (the key and the `recap:` block may both be absent — the gate is enforced).
- The waiver is **explicit**: read from config only, never inferred.
- The waiver covers **only** the recap gate. The verify-green gate is never waivable here.
- A waived run is **never silent**: the skill MUST state in its pre-flight output that the recap gate was waived (e.g. *"Recap gate waived (`recap.required_for_ship: false`); shipping on verify-green only."*), and MUST record `recap_status: waived` on the `ship.completed` event. When enforced, `recap_status` is `enforced`. (AC: `records-recap-waiver`)

This Feature only makes the gate **waivable**. Reducing recap's per-AC token cost (incremental / cheaper recap) is a separate, deferred effort.

## Architectural Boundary

This boundary is load-bearing and applies across the whole skill: **ship gates, records, and dispatches once — it never executes or orchestrates.** Concretely, ship MUST NOT itself perform any of:

- deploy mechanics (build, push, migrate);
- sequencing of multiple steps, or retrying a failed step;
- rollback, canary rollout, or feature-flag flips;
- scheduling, or dispatching work across runs;
- multi-feature or multi-project release coordination.

All of these belong to the delegate or to a separate orchestration layer. The `ship:` config block MUST expose **only** `delegate.skill` and `delegate.args`; it MUST NOT grow any field expressing the concerns above. Working inside one project is ship's scope; executing and dispatching work across things is not. (AC: `bars-execution-and-orchestration`)

## Red Flags

- Performing a deploy, migration, or push directly instead of delegating it.
- Sequencing or retrying delegates, or interposing orchestration between ship and the delegate.
- Owning rollback, canary, or feature-flag logic in ship.
- Adding any `ship:` config field beyond `delegate.skill` / `delegate.args`.
- Transitioning status or emitting `ship.completed` on anything other than the delegate's explicit success.
- Coordinating more than one Feature in a single ship run.
- Carrying a hardcoded reviewer instead of loading `gates.ship.pre_dispatch`.
- Waiving the verify-green gate (never waivable), or waiving the recap gate by any path other than an explicit `recap.required_for_ship: false` config — or waiving recap silently (a waived run MUST disclose it and record `recap_status: waived`).

## References

- [Feature: Ship Skill](../../spec/features/skills/ship/README.md) — the SpecScore Feature this skill implements.
- [Plan: Ship Skill](../../spec/plans/ship.md) — the six-task plan this skill realizes.
- [verify SKILL.md](../verify/SKILL.md), [recap SKILL.md](../recap/SKILL.md) — sibling pipeline skills whose pre-flight and report-resolution conventions ship mirrors.
