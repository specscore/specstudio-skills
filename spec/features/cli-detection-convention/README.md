# Feature: CLI Detection Convention

> [SpecScore.**Studio**](https://specscore.studio): | [Explore](https://specscore.studio/app/github.com/specscore/specstudio-skills/spec/features/cli-detection-convention?op=explore) | [Edit](https://specscore.studio/app/github.com/specscore/specstudio-skills/spec/features/cli-detection-convention?op=edit) | [Ask question](https://specscore.studio/app/github.com/specscore/specstudio-skills/spec/features/cli-detection-convention?op=ask) | [Request change](https://specscore.studio/app/github.com/specscore/specstudio-skills/spec/features/cli-detection-convention?op=request-change) |
**Status:** Approved
**Date:** 2026-06-03
**Owner:** alexander.trakhimenok
**Source Ideas:** cli-detection-convention
**Supersedes:** —
**Grade:** A

## Summary

One mechanism for skills to detect the `specscore` CLI — invoke it and branch on the exit status — with a per-skill-class response policy captured in a shared reference, plus conversion of three representative skills (`ideate`, `relocate-idea`, `consilium`) to follow it.

## Problem

The `command -v specscore` probe is repeated across five skills (`ideate`, `init`, `sidekick`, `relocate-idea`, `consilium`) but encodes three different intents: CLI-optional-with-fallback, CLI-mandatory-no-fallback, and detection-drives-a-wizard. There is no shared rule, so each author re-derives detection and the behaviors drift. The naive simplification — "just call it and fall back on any failure" — is also wrong: it cannot tell "binary absent" from "binary present but errored," so it would silently mask real CLI failures and risk double-writes after a partial mutation. The detection *mechanism* is actually uniform; only the *response* to each outcome legitimately varies. This Feature pins that down once.

## Behavior

### The Shared Convention

A single shared reference is the source of truth for how every skill detects the CLI. It states the one mechanism and a response table keyed by skill class.

#### REQ: convention-doc

The system MUST provide a shared reference at `skills/shared/cli-detection.md` that defines (a) the single detection mechanism and (b) a response table with one row per skill class: **optional**, **mandatory**, **capability-gated**, and **wizard**. The table MUST state, for each class, the action on each branch outcome.

#### REQ: single-mechanism

Every skill that depends on the `specscore` CLI MUST detect it by invoking the relevant `specscore` command and branching on that command's exit status. Skills MUST NOT use a standalone `command -v specscore` probe as the detection mechanism.

#### REQ: branch-outcomes

The detection branch MUST distinguish four outcomes: **success** (exit `0`), **`127`** (binary absent — provided by the shell, not by `specscore`), the **dedicated "too old / missing subcommand" exit code** (assumed implemented in the CLI per the `specscore-cli` seed), and **any other non-zero** (the command ran and genuinely failed).

### Per-Class Response Policy

Only the response to each outcome varies. Each converted skill MUST implement the response row for its class.

#### REQ: optional-response

For skills where the CLI is optional and a real fallback exists (`ideate`, `sidekick`): on **success** use the CLI; on **`127`** take the fallback; on **any other non-zero** surface the error and MUST NOT take the fallback.

#### REQ: mandatory-response

For skills where the CLI is mandatory with no fallback (`relocate-idea`): on **`127`** emit the standardized install message (pointing at `/specscore:install`) and stop; on **any other non-zero** surface the error.

#### REQ: capability-gated-response

For skills that depend on a specific subcommand (`consilium` needs `consilium verdict`): on the **dedicated "too old / missing subcommand" exit code** emit an upgrade message; on **`127`** emit the install message and stop; on **any other non-zero** surface the error. The capability check MUST run **before** expensive multi-step work (e.g. before `consilium`'s 9-role panel), not after.

### Skill Conversions

Three representative skills — one per class that changes behavior — adopt the convention in this Feature. `init` (wizard class) is documented in the convention but not modified here.

#### REQ: ideate-conversion

`skills/ideate/SKILL.md` Step 3a MUST replace its `command -v specscore` probe with the call-and-branch mechanism per the optional-response policy, and MUST cite `skills/shared/cli-detection.md`.

#### REQ: relocate-idea-conversion

`skills/relocate-idea/SKILL.md` MUST follow the mandatory-response policy, cite `skills/shared/cli-detection.md`, and use the standardized install message.

#### REQ: consilium-conversion

`skills/consilium/SKILL.md` MUST follow the capability-gated-response policy — branching on the dedicated exit code for a missing `consilium verdict` subcommand to an upgrade message, and running the capability check before the 9-role panel — and MUST cite `skills/shared/cli-detection.md`.

## Acceptance Criteria

### AC: doc-exists (verifies REQ:convention-doc)

**Given** the repository
**When** `skills/shared/cli-detection.md` is read
**Then** it describes the call-and-branch detection mechanism and contains a response table with rows for the optional, mandatory, capability-gated, and wizard classes

### AC: no-command-v (verifies REQ:single-mechanism)

**Given** a skill converted by this Feature
**When** its `SKILL.md` CLI-detection step is inspected
**Then** the step instructs invoking `specscore` and branching on the exit status, and uses no `command -v specscore` probe as the detection mechanism

### AC: four-outcomes (verifies REQ:branch-outcomes)

**Given** `skills/shared/cli-detection.md`
**When** the mechanism section is read
**Then** it enumerates exactly the four outcomes — success, `127`, the dedicated too-old/missing-subcommand code, and other non-zero — and the meaning of each

### AC: optional-127-fallback (verifies REQ:optional-response)

**Given** the convention is applied to `ideate`
**When** the first `specscore` call exits `127`
**Then** `ideate` proceeds via its direct-write fallback path

### AC: optional-nonzero-no-fallback (verifies REQ:optional-response)

**Given** the convention is applied to `ideate`
**When** the `specscore idea new` call exits non-zero for a reason other than `127`
**Then** `ideate` surfaces the error and does NOT take the fallback path

### AC: mandatory-absent-install (verifies REQ:mandatory-response)

**Given** the convention is applied to `relocate-idea`
**When** the `specscore` call exits `127`
**Then** `relocate-idea` emits the standardized install message pointing at `/specscore:install` and stops without writing

### AC: capability-upgrade (verifies REQ:capability-gated-response)

**Given** the convention is applied to `consilium`
**When** `specscore` is present but returns the dedicated "missing subcommand" exit code for `consilium verdict`
**Then** `consilium` emits an upgrade message rather than a generic failure

### AC: capability-precheck-order (verifies REQ:capability-gated-response)

**Given** the converted `consilium` skill
**When** its `SKILL.md` is inspected
**Then** the capability check is specified to run before the 9-role panel dispatch

### AC: ideate-cites (verifies REQ:ideate-conversion)

**Given** `skills/ideate/SKILL.md` after conversion
**When** Step 3a is read
**Then** the `command -v` probe is gone and the step cites `skills/shared/cli-detection.md`

### AC: relocate-cites (verifies REQ:relocate-idea-conversion)

**Given** `skills/relocate-idea/SKILL.md` after conversion
**When** the CLI-detection step is read
**Then** it follows the mandatory-response policy and cites `skills/shared/cli-detection.md`

### AC: consilium-cites (verifies REQ:consilium-conversion)

**Given** `skills/consilium/SKILL.md` after conversion
**When** the CLI-detection step is read
**Then** it follows the capability-gated-response policy and cites `skills/shared/cli-detection.md`

## Rehearse Integration

No executable Rehearse stubs are scaffolded. Every AC is a documentation-conformance or skill-instruction-behavior check verified by inspecting `skills/shared/cli-detection.md` and the three converted `SKILL.md` files (grep-checkable assertions), not an executable runtime surface in this repo. The runtime behaviors the ACs describe (e.g. fallback on `127`) live in the skill instructions, which are prose, not code under test here. Revisit if any of these skills gains an executable test harness.

## Open Questions

- The exact value of the dedicated "too old / missing subcommand" exit code is owned by the `specscore-cli` seed (`specscore-cli-should-return-a-dedicated-documented-exit`); this Feature assumes it exists and references it symbolically.
- When the remaining CLI-touching skills (`specify`, `plan`, `implement`, `sidekick`, `init`) are converted to cite the convention — deferred to follow-up, not blocking this Feature.
- Whether to later add a lint rule flagging `command -v specscore` for automatic drift detection (out of scope here; prose enforcement chosen).

---
*This document follows the https://specscore.md/feature-specification*
