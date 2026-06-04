# Feature: Ship Skill

> [SpecScore.**Studio**](https://specscore.studio): | [Explore](https://specscore.studio/app/github.com/specscore/specstudio-skills/spec/features/skills/ship?op=explore) | [Edit](https://specscore.studio/app/github.com/specscore/specstudio-skills/spec/features/skills/ship?op=edit) | [Ask question](https://specscore.studio/app/github.com/specscore/specstudio-skills/spec/features/skills/ship?op=ask) | [Request change](https://specscore.studio/app/github.com/specscore/specstudio-skills/spec/features/skills/ship?op=request-change) |

**Status:** Approved
**Date:** 2026-06-03
**Owner:** alexander.trakhimenok
**Source Ideas:** ship-skill, right-size-recap-cost
**Supersedes:** —
**Grade:** B

## Summary

`specstudio:ship` is the SpecStudio pipeline's terminal skill. It enforces spec-aware release gates for a single Feature, dispatches the actual deploy to a project-configured delegate skill, and — only on the delegate's explicit success — transitions the Feature `Implementing → Stable` and emits `ship.completed`. It gates, records, and performs a single dispatch; it never executes or orchestrates a deploy itself.

## Problem

The handoff from "code merged" to "feature actually shipped" is where deploy-time mistakes happen. Generic shipping skills (e.g. `gh-deploy`, `agent-skills:shipping-and-launch`) cover the deploy *mechanics* but not the *spec-aware* checks: did every AC's `verify` report come back green? Did `recap` surface no contradictions? Has a human signed off? Without a spec-aware gate, a Feature can reach production with unverified ACs or unreconciled drift, and nothing transitions the Feature to `Stable` to close the pipeline honestly.

A spec-aware `ship` skill exists to enforce those gates before any deploy action is taken, and to record the lifecycle outcome — while deliberately delegating the high-blast-radius deploy execution to a tool the project already trusts.

## Behavior

### Invocation and Scope

#### REQ: single-feature-input

The skill MUST accept exactly one positional argument — a Feature slug resolving to `spec/features/<feature-slug>/README.md` — and MUST operate on that single Feature only. It MUST NOT accept or coordinate a set of Features in one invocation.

### Pre-Flight Machine Gates

These are hard gates the skill enforces itself by reading committed artifacts (mirroring how `specstudio:verify` and `specstudio:recap` resolve reports). They run before any reviewer or delegate is dispatched. Any failure refuses, writes nothing, and exits non-zero.

#### REQ: status-preflight

The skill MUST refuse unless the Feature's `**Status:**` is `Implementing`. `Implementing → Stable` is the only lifecycle transition that reaches `Stable`; a Feature in any other status cannot be shipped. On refusal the skill MUST print the current Status and recommend the appropriate prior step.

#### REQ: verify-green-gate

The skill MUST resolve the latest `spec/features/<feature-slug>/_verify/<sha>.md` report reachable at HEAD and MUST refuse unless every AC in that report's YAML summary carries a `pass` verdict (zero `fail`, zero `error`, zero `unmapped`). If no verify report exists, the skill MUST refuse and recommend `specstudio:verify` first.

#### REQ: recap-contradiction-gate

When the recap gate is enforced (the default), the skill MUST resolve the latest `spec/features/<feature-slug>/_recap/<sha>.md` report reachable at HEAD and MUST refuse unless that report's YAML summary contains zero `contradiction` verdicts; if no recap report exists, the skill MUST refuse and recommend `specstudio:recap` first. When `recap.required_for_ship` is `false` in `specscore.yaml`, the skill MUST skip the recap gate entirely — performing no presence check and no contradiction check — and proceed. The waiver MUST be explicit: it is read from configuration only and MUST NOT be inferred from project size, token budget, or any heuristic. The `verify-green-gate` is unaffected by this waiver and remains mandatory in all cases.

### Recap Gate Waiver

#### REQ: recap-waiver-config-and-logging

The recap gate's enforcement is governed by a top-level `recap:` config block in `specscore.yaml` exposing a single boolean field, `required_for_ship`, whose default is `true` (recap stays a mandatory upstream gate unless a project explicitly opts out). When the gate is waived (`required_for_ship: false`), the skill MUST NOT proceed silently: it MUST disclose the waiver in its pre-flight output AND MUST record `recap_status` (one of `enforced` | `waived`) on the `ship.completed` event payload so that every ship — waived or not — carries an auditable record of whether the drift gate ran. When the gate is enforced, `recap_status` is `enforced`.

### Reviewer Gate (Judgment and Human Go/No-Go)

#### REQ: ship-reviewer-gate

After the pre-flight machine gates pass, the skill MUST dispatch the reviewer gate registered under the ship gate-point event via the shared reviewer-gates loader and runner ([loader.md](../../../../skills/shared/reviewer-gates/loader.md), [runner.md](../../../../skills/shared/reviewer-gates/runner.md)). The gate releases only when every configured reviewer entry returns `Approved` under AND-composition. A `type: human` entry captures the operator's go/no-go decision. The skill MUST carry no hardcoded baseline reviewer and MUST NOT fall back to a built-in reviewer when the gate is unconfigured — it follows the loader's refuse-or-release outcome exactly. The reviewer gate uses only the existing `ai` and `human` reviewer types; ship introduces no new reviewer type.

### Deploy Dispatch (Delegation)

#### REQ: single-delegate-dispatch

When the reviewer gate releases AND a `ship.delegate` is configured in `specscore.yaml`, the skill MUST dispatch exactly one delegate skill, passing the delegate's configured `args`. The skill MUST invoke the delegate once. It MUST NOT sequence multiple delegates, retry a failed delegate, or interpose any orchestration logic between itself and the delegate.

#### REQ: no-delegate-handback

When no `ship.delegate` is configured, the skill MUST run the gates, then hand back to the user with a summary of gate results and an explicit statement that no deploy delegate is configured. In this path the skill MUST NOT attempt any deploy and MUST NOT transition the Feature's status. It never guesses how to deploy.

#### REQ: explicit-success-required

The skill MUST treat only an explicit success signal from the delegate as success. On delegate failure, an ambiguous result, or no clear success signal, the skill MUST NOT transition the Feature's status, MUST NOT retry, and MUST surface the delegate's outcome verbatim to the user.

### Lifecycle Transition and Event

#### REQ: transition-on-success

On the delegate's explicit success, the skill MUST transition the Feature `Implementing → Stable` via `specscore feature change-status <feature-slug> --to Stable`. The transition MUST NOT occur on any other outcome.

#### REQ: emit-ship-completed

On a successful ship (delegate success and status transition), the skill MUST emit the `ship.completed` event exactly once, after applying [publication-policy.md](../../../../skills/shared/publication-policy.md) for that checkpoint, with `publication_result` recorded on the event. The no-delegate-handback and any refusal paths MUST NOT emit `ship.completed`.

### Architectural Boundary

#### REQ: no-execution-or-orchestration

The skill MUST NOT itself perform deploy mechanics (build, push, migrate), nor sequencing, retry, rollback, canary, feature-flag flips, scheduling, or multi-feature/multi-project coordination. Those concerns belong to the delegate or to a separate orchestration layer. This requirement is the load-bearing boundary that keeps ship within single-project Studio scope; the `ship:` config block MUST NOT grow fields that express any of these concerns.

## Acceptance Criteria

### AC: rejects-non-feature-input (verifies REQ:single-feature-input)

**Given** the user invokes `specstudio:ship` with zero or more than one positional argument
**When** the skill performs input resolution
**Then** it refuses with a usage error, writes nothing, and exits non-zero.

### AC: refuses-non-implementing-status (verifies REQ:status-preflight)

**Given** a Feature whose `**Status:**` is not `Implementing` (e.g. `Approved` or `Stable`)
**When** `specstudio:ship <feature-slug>` runs pre-flight
**Then** it prints the current Status, refuses, dispatches no reviewer and no delegate, and exits non-zero.

### AC: refuses-when-verify-not-green (verifies REQ:verify-green-gate)

**Given** a Feature in `Implementing` whose latest `_verify/<sha>.md` report at HEAD contains at least one non-`pass` AC verdict (or no verify report exists)
**When** the skill runs pre-flight
**Then** it refuses, names the failing/missing verify condition, recommends `specstudio:verify`, and exits non-zero.

### AC: refuses-on-recap-contradiction (verifies REQ:recap-contradiction-gate)

**Given** the recap gate is enforced (the default; `recap.required_for_ship` is absent or `true`) and a Feature in `Implementing` whose latest `_recap/<sha>.md` report at HEAD contains at least one `contradiction` verdict (or no recap report exists)
**When** the skill runs pre-flight
**Then** it refuses, names the contradiction/missing-recap condition, recommends `specstudio:recap`, and exits non-zero.

### AC: proceeds-when-recap-waived (verifies REQ:recap-contradiction-gate)

**Given** `recap.required_for_ship` is `false` in `specscore.yaml` and a Feature in `Implementing` whose verify report at HEAD is all-green, regardless of whether a recap report exists or what verdicts it carries
**When** the skill runs pre-flight
**Then** it skips the recap gate entirely (no presence check, no contradiction check), keeps the verify-green gate enforced, and proceeds to the reviewer gate.

### AC: records-recap-waiver (verifies REQ:recap-waiver-config-and-logging)

**Given** a ship run in which `recap.required_for_ship` is `false`
**When** the skill completes pre-flight and (on success) emits `ship.completed`
**Then** it discloses the waiver in its pre-flight output and the emitted `ship.completed` payload carries `recap_status: waived`; and when the gate is enforced instead, the payload carries `recap_status: enforced`.

### AC: gate-releases-only-on-all-approved (verifies REQ:ship-reviewer-gate)

**Given** pre-flight has passed and the ship gate-point event has configured reviewers including a `type: human` entry
**When** the reviewer gate runs and every entry returns `Approved`
**Then** the skill proceeds to deploy dispatch; and if any entry returns `Issues Found`, the skill halts at the gate without dispatching a delegate.

### AC: dispatches-single-configured-delegate (verifies REQ:single-delegate-dispatch)

**Given** the reviewer gate has released and `specscore.yaml` configures one `ship.delegate` with a skill name and args
**When** the skill performs deploy dispatch
**Then** it invokes that single delegate exactly once with the configured args and performs no sequencing or retry.

### AC: hands-back-when-no-delegate (verifies REQ:no-delegate-handback)

**Given** the reviewer gate has released and no `ship.delegate` is configured
**When** the skill reaches deploy dispatch
**Then** it summarizes gate results, states that no delegate is configured, transitions no status, emits no `ship.completed`, and exits without attempting a deploy.

### AC: no-transition-on-delegate-failure (verifies REQ:explicit-success-required)

**Given** a configured delegate is dispatched and returns failure or an ambiguous result
**When** the skill evaluates the delegate outcome
**Then** it leaves the Feature in `Implementing`, does not retry, surfaces the delegate's outcome, and emits no `ship.completed`.

### AC: transitions-to-stable-on-success (verifies REQ:transition-on-success)

**Given** a configured delegate returns explicit success for a Feature in `Implementing`
**When** the skill processes the success
**Then** it runs `specscore feature change-status <feature-slug> --to Stable` and the Feature's `**Status:**` becomes `Stable`.

### AC: emits-ship-completed-once (verifies REQ:emit-ship-completed)

**Given** a successful ship (delegate success and status transition to `Stable`)
**When** the skill finalizes
**Then** it applies publication policy and emits exactly one `ship.completed` event carrying `publication_result`; no `ship.completed` is emitted on refusal or hand-back.

### AC: bars-execution-and-orchestration (verifies REQ:no-execution-or-orchestration)

**Given** the skill and the `ship:` config block
**When** a ship run is performed end-to-end
**Then** the skill performs no deploy mechanics, sequencing, retry, rollback, canary, flag-flips, scheduling, or multi-feature coordination, and the `ship:` config exposes no fields expressing those concerns.

## Dependencies

This Feature composes existing mechanisms and requires small, precedented additions to sibling Features:

- **[reviewer-gates](../../reviewer-gates/README.md):** register a ship gate-point event (analogous to `implementation.pre_commit` / `implementation.pre_push`) so the reviewer gate can be keyed under `gates:`. No new reviewer `type` is required.
- **[events catalog](../../../../skills/shared/events.md):** register the `ship.completed` event (and the ship gate-point event identifier).
- **[flexible-lifecycle-flows](../../flexible-lifecycle-flows/README.md):** add the terminal `recap → ship` node to the valid lifecycle graph.
- **[change-publication-policy](../../change-publication-policy/README.md):** the `ship.completed` checkpoint resolves through the existing publication-policy ladder.
- **[third-party-integration](../../third-party-integration/README.md):** the delegate-dispatch handoff follows the third-party skill integration contract.

## Rehearse Integration

Rehearse scenarios are deferred for this Feature pending the dependency resolutions above (the ship gate-point event and `ship.completed` registration). Each AC is phrased as an observable Given/When/Then so that `_tests/<req-slug>-<ac-slug>.md` scenarios can be scaffolded once those identifiers are registered. No `_tests/` stubs are scaffolded at spec time.

## Not Doing / Out of Scope

- **Executing deploy mechanics** (build, push, migrate) — delegated to a configured deploy skill; ship never deploys.
- **Sequencing, retry, rollback, canary, or feature-flag flips** — operational execution owned by the delegate, not ship.
- **Multi-feature / multi-project release coordination** — cross-thing orchestration is out of single-project Studio scope.
- **Scheduling or dispatching work across runs** — belongs to a separate orchestration layer, not ship.
- **An ordered delegate pipeline** — MVP dispatches exactly one delegate; ordered multi-step deploy choreography lives inside the delegate.
- **A new reviewer `type`** — pre-flight handles machine checks; the reviewer gate uses the existing `ai`/`human` types only.
- **Incremental / cheaper recap** — reducing recap's per-AC token cost (only re-narrating changed ACs) is the deferred fast-follow from the `right-size-recap-cost` Idea; this Feature only makes the gate waivable, not cheaper.
- **Per-AC granular waiver** — `recap.required_for_ship` waives the gate for the whole Feature; selectively waiving individual ACs is out of scope.
- **An auto-skip heuristic** — the waiver is an explicit config opt-in only; ship never infers it from project size or token budget.

## Open Questions

- What is the canonical identifier for the ship gate-point event (e.g. `ship.pre_dispatch`), and is its registration owned by `reviewer-gates`?
- Should ship own the `Implementing → Stable` transition directly, or delegate the status write to lifecycle tooling?
- Where do the `ship:` and the new `recap:` config schemas durably live — defined by this Feature, or owned by a dedicated config Feature alongside `reviewer-gates` and `change-publication-policy`? (The `recap.required_for_ship` field is documented here for now; durable schema ownership tracks this question.)
- When the `review` stage ships, should a waived recap (`recap.required_for_ship: false`) also relax `review`'s own recap expectations, or does `review` keep an independent recap policy?

## Sidekick Seeds Generated

- [run-recap-s-per-ac-drift-narrators-via-dynamic-parallel](../../../ideas/seeds/run-recap-s-per-ac-drift-narrators-via-dynamic-parallel.md) — captured 2026-06-04 by specstudio:specify

---
*This document follows the https://specscore.md/feature-specification*
