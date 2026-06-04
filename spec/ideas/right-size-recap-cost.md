# Idea: Right-Size Recap Cost (Configurable Skip + Incremental Re-Recap)

**Status:** Specified
**Date:** 2026-06-04
**Owner:** alexander.trakhimenok
**Promotes To:** skills/ship
**Supersedes:** —
**Related Ideas:** —

## Problem Statement

How might we let low-stakes projects avoid recap's per-AC token cost without silently dropping the drift gate everyone else depends on?

## Context

Triggered by a user complaint: ship's recap gate (ship/SKILL.md Pre-Flight step 4, AC refuses-on-recap-contradiction) is 'strictly mandatory ... no waiver path in the MVP'. Running specstudio:recap dispatches one AI subagent per AC, serially (recap/SKILL.md step 5), so a 12-AC feature is 12 sequential subagent calls — the real token sink. The phrase 'in the MVP' signals a waiver was deliberately deferred, not rejected.

## Recommended Direction

Pursue two complementary levers under one goal — right-sizing recap cost for the value it returns. MVP: a configurable, explicit, logged waiver of ship's recap-no-contradiction gate, defaulting to required, with verify-green left untouched as the correctness floor. Fast-follow: incremental re-recap that only narrates ACs whose mapped commit set changed since the last recap, carrying prior verdicts forward — cutting cost for everyone while keeping the gate.

## Alternatives Considered

- **Skip-only, no cheaper recap.** Just add the waiver and stop. Lost because it serves only the solo/low-stakes case and leaves every other project paying full per-AC token cost on every ship — it fixes the symptom for some and ignores the cause for all. Folded in as the fast-follow rather than dropped.

- **Cheaper-recap-only (no waiver).** Make recap incremental/batched but keep it mandatory. Lost as the *sole* answer: a cheaper-but-still-required recap still imposes non-zero cost and friction on a solo maintainer who genuinely does not want a drift verdict. The stated audience wants *zero* recap cost, which only a waiver delivers.

- **Per-run CLI flag (`ship --skip-recap`).** Lost as the primary mechanism: a per-run flag is easy to paste reflexively and leaves no durable project-level record, which fails the "explicit + logged" floor. A persisted `recap:` config policy is auditable in version control and reviewable in PRs. (A per-run override could complement config later, but is not the MVP.)

- **Merge verify + recap into one subagent pass.** Lost on blast radius: verify checks *correctness* (does the code satisfy the AC) and recap checks *drift* (does the code's behavior diverge from what the spec names), and the pipeline keeps them as distinct gates on purpose. Collapsing them couples two independent verdicts and rewrites two skills to save tokens a waiver saves for free.

## MVP Scope

A timeboxed (~1 week) spike: add a top-level recap config policy (default required_for_ship: true) that, when set false, lets ship proceed without a contradiction-free recap report. Verify-green stays mandatory. The waiver is explicit (config only, never inferred) and logged: ship.completed records recap_status: waived|enforced and the publication disclosure names it. No incremental recap yet.

## Not Doing (and Why)

- Putting the waiver flag under the ship: config block — ship/SKILL.md bars-execution-and-orchestration AC says ship config exposes ONLY delegate.skill/args; a recap policy belongs in a recap: block
- Per-AC granular waiver — MVP waives recap whole-feature only; partial skips add config surface without serving the solo/low-stakes case
- Parallel recap subagents — recap's own Not Doing forbids concurrency; out of scope for this Idea
- An auto-skip heuristic — the skip must be an explicit opt-in, never inferred from project size or token budget
- Touching or weakening the verify-green gate — correctness stays mandatory even when drift is waived
- Deleting or deprecating the recap skill — recap stays the default; this only makes it optional in low-stakes contexts

## Key Assumptions to Validate

| Tier | Assumption | How to validate |
|------|------------|-----------------|
| Must-be-true | Recap's per-AC serial subagent cost is material enough that solo/low-stakes users want to avoid it. | Measure token spend for a real recap run across a 5-AC and a 12-AC feature; confirm the magnitude matches the complaint that motivated this Idea. |
| Must-be-true | Solo/low-stakes maintainers accept trading the drift verdict away (keeping only verify-green) for the token savings. | Ask the requesting user directly; confirm verify-green alone is an acceptable correctness floor for their projects before building the waiver. |
| Should-be-true | A persisted `recap:` config policy is discoverable enough that it won't be left `false` by accident in a project that later becomes high-stakes. | Prototype the config + ship disclosure wording; check that a waived recap is visibly surfaced in `ship.completed` and the publication trail at ship time, not buried. |
| Should-be-true | The waiver flag can live in a top-level `recap:` block without violating ship's `bars-execution-and-orchestration` boundary (which restricts only the `ship:` block). | Re-read the Ship Feature's AC text; confirm a recap-policy config outside the `ship:` block does not touch the barred surface, or scope an amendment if it does. |
| Might-be-true | Incremental re-recap can reliably detect "changed ACs" from the mapped commit-set diff between two recap revisions. | Defer until the fast-follow; spike a commit-set comparison against two real recap runs and confirm unchanged-AC verdicts can be safely carried forward. |


## SpecScore Integration

- **New Features this would create:** A recap-gate-policy capability (config schema + ship enforcement behavior) — likely an amendment to the Ship Skill Feature rather than a standalone Feature. The incremental-re-recap fast-follow would amend the Recap Skill Feature.
- **Existing Features affected:** Ship Skill (`refuses-on-recap-contradiction` AC gains a config-gated waiver path; `bars-execution-and-orchestration` boundary must be checked since it constrains the `ship:` block); Recap Skill (only for the fast-follow incremental mode). Repo config schema (`specscore.yaml`) gains a `recap:` block.
- **Dependencies:** None blocking. Conceptually parallels the existing reviewer-gates config pattern (`gates.<stage>.reviewers`) — worth aligning naming/conventions with it.

## Open Questions

- Where does the waiver config live — a new top-level `recap:` block (`required_for_ship: true|false`), or `gates.ship.recap` alongside the reviewer-gate config? The former keeps ship's "only delegate" boundary intact; the latter co-locates it with other ship gates.
- When recap is waived for ship, is recap also skipped at the `review` stage (which sits between recap and ship), or does `review` keep its own recap expectation independently?
- Is config-plus-event-log sufficient for "explicit + logged," or should ship also require a loud per-run acknowledgement when it proceeds on a waived recap?

---
*This document follows the https://specscore.md/idea-specification*
