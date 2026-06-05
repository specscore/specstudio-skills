# Feature: CLI Detection and Artifact-Creation Convention

> [SpecScore.**Studio**](https://specscore.studio): | [Explore](https://specscore.studio/app/github.com/specscore/specstudio-skills/spec/features/cli-detection-convention?op=explore) | [Edit](https://specscore.studio/app/github.com/specscore/specstudio-skills/spec/features/cli-detection-convention?op=edit) | [Ask question](https://specscore.studio/app/github.com/specscore/specstudio-skills/spec/features/cli-detection-convention?op=ask) | [Request change](https://specscore.studio/app/github.com/specscore/specstudio-skills/spec/features/cli-detection-convention?op=request-change) |
**Status:** Approved
**Date:** 2026-06-03
**Owner:** alexander.trakhimenok
**Source Ideas:** cli-detection-convention, require-cli-for-new-artifacts
**Supersedes:** —
**Grade:** A

## Summary

One mechanism for skills to detect the `specscore` CLI — invoke it and branch on the exit status — with a per-skill-class response policy captured in a shared reference. For the skills that **create new artifacts** (`ideate`, `sidekick`, `specify`, `plan`, `init`), creation is CLI-required: the skill calls the relevant `specscore … new` scaffold and never hand-writes an artifact from an embedded template; on a missing CLI it offers install-then-retry rather than a divergent fallback.

## Problem

The `command -v specscore` probe was repeated across five skills but encoded three different intents (optional-with-fallback, mandatory-no-fallback, wizard), so each author re-derived detection and the behaviors drifted. The naive "just call it and fall back on any failure" is wrong: it cannot tell "binary absent" from "binary present but errored," masking real failures and risking double-writes. The detection *mechanism* is uniform; only the *response* legitimately varies.

A second, related drift exists on the **creation** path. Producer skills each shipped a full authoritative artifact-schema block *and* a direct-write fallback that re-implemented what `specscore … new` already does. The schema was duplicated between every skill and the CLI, so the two could silently diverge, and the fallback's only effect for a CLI-absent user was to produce a divergent artifact that lint might later reject. This Feature retires the optional-with-fallback creation path: artifact creation goes through the CLI, the CLI scaffold is the single source of truth for artifact structure, and the producer skills carry no embedded schema.

## Behavior

### The Shared Convention

A single shared reference is the source of truth for how every skill detects the CLI. It states the one mechanism and a response table keyed by skill class.

#### REQ: convention-doc

The system MUST provide a shared reference at `skills/shared/cli-detection.md` that defines (a) the single detection mechanism and (b) a response table whose rows cover each skill behavior: **creation** (producers that scaffold new artifacts), **mandatory** (hard-require, no fallback), **capability-gated** (depends on a specific subcommand), and **wizard** (interactive bootstrap). The table MUST state, for each row, the action on each branch outcome.

#### REQ: single-mechanism

Every skill that depends on the `specscore` CLI MUST detect it by invoking the relevant `specscore` command and branching on that command's exit status. Skills MUST NOT use a standalone `command -v specscore` probe as the detection mechanism.

#### REQ: branch-outcomes

The detection branch MUST distinguish four outcomes: **success** (exit `0`), **`127`** (binary absent — provided by the shell, not by `specscore`), **exit `8`** (`UnsupportedCommand` — binary present but too old / missing the required subcommand), and **any other non-zero** (the command ran and genuinely failed).

### Required-CLI Artifact Creation

The producer skills (`ideate`, `sidekick`, `specify`, `plan`, `init`) create new artifacts. This topic governs that creation path and supersedes the former optional-with-fallback behavior for it.

#### REQ: creation-via-cli

Every producer skill MUST create a new artifact exclusively by invoking the relevant `specscore … new` scaffold command (e.g. `specscore idea new`, `specscore feature new`). A producer MUST NOT hand-write a new artifact from an embedded template.

#### REQ: no-embedded-schema

Producer skills MUST NOT carry an embedded authoritative artifact schema/template block. The CLI scaffold output is the single source of truth for artifact structure. The skill describes only how to FILL the scaffold's sections via editing, not the artifact's file format.

#### REQ: creation-no-fallback

On exit `127` (CLI absent) for a creation call, a producer skill MUST NOT fall back to a direct write. It MUST emit the standardized install message (pointing at `/specscore:install`) and offer install-then-retry: once the CLI is installed, re-run the same scaffold call. On exit `8` (too old / missing the scaffold subcommand) it MUST emit an upgrade message and offer upgrade-then-retry. On any other non-zero it MUST surface the error and MUST NOT fall back.

#### REQ: cli-blackbox

From a producer skill's perspective the CLI is a black box: the skill invokes the scaffold command and MUST NOT depend on how the CLI produces the artifact (for example, where it sources templates). Any template-sourcing behavior is internal to the CLI and out of scope for the skills.

#### REQ: spec-url-reference

In place of the deleted embedded schema, a producer skill MAY carry a one-line read-only pointer to the artifact's specification page (`https://specscore.md/<artifact-type>-specification`). The skill MUST NOT fetch any URL and write the artifact from it; creation always goes through `specscore … new`.

### Per-Class Response Policy (Non-Creation)

For skills whose CLI use is not artifact creation, only the response to each outcome varies. Each converted skill MUST implement the response row for its class.

#### REQ: mandatory-response

For skills where the CLI is mandatory with no fallback (`relocate-idea`): on **`127`** emit the standardized install message (pointing at `/specscore:install`) and stop; on **any other non-zero** surface the error.

#### REQ: capability-gated-response

For skills that depend on a specific subcommand (`consilium` needs `consilium verdict`): on **exit `8`** (`UnsupportedCommand`) emit an upgrade message; on **`127`** emit the install message and stop; on **any other non-zero** surface the error. The capability check MUST run **before** expensive multi-step work (e.g. before `consilium`'s 9-role panel), not after.

### Skill Conversions

The detection mechanism is adopted by the representative skills below; the creation mandate is adopted by every producer skill.

#### REQ: producer-creation-conversion

Each producer skill — `skills/ideate/SKILL.md`, `skills/sidekick/SKILL.md`, `skills/specify/SKILL.md`, `skills/plan/SKILL.md`, and `skills/init/SKILL.md` — MUST follow the Required-CLI Artifact Creation policy (creation via `specscore … new`, no embedded schema, install/upgrade-then-retry, CLI as black box) and MUST cite `skills/shared/cli-detection.md`.

#### REQ: relocate-idea-conversion

`skills/relocate-idea/SKILL.md` MUST follow the mandatory-response policy, cite `skills/shared/cli-detection.md`, and use the standardized install message.

#### REQ: consilium-conversion

`skills/consilium/SKILL.md` MUST follow the capability-gated-response policy — branching on **exit `8`** (`UnsupportedCommand`) for a missing `consilium verdict` subcommand to an upgrade message, and running the capability check before the 9-role panel — and MUST cite `skills/shared/cli-detection.md`.

## Acceptance Criteria

### AC: doc-exists (verifies REQ:convention-doc)

**Given** the repository
**When** `skills/shared/cli-detection.md` is read
**Then** it describes the call-and-branch detection mechanism and contains a response table with rows for the creation, mandatory, capability-gated, and wizard behaviors

### AC: no-command-v (verifies REQ:single-mechanism)

**Given** a skill converted by this Feature
**When** its `SKILL.md` CLI-detection step is inspected
**Then** the step instructs invoking `specscore` and branching on the exit status, and uses no `command -v specscore` probe as the detection mechanism

### AC: four-outcomes (verifies REQ:branch-outcomes)

**Given** `skills/shared/cli-detection.md`
**When** the mechanism section is read
**Then** it enumerates exactly the four outcomes — success, `127`, exit `8` (too old / missing subcommand), and other non-zero — and the meaning of each

### AC: creation-uses-scaffold (verifies REQ:creation-via-cli)

**Given** a converted producer skill
**When** its artifact-creation step is read
**Then** it creates the new artifact by invoking a `specscore … new` scaffold command and contains no instruction to hand-write the artifact from a template

### AC: no-schema-block (verifies REQ:no-embedded-schema)

**Given** a converted producer skill's `SKILL.md`
**When** it is inspected for an authoritative artifact schema/template block
**Then** no such block is present, and the skill instead describes how to fill the scaffold's sections

### AC: creation-127-install-retry (verifies REQ:creation-no-fallback)

**Given** the convention is applied to a producer skill
**When** its creation `specscore … new` call exits `127`
**Then** the skill emits the install message pointing at `/specscore:install` and offers install-then-retry, and does NOT write the artifact via a direct-write fallback

### AC: creation-nonzero-no-fallback (verifies REQ:creation-no-fallback)

**Given** the convention is applied to a producer skill
**When** its creation call exits non-zero for a reason other than `127` or `8`
**Then** the skill surfaces the error and does NOT take a direct-write fallback

### AC: cli-blackbox-stated (verifies REQ:cli-blackbox)

**Given** a converted producer skill
**When** its creation step is read
**Then** it does not depend on how the CLI produces the artifact (no template-sourcing instructions), treating the scaffold command as a black box

### AC: spec-url-not-fetched (verifies REQ:spec-url-reference)

**Given** a producer skill that includes a specification-page pointer
**When** its creation step is read
**Then** the pointer is a read-only reference and the skill never fetches a URL to write the artifact

### AC: mandatory-absent-install (verifies REQ:mandatory-response)

**Given** the convention is applied to `relocate-idea`
**When** the `specscore` call exits `127`
**Then** `relocate-idea` emits the standardized install message pointing at `/specscore:install` and stops without writing

### AC: capability-upgrade (verifies REQ:capability-gated-response)

**Given** the convention is applied to `consilium`
**When** `specscore` is present but returns exit `8` (`UnsupportedCommand`) for `consilium verdict`
**Then** `consilium` emits an upgrade message rather than a generic failure

### AC: capability-precheck-order (verifies REQ:capability-gated-response)

**Given** the converted `consilium` skill
**When** its `SKILL.md` is inspected
**Then** the capability check is specified to run before the 9-role panel dispatch

### AC: producers-cite (verifies REQ:producer-creation-conversion)

**Given** `skills/ideate/SKILL.md`, `skills/sidekick/SKILL.md`, `skills/specify/SKILL.md`, `skills/plan/SKILL.md`, and `skills/init/SKILL.md` after conversion
**When** each one's creation step is read
**Then** it follows the Required-CLI Artifact Creation policy and cites `skills/shared/cli-detection.md`

### AC: relocate-cites (verifies REQ:relocate-idea-conversion)

**Given** `skills/relocate-idea/SKILL.md` after conversion
**When** the CLI-detection step is read
**Then** it follows the mandatory-response policy and cites `skills/shared/cli-detection.md`

### AC: consilium-cites (verifies REQ:consilium-conversion)

**Given** `skills/consilium/SKILL.md` after conversion
**When** the CLI-detection step is read
**Then** it follows the capability-gated-response policy and cites `skills/shared/cli-detection.md`

## Rehearse Integration

No executable Rehearse stubs are scaffolded. Every AC is a documentation-conformance or skill-instruction-behavior check verified by inspecting `skills/shared/cli-detection.md` and the converted `SKILL.md` files (grep-checkable assertions), not an executable runtime surface in this repo. The runtime behaviors the ACs describe (e.g. install-then-retry on `127`) live in the skill instructions, which are prose, not code under test here. Revisit if any of these skills gains an executable test harness.

## Open Questions

- **Resolved:** the "too old / missing subcommand" exit code is **`8`** (`UnsupportedCommand`), implemented in `specscore-cli` (see its `docs/exit-codes.md`).
- The `specscore … new` scaffold verbs for every producer's artifact type (seed, feature, plan, project bootstrap) must exist before each producer's conversion lands; `idea new` is confirmed. Where a verb is missing, that producer's conversion blocks on `specscore-cli` work. This is a sequencing dependency, not a design open question.
- `init` is both a wizard and a producer; its interactive bootstrap is a wizard behavior while its artifact scaffolding follows the creation mandate. Whether the cold-start (brand-new repo, no CLI) install-then-retry is acceptable in practice is validated during `init`'s conversion.
- How the CLI sources its scaffold templates (e.g. from specscore.md with an embedded offline fallback) is **out of scope** here — it is internal CLI behavior tracked in the `specscore` / `specscore.md` repos.
- Whether to later add a lint rule flagging `command -v specscore` or an embedded schema block for automatic drift detection (out of scope here; prose enforcement chosen).

## Sidekick Seeds Generated

- [specscore-cli-needs-a-seed-scaffold-verb-e-g-specscore-seed](../../../../specscore-cli/spec/ideas/seeds/specscore-cli-needs-a-seed-scaffold-verb-e-g-specscore-seed.md) — captured 2026-06-05 by specstudio:plan

---
*This document follows the https://specscore.md/feature-specification*
