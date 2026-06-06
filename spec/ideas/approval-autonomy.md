---
format: https://specscore.md/idea-specification
status: Specified
---

# Idea: Approval Autonomy for implement & plan (Commit-Autonomous, Push-Gated)

**Status:** Specified
**Date:** 2026-06-02
**Owner:** alexander.trakhimenok
**Promotes To:** approval-autonomy
**Supersedes:** —
**Related Ideas:** —

## Problem Statement

How might we let implement and plan climb autonomously through commit-scope approval gates — stopping for a human only at the push boundary and on genuine anomalies — so prolonged unattended runs are possible without weakening revert-safety?

## Context

Today implement hard-gates per-batch user approval (SKILL step 13) and plan gates on approval, while publication-policy.md deliberately CANNOT release the approval phrase (line 24; CLI REQ no-user-prompting). That invariant blocks unattended execution — the user's actual goal. Two existing pieces make a bounded release tractable: the gates: block (owned by the reviewer-gates Feature) already models the human approval as a typed reviewer entry, and publication-policy already branch-gates push (main/master/release/* denied by default via branch-check). The missing primitive is an opt-in policy that auto-supplies ONLY the human commit-scope consolidated-diff approval for clean batches, preserving one human checkpoint at push and loud halts on anomalies. Neighbors all Implementing: configurable-change-publication-policy, reviewer-gates, specstudio-implement-skill, specstudio-plan-skill.

## Recommended Direction

Introduce an opt-in approval-autonomy policy, configured in specscore.yaml beside/within the gates: block and read by implement and plan. When enabled: (1) commit-scope checkpoints (consolidated batch diff resolving to [stage, commit]) auto-release the HUMAN approval gate, but only when the batch is clean; (2) push stays gated — at run end the skill presents a cumulative review (every auto-committed batch's commits plus consolidated diff) and requires explicit human approval before push, with publication-policy push branch-safety still applied; (3) anomaly-halt — any of sibling line-overlap conflict, BLOCKED subagent, lint failure that --fix did not resolve, or source-Feature drift forces a human stop regardless of autonomy, and after the user fixes it autonomy must be re-armed explicitly before resuming; (4) branch-agnostic in MVP — the irreversible action (push) is already branch-safe in publication-policy, so approval-autonomy stays a pure convenience layer and branch logic is not duplicated. The release is narrowly bounded: it NEVER auto-releases a push gate or an AI reviewer-gate; it only auto-supplies the human commit-scope diff approval for clean batches. This is a principled, minimal amendment to publication-policy's never-skip-approval clause rather than its removal.

## Alternatives Considered

- **Branch-tiered approval (auto-release on worktree/feature, keep the gate on main).** Richer and closer to the original framing, but it duplicates branch logic that already lives in `publication-policy`'s push branch-check, spreads the same concern across two layers, and touches more implementation surface. The irreversible action (push) is already branch-safe, so the branch dimension buys little safety at the approval layer. Lost to branch-agnostic MVP; branch-scoping is a clean additive overlay later.
- **Fold the release into publication-policy (let a configured `commit`/`push` action skip approval).** Rejected: it violates the deliberate "publication policy is never an approval policy" invariant (`publication-policy.md:24`, CLI REQ `no-user-prompting`) and conflates a deterministic action/safety layer with a human-gate layer. Keeping approval-autonomy a separate, narrowly-scoped concept preserves that contract instead of eroding it.
- **A global "auto-approve everything including push" toggle (bypass-style).** Rejected: it removes the one meaningful human checkpoint (push) and would bypass the anomaly halts, making bad autonomous runs publishable. The whole value here is *bounded* autonomy — commit-scope only, push gated, halts preserved.

## MVP Scope

A spike across two skills and one config contract: (a) a minimal approval-autonomy config (enable + the bounded commit-autonomous/push-gated semantics) resolvable per project for implement and plan; (b) implement and plan honor it — auto-release clean commit-scope batch gates, accumulate commits, halt loudly on the four anomalies, require explicit re-arm after a fix, and present a single cumulative-review push gate at the end. Branch-agnostic; no branch config, no per-event matrix, no new CLI command.

## Not Doing (and Why)

- Branch-tiered autonomy (auto off-main, gate on main) — deferred; safety already lives at the push gate, branch-scoping is a clean additive overlay later
- Auto-releasing AI reviewer-gates — never; only the human commit-scope consolidated-diff gate is auto-supplied
- Auto-push without human approval — push is always the single human checkpoint; publication-policy branch-safety still applies
- Applying to ideate/specify — MVP is implement + plan only
- Auto-resume after an anomaly fix — explicit re-arm is required so autonomy never silently blows past a problem region
- A standalone CLI autonomy-resolver command — MVP leans on config read by skills; revisit only if determinism demands it, mirroring cli/publication-policy

## Key Assumptions to Validate

| Tier | Assumption | How to validate |
|------|------------|-----------------|
| Must-be-true | `push` is already independently gated and branch-safe in `publication-policy`, so auto-committing without per-batch review stays revertable and cannot publish to a protected branch | Confirm `publication branch-check` denies `main`/`master`/`release/*` by default and that `implement`/`plan` route every push through publication-policy's push safety |
| Must-be-true | The four anomaly conditions (sibling line-overlap conflict, BLOCKED subagent, lint failure `--fix` did not resolve, source-Feature drift) are already detectable at the `implement` checkpoint | Map each to existing `implement` SKILL steps 8–10 and the exception list; confirm `plan` has an equivalent halt surface or scope anomalies to its own gate |
| Should-be-true | A single cumulative review at the push boundary is an acceptable quality gate versus reviewing each batch | Run a multi-batch plan end-to-end; confirm the stacked-diff review is legible and catches the same issues per-batch review would |
| Should-be-true | Explicit re-arm after an anomaly halt is the right friction (vs. auto-resume or full drop-to-manual) | Dogfood: measure whether re-arm feels like the correct "stop, I looked, continue" beat during real runs |
| Might-be-true | Branch-agnostic autonomy is sufficient long-term for most users (branch-scoping never needed) | Defer; revisit only if local commits accumulating on an unintended branch becomes a real annoyance in practice |


## SpecScore Integration

- **New Features this would create:** an `approval-autonomy` Feature in `specstudio-skills` — a shared contract doc (sibling to `skills/shared/publication-policy.md`) defining the config shape, the bounded "auto-supply the human commit-scope diff approval" semantics, the anomaly-halt set, the explicit re-arm rule, and the cumulative-review push gate; plus the rules `implement` and `plan` follow to honor it.
- **Existing Features affected:** `specstudio-implement-skill` (per-batch gate, steps 8–13), `specstudio-plan-skill` (approval gate), `reviewer-gates` (the `gates:` block is the natural config home; the human gate is the entry being auto-released), `change-publication-policy` / `publication-policy.md` (a narrow amendment to the never-skip-approval clause carving out the bounded human commit-scope release; push branch-safety unchanged).
- **Dependencies:** leans on `publication-policy` push branch-check for the safety floor; benefits from but is not blocked by the `cli/publication-policy` Draft Feature being built (approval-autonomy governs the human gate, not commit/push action resolution).

## Open Questions

- Should the bounded release be expressed as a field on the existing `type: human` entry under `gates.<skill>` (e.g., `auto_release: commit-scope`) or as a separate top-level `autonomy:` block? Lean: extend the human reviewer entry, since that is literally the gate being released — decide at specify time with the reviewer-gates owner.
- What config scopes apply (project / user / session), and should they mirror publication-policy's scope ladder for consistency? Decide at specify time.
- Does "cumulative review" reuse `implement`'s existing consolidated-diff presentation aggregated across batches, or warrant a new end-of-run summary view? Lean: reuse and aggregate.
- How does explicit re-arm behave across a chained `plan → implement` run — is autonomy re-armed once for the whole run, or per skill?
- In the current-branch / `main` opt-in topology (see `implement-execution-topology`), the worktree-isolation safety net that justifies commit-autonomy is **gone** — auto-committed work is no longer cheaply revertable. What autonomy, if any, is permitted there? Lean: sharply restrict or disable commit-autonomy in current-branch mode (and hardest on `main`/`master`), so autonomy is only available where isolation backstops it; decide the exact reduced policy at specify time.

---
*This document follows the https://specscore.md/idea-specification*
