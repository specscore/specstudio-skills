# Idea: CLI Detection Convention — Match the Check to the Skill's Dependency on the CLI

**Status:** Specified
**Date:** 2026-06-03
**Owner:** alexander.trakhimenok
**Promotes To:** cli-detection-convention
**Supersedes:** —
**Related Ideas:** —

## Problem Statement

How might we let each specstudio skill decide whether to detect, hard-require, or gracefully fall back from the `specscore` CLI based on its actual dependency on the CLI — instead of repeating one probe-then-branch idiom whose correct answer differs per skill?

## Context

A user questioned the `ideate` skill's Step 3a probe (`command -v specscore … && specscore --version`) as excessive: 'assume it exists, call it, and fall back on failure.' Auditing the repo, the same probe appears in 5 skills (ideate, init, sidekick, relocate-idea, consilium) but encodes THREE different intents: (1) CLI optional with a real fallback (ideate→direct write, sidekick→file append); (2) CLI mandatory with NO fallback, where a clean install message beats a leaked `command not found` (relocate-idea, consilium — consilium even checks for specific subcommands, which try-and-fallback cannot do); (3) detection that drives a wizard branch + install offer (init). One mechanism is doing three jobs, and the 'do we need the check' answer differs per job.

## Recommended Direction

Use **one detection mechanism in every skill**, and let only the *response* vary. No skill needs a standalone `command -v` probe: attempt the relevant `specscore` call and branch on its outcome —

- **exit `127`** — binary not on PATH (not installed).
- **other non-zero** — binary present but the call failed; this *includes* "present but too old / missing a required subcommand."
- **success** — proceed.

`command -v specscore` is never *necessary*. `specscore` ships a cheap read-only `--version` you can call to fail fast, and the mandatory/wizard skills have to run the real command anyway. The detection mechanism is therefore uniform; what differs per skill is only the **response policy** to each outcome:

| Skill class | success | `127` (absent) | other non-zero |
|-------------|---------|----------------|----------------|
| **Optional** (`ideate`, `sidekick`) | use CLI | take the fallback | surface error, do **not** fall back |
| **Mandatory** (`relocate-idea`) | proceed | install message → stop | surface error |
| **Capability-gated** (`consilium`) | proceed | install message → stop | dedicated "too old / missing `consilium verdict`" code → upgrade message; else surface |
| **Wizard** (`init`) | parse version, offer update if newer | offer install | surface error |

Two refinements survive the collapse to one mechanism:

1. **`127` only means "binary absent."** "Present but too old / missing a subcommand" is signalled by a *dedicated `specscore` exit code* — a CLI enhancement this Idea **assumes is implemented** (tracked as a seed in the `specscore-cli` repo; see Dependencies). Skills that depend on a specific subcommand (`consilium`) branch on that code to an "upgrade" message, distinct from both `127` (absent) and a generic command failure.
2. **Do the check before expensive or mutating work.** `consilium` should verify its arbiter subcommand *before* running the 9-role panel — a cheap up-front call, not a probe at the end. (A `127` means the command never ran, so for mutating first calls like `ideate`'s `idea new` nothing was written; the fallback stays safe and a partial-write failure surfaces as a non-127 error instead of triggering the fallback. No separate probe buys anything here.)

The deliverable is `skills/shared/cli-detection.md` capturing this one mechanism plus the response table, and a one-line cross-reference from each skill naming its class — not a behavior rewrite of all of them at once.

## Alternatives Considered

- **Blanket "try the call, fall back on any non-zero exit"** (the original literal proposal). Rejected. It cannot distinguish "CLI absent" (`127`) from "CLI present but errored" (1/2/…); the latter would be silently masked by the fallback, hiding real bugs and risking a double-write after a partial mutation.
- **Three distinct *detection* patterns — a separate `command -v` probe for the mandatory/wizard skills, call-and-branch only for the optional ones** (this Idea's own first draft). Rejected. Detection is uniform across all skills; `127`-vs-error-vs-success is sufficient everywhere, and the mandatory/wizard skills run the command regardless. Only the *response* differs, so the variation belongs in a response table, not in three mechanisms.
- **Keep the status-quo probe in every skill.** Rejected. The `command -v` prefix is redundant with the exit code of the real call, and it cannot detect "present but too old" — which the call's own error *can*.
- **A single shared `cli-detect` bash snippet every skill sources identically.** Possibly worth it to prevent copy-paste drift of the branch logic, but it is a packaging choice *under* this convention, not the convention itself (see Open Questions).

## MVP Scope

A one-to-two-day pass: write `skills/shared/cli-detection.md` defining the single call-and-branch mechanism plus the response table, then apply it to **three representative skills** — `ideate` (Optional: replace Step 3a's `command -v` probe with the `127`-branch rule), `relocate-idea` (Mandatory: cite the convention, standardize the install message), and `consilium` (Capability-gated: encode the "non-127 = upgrade" branch and move the capability check ahead of the 9-role panel). Three skills, not two, because `consilium` is the one that exercises the surviving "non-127 means present-but-incapable" nuance — without it the convention is untested on its hardest row. Leave the remaining skills referencing the doc as a follow-up. Done when all three pass `specscore spec lint` and a maintainer agrees the response table covers every current skill that touches the CLI.

## Not Doing (and Why)

- Blanket 'try, fall back on any non-zero exit' — masks real CLI errors and risks partial-mutation double-writes
- Changing specscore CLI exit-code semantics — the convention relies on existing POSIX 127-for-not-found behavior
- Rewriting all 12 skills in one pass — MVP applies the rule to the clearest representatives only

## Key Assumptions to Validate

| Tier | Assumption | How to validate |
|------|------------|-----------------|
| Must-be-true | One call-and-branch mechanism (`127` / other-error / success) plus a per-skill response table covers every current and near-future skill that touches the CLI — none needs a separate detection mechanism | Map all 12 skills' `specscore` usage onto the response table; confirm each is a *response* difference, not a mechanism difference |
| Must-be-true | A not-found `specscore` reliably yields exit `127`, distinct from the CLI's own error codes, across the shells and harness contexts skills actually run in | Invoke a missing binary and a deliberately-erroring `specscore` call under bash/zsh/sh and the agent Bash tool; confirm `127` vs non-127 separation holds |
| Must-be-true | The `specscore` CLI exposes a dedicated, stable exit code for "present but too old / missing subcommand," distinct from `127` and from generic failure (assumed implemented per the `specscore-cli` seed) | Once the CLI ships it, call an unknown subcommand / run an old version; confirm the dedicated code is returned and stable across versions |
| Should-be-true | `skills/shared/cli-detection.md` is the right home (matches the existing `shared/*.md` convention) versus inlining the rule in each skill | Confirm other cross-cutting rules (publication-policy, path-conventions) live in `shared/` and are cited, not duplicated |
| Might-be-true | Skills should share one documented bash snippet for the branch logic to prevent copy-paste drift, rather than each author re-writing it | Defer; revisit once the three MVP skills are converted and any divergence is visible |


## SpecScore Integration

- **New Features this would create:** likely none — a shared reference doc (`skills/shared/cli-detection.md`), not a Feature. Confirm at spec time.
- **Existing Features affected:** the skills that touch the CLI — `ideate`, `sidekick`, `relocate-idea`, `consilium`, `init`, and the other producers (`specify`, `plan`, `implement`) — each gains a one-line cross-reference; `ideate`, `relocate-idea`, and `consilium` change behavior in MVP.
- **Dependencies:** aligns with the existing `shared/*.md` convention (`publication-policy.md`, `path-conventions.md`). **Assumed (cross-repo) dependency:** the `specscore` CLI returns a dedicated, documented exit code for "present but too old / missing subcommand" (outside the shell-reserved set). This Idea **assumes it is implemented**; the CLI change is tracked as a sidekick seed (`specscore-cli-should-return-a-dedicated-documented-exit`) in the `specscore-cli` repo. ("Binary absent" stays the shell's `127`; the CLI cannot influence that and need not try.)

## Open Questions

- Exact home and name: `skills/shared/cli-detection.md`, or fold into an existing shared doc?
- Ship a single documented bash idiom (a copy-paste snippet skills reference) versus prose each author re-implements?
- **Decided:** capability-gated skills detect "present but missing subcommand" via a **dedicated `specscore` exit code** (tracked in the `specscore-cli` seed), not a capability query or error-text heuristic. **Open:** the exact code value (number TBD in the CLI seed; must sit outside the shell-reserved set `0/1/2/126/127/128/130`).
- Should the install message be fully unified (`/specscore:install`) given `consilium` currently emits a bespoke message naming a companion plan? Unify the pointer, keep the case-specific detail?

## Sidekick Seeds Generated

- [specscore-cli-should-return-a-dedicated-documented-exit](../../../specscore-cli/spec/ideas/seeds/specscore-cli-should-return-a-dedicated-documented-exit.md) — captured 2026-06-03 by specstudio:ideate

---
*This document follows the https://specscore.md/idea-specification*
