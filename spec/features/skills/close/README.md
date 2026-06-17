---
format: https://specscore.md/feature-specification
status: Approved
---

# Feature: Close

> [SpecScore.**Studio**](https://specscore.studio): | [Explore](https://specscore.studio/app/github.com/specscore/specstudio-skills/spec/features/skills/close?op=explore) | [Edit](https://specscore.studio/app/github.com/specscore/specstudio-skills/spec/features/skills/close?op=edit) | [Ask question](https://specscore.studio/app/github.com/specscore/specstudio-skills/spec/features/skills/close?op=ask) | [Request change](https://specscore.studio/app/github.com/specscore/specstudio-skills/spec/features/skills/close?op=request-change) |
**Status:** Approved
**Date:** 2026-06-16
**Owner:** alexander.trakhimenok
**Source Ideas:** —
**Supersedes:** —

## Summary

The `close` skill retires a SpecScore artifact — an Idea, Feature, or sidekick seed — by driving the appropriate `specscore <kind> change-status` CLI verb with a `--note` reason, **never** by hand-editing. It gives "retire this artifact" one deliberate home, parallel to `ship` and `recap`. Negative transitions (e.g. `Rejected`) require a reason.

## Problem

SpecScore artifacts reach end-of-life — shipped, superseded, rejected, or parked — but there is no single skill to retire one cleanly. Agents and users hand-edit the body `**Status:**` line or frontmatter `status:`, which (a) skips the CLI state-machine validation (a hand-edit can make an illegal jump silently), (b) forgets index/mirror sync (producing `status-mirror` / `*-index-row-sync` lint errors that then need `migrate`/`--fix` recovery), and (c) violates the SpecScore tenet that every status transition goes through a CLI verb. The *reason* for a closure is also lost unless someone separately edits the body. This was felt directly during seed-queue triage: implemented seeds had no clean close path, and the one closed seed in the wild used a hand-edited `status: completed`.

`close` drives the right `specscore <kind> change-status` verb and records the reason in the **same call** (one invocation, two actions). Captured in seed `should-there-be-a-close-lifecycle-skill-that-retires-an`; the seed-kind path depends on the `cli/sidekick/change-status` verb (specscore-cli#72), and the optional/required `--note` behavior depends on the `lifecycle-transitions` `optional-transition-note` / `reason-required-transitions` REQs.

## Behavior

### Resolving the artifact and verb

#### REQ: resolve-artifact-kind

The skill MUST accept one artifact reference (a slug, feature-id, or path) and resolve its kind by location: Idea (`spec/ideas/<slug>.md`), Feature (`spec/features/<id>/README.md`), or sidekick seed (`spec/ideas/seeds/<slug>.md`). The resolved kind selects the CLI verb — `specscore idea|feature|sidekick change-status`. If the reference resolves to no artifact, or to more than one kind ambiguously, the skill MUST stop and ask the user rather than guess.

### Driving the CLI, never hand-editing

#### REQ: cli-only-transition

The skill MUST perform every status transition exclusively via `specscore <kind> change-status <id> --to=<status> [--note <markdown>]`. It MUST NOT edit the body `**Status:**` line, frontmatter `status:`, the `type:` key, or any index row directly — not as a primary path and not as a fallback on CLI failure. This is the skill's load-bearing invariant; a hand-edit anywhere is a contract violation.

#### REQ: confirm-terminal-before-close

Closing is terminal. Before invoking the verb, the skill MUST present the candidate terminal status(es) for the resolved kind (seed → `Implemented` | `Rejected` | `Archived`; Idea → its legal terminals; Feature → `Deprecated`) and obtain an explicit user choice/confirmation. It MUST NOT auto-select a terminal status.

### Reason capture

#### REQ: reason-for-negative-transitions

For a negative transition (e.g. a seed or Idea closed as `Rejected`), the skill MUST obtain a reason and pass it via `--note`. It MUST NOT invoke the verb for a reason-required transition without a reason (the CLI also enforces this with exit `2`; the skill obtains it first to avoid the round-trip). For non-negative transitions (`Implemented`, `Archived`), the skill SHOULD offer to record an optional `--note` (the rationale or where the work shipped).

The reason source depends on the invoker: when `close` is driven by an interactive human, it collects the reason from the user up-front. When `close` is invoked by **another skill** (non-interactive), the reasoning MUST be supplied by the caller as an explicit reason argument — the skill MUST NOT fabricate or auto-generate a reason for a reason-required transition; if the caller passed none, it MUST refuse and surface the missing-reason requirement to the caller. The caller owns the reasoning, so no plausible-but-wrong justification is ever written into the artifact.

### Exit-status handling

#### REQ: branch-on-cli-exit

The skill MUST branch on the verb's exit status and never fall back to a hand-edit on any non-zero: `0` → success (surface the `<id>: <from> → <to>` line); `1` → archive collision (surface, stop); `2` → invalid args or a missing required reason (collect the reason and retry, or correct `--to`); `3` → artifact not found (surface, stop); `4` → illegal transition (surface the current status and legal source set, stop); `10` → rollback applied (surface the lint/IO error, stop); `127` → CLI missing (`/specscore:install`, then retry); `8` → upgrade-then-retry.

### Kind availability

#### REQ: seed-close-requires-verb

Closing a **seed** depends on the `cli/sidekick/change-status` verb (specscore-cli#72). Until that verb is available in the active `specscore` binary, the skill MUST detect its absence (exit `127`/`8` on the `sidekick change-status` call) and surface install/upgrade guidance — it MUST NOT hand-edit the seed as a workaround. Idea and Feature closes are unaffected (their `change-status` verbs already exist).

## Acceptance Criteria

### AC: resolves-kind-and-verb (verifies REQ:resolve-artifact-kind)

**Scenario:** seed reference selects the sidekick verb
**Given** a reference to a seed at `spec/ideas/seeds/foo.md`
**When** `close` runs
**Then** it selects `specscore sidekick change-status` (not `idea` or `feature change-status`) as the transition verb.

### AC: never-hand-edits (verifies REQ:cli-only-transition)

**Scenario:** CLI is the only mutation path
**Given** any `close` invocation that performs a transition
**When** the status change is applied
**Then** the only mutation is a `specscore <kind> change-status` call, and the skill performs no direct edit of the `**Status:**` line, frontmatter `status:`/`type:`, or index rows — including when the CLI returns a non-zero exit.

### AC: confirms-terminal-before-close (verifies REQ:confirm-terminal-before-close)

**Scenario:** explicit confirmation before a terminal transition
**Given** a resolved seed and the candidate terminals `Implemented | Rejected | Archived`
**When** `close` is about to transition it
**Then** it presents the candidate terminals and waits for an explicit user choice before invoking the verb; it never auto-selects.

### AC: collects-reason-for-rejected (verifies REQ:reason-for-negative-transitions)

**Scenario:** negative transition requires a reason
**Given** the user wants to close a seed as `Rejected`
**When** `close` runs
**Then** it collects a reason and passes it via `--note`; with no reason it does not invoke the verb.

### AC: surfaces-illegal-transition (verifies REQ:branch-on-cli-exit)

**Scenario:** CLI rejects an illegal transition
**Given** `specscore sidekick change-status` returns exit `4` (illegal transition)
**When** `close` runs it
**Then** `close` surfaces the current status and the legal source set and stops, without attempting a hand-edit.

### AC: seed-close-blocked-without-verb (verifies REQ:seed-close-requires-verb)

**Scenario:** the seed verb is not yet installed
**Given** the active `specscore` binary has no `sidekick change-status` verb (exit `127`/`8`)
**When** `close` targets a seed
**Then** it surfaces install/upgrade guidance and does not hand-edit the seed; an Idea or Feature close in the same situation still proceeds via its existing verb.

### AC: closes-via-cli-on-success (verifies REQ:branch-on-cli-exit)

**Scenario:** successful transition is surfaced verbatim
**Given** a confirmed close of seed `foo` as `Implemented` with the CLI available
**When** `close` invokes `specscore sidekick change-status foo --to=implemented` and it exits `0`
**Then** `close` surfaces the verb's `foo: Queued → Implemented` success line and performs no further file mutation.

### AC: ai-caller-must-pass-reason (verifies REQ:reason-for-negative-transitions)

**Scenario:** skill-invoked reason-required close with no caller-supplied reason
**Given** `close` is invoked by another skill (non-interactive) to close a seed as `Rejected` with no reason argument
**When** `close` runs
**Then** it refuses, surfaces that the caller must supply a reason for the reason-required transition, and neither invokes the verb nor fabricates a reason.

## Rehearse Integration

The ACs exercise the skill's orchestration (kind resolution, CLI dispatch, exit-status branching, confirmation/reason capture) rather than a pure-function or CLI surface of its own. Standalone Rehearse `_tests/` stubs are deferred — mirroring `ship`/`recap` — because the load-bearing behavior is verified against the real `specscore <kind> change-status` verbs at implementation time. Revisit if a black-box scenario suite for the skill is wanted.

## Not Doing

- **Bulk close.** Closing several artifacts in one invocation (e.g. the 5 implemented dogfood seeds) is out of scope for the MVP — one artifact per invocation; the batch case is five sequential calls. A future revision MAY add bulk close with its own confirmation and partial-failure semantics.
- **Structured cross-repo dependency.** The seed-close path depends on `cli/sidekick/change-status` in `specscore-cli` (specscore-cli#72). `**Source Ideas:**`/`Depends-On` resolve only same-repo, and structured cross-repo references are not yet supported by lint, so the dependency is recorded in prose (see Problem) rather than a structured field. Revisit when cross-repo reference support lands.

## Open Questions

None — the three questions raised during drafting were resolved at the specify review gate (2026-06-17): the AI-driven-close reason source is folded into [`reason-for-negative-transitions`](#req-reason-for-negative-transitions) (caller must pass an explicit reason; no fabrication); the cross-repo dependency stays in prose; and bulk close is out of scope (see Not Doing).

---
*This document follows the https://specscore.md/feature-specification*
