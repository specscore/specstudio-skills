# Plan: Reviewer Gates MVP

**Status:** Completed
**Source Feature:** reviewer-gates
**Date:** 2026-05-22
**Owner:** alex
**Supersedes:** —
**Mode:** full

## Summary

Decomposes the [reviewer-gates Feature](../features/reviewer-gates/README.md) into seven linearly-presented tasks that ship the MVP typed-per-stage reviewer-gate contract — schema loader + validator, gate runner with AND-composition and rerun policy, `specstudio:specify` wiring, three doc carve-out/cleanup tasks, and a final visibility-links task. All 16 source-Feature ACs are covered by tasks; none deferred.

## Approach

Tasks 1–2 build the primitive (load-time validator → runtime dispatcher); Task 3 wires the only MVP consumer (`specstudio:specify`), making the new code path live; Tasks 4–6 carve out the legacy Reviewer shape from the spec tree (`third-party-integration`, the `specify` Feature, the draft `review` Feature); Task 7 finishes with visibility cross-links from the root `README.md` and skill docs. The 7 load-time-validation ACs are bundled into Task 1 (one validator code path, fixture-driven scenarios), the 3 dispatch/composition/rerun ACs into Task 2 (one runtime loop), and the 2 visibility ACs into Task 7 (mechanical link inserts).

The dependency graph (declared via `**Depends-On:**`) yields four execution batches: Batch 1 runs Tasks 1, 4, 5, 6 in parallel — Task 1 is the loader code while Tasks 4–6 are independent doc edits on different `spec/features/` files; Batch 2 runs Task 2 (depends on Task 1's loader output); Batch 3 runs Task 3 (depends on Tasks 1 + 2); Batch 4 runs Task 7 (depends on Task 5 because Task 7's visibility-link target — the new `### Reviewer gate` topic in `spec/features/skills/specify/README.md` — is created by Task 5).

## Tasks

### Task 1: Gate-config loader and load-time validator

**Verifies:** reviewer-gates#ac:gates-block-preserved, reviewer-gates#ac:untyped-entry-refused, reviewer-gates#ac:unknown-type-refused, reviewer-gates#ac:ai-entry-shape-violations-refused, reviewer-gates#ac:human-entry-min-approvers-cap, reviewer-gates#ac:human-entry-rejects-prompt, reviewer-gates#ac:missing-gates-block-refuses-with-error
**Status:** done
**Depends-On:** —

Implement a shared gate-config loader that reads `gates.<skill>.reviewers` from `specscore.yaml` and validates every entry per the Feature's REQs: `reviewer-entry-required-fields` (name + type required), `mvp-type-set` (`{ai, human}` only), `ai-entry-shape` (prompt inside repo working tree + documented blocker/advisory taxonomy), `human-entry-shape` (no prompt, `min_approvers: 1` cap), `no-untyped-entry`. On any violation the loader refuses with a clear error pointing at this Feature, dispatches nothing, exits non-zero. Also handles the three missing/empty states from `missing-gates-block-refuses` (no `gates:` key, no sub-key, empty `reviewers: []`). The loader's output is a validated, ordered reviewer-entry list ready for the runner (Task 2).

### Task 2: Gate runner — serial dispatch, AND composition with halt-after-first-failure, rerun policy

**Verifies:** reviewer-gates#ac:serial-dispatch-observed, reviewer-gates#ac:and-composition-blocks-on-any-issues-found, reviewer-gates#ac:rerun-policy-applies-on-structural-fix
**Status:** done
**Depends-On:** 1

Implement the gate runner that consumes Task 1's validated reviewer list and executes the dispatch loop: one reviewer at a time in list order, never concurrent, halt-after-first-`Issues Found` within a pass (per the tightened `and-composition` REQ). `type: ai` entries dispatch via the consumer skill's Agent tool with the prompt file as system prompt; `type: human` entries dispatch via the existing approval-phrase recognizer used by `ideate`/`specify`. Aggregate verdicts under AND; surface the first `Issues Found` finding to the user; never silently downgrade Blocker to Advisory. Implement the rerun-policy: on `Issues Found`, after the user's fix re-dispatch every reviewer that previously returned `Issues Found`, AND every reviewer that previously returned `Approved` if the fix touched a structural section (`## Behavior`, `## Architecture`, `## Acceptance Criteria` for Feature artifacts). **Test-harness for `serial-dispatch-observed`:** use a mocked Agent-tool spy that records dispatch start/end timestamps per reviewer entry, then asserts (a) no two start/end intervals overlap, and (b) start order equals registry order — matches the AC's literal "instrumentation that records dispatch start/end timestamps" language and rules out the looser list-order-only reading. Also add a code comment confirming the per-artifact-type extension story for `rerun-policy` (currently scoped to Feature artifacts; other artifact types use their spec's structurally-load-bearing sections).

### Task 3: Wire `specstudio:specify` to consume `gates.specify`

**Verifies:** reviewer-gates#ac:specify-loads-gate-not-builtin
**Status:** done
**Depends-On:** 1, 2

Modify `skills/specify/SKILL.md` to resolve its reviewer list exclusively from `gates.specify.reviewers` via Task 1's loader and dispatch via Task 2's runner. Remove the hardcoded baseline-reviewer dispatch from the skill body. Remove the separate "User Review Gate" step — the user-approval phrase is now collected via a `type: human` entry in the gate's reviewer list. The existing reviewer prompt at `skills/specify/references/reviewer-prompt.md` stays in place; it is referenced by a `type: ai` entry users add to their own `specscore.yaml`. Update this repo's `specscore.yaml` with a minimal `gates.specify` config that registers the baseline-reviewer entry + a human entry, so dogfooding this Plan's own subsequent `plan.updated` cycles keeps working.

### Task 4: Revise `third-party-integration` — carve out the Reviewer shape

**Verifies:** reviewer-gates#ac:third-party-integration-revised
**Status:** done
**Depends-On:** —

Edit `spec/features/third-party-integration/README.md` in place: (a) remove the six REQs `reviewer-registration-mechanism`, `reviewer-registry-entry-shape`, `reviewer-prompt-location`, `reviewer-contract`, `reviewer-composition`, `reviewer-no-canonical-writes`; (b) remove the AC `reviewer-registration-and-composition`; (c) remove the `### Reviewer shape` section and all references to `reviewers:` in `specscore.yaml`; (d) add a row to the `## Interaction with Other Features` table pointing at `reviewer-gates`; (e) remove the `spec/reviewers/<name>/` row from the path table. Confirm `specscore spec lint` passes after the edit; the Producer + Capability shapes and the snippet-versioning AC remain intact.

### Task 5: Revise the `specify` Feature — remove legacy reviewer REQs and collapse the gate topics

**Verifies:** reviewer-gates#ac:specify-feature-revised
**Status:** done
**Depends-On:** —

Edit `spec/features/skills/specify/README.md` in place: (a) remove the five REQs `reviewer-subagent-required`, `reviewer-baseline-blockers`, `reviewer-extension-hook`, `reviewer-composition`, `user-approval-required`; (b) remove or replace the dependent ACs — at minimum replace `reviewer-then-user` with a single new AC that asserts `gates.specify` consumption; (c) collapse the two topic sections `### Reviewer subagent gate` and `### User Review Gate` into one `### Reviewer gate` topic that delegates to the `reviewer-gates` Feature via a link. All other REQs and ACs in `specify` remain unchanged. Confirm `specscore spec lint` passes.

### Task 6: Archive the draft `review` Feature

**Verifies:** reviewer-gates#ac:review-feature-archived
**Status:** done
**Depends-On:** —

Transition `spec/features/skills/review/README.md` to `**Status:** Archived` via `specscore feature change-status review --to=archived` (or equivalent in-place edit if the CLI requires it for a Draft Feature). Add `**Archive Reason:** Superseded by reviewer-gates — reviews are stage-internal under each producer's gate; no standalone review skill is required.` Confirm `specscore spec lint` passes. No replacement skill is created.

### Task 7: Wire visibility links — root `README.md`, skill docs, features index

**Verifies:** reviewer-gates#ac:root-readme-link-present, reviewer-gates#ac:skill-doc-cross-links-present
**Status:** done
**Depends-On:** 5

Edit four files: (1) repo-root `README.md` — add at least one link whose href resolves to `spec/features/reviewer-gates/README.md`, and update the pipeline-overview sentence to remove the standalone `review` step (since reviews become stage-internal). (2) `skills/specify/SKILL.md` — add at least one link to the `reviewer-gates` Feature in its reviewer-dispatch section. (3) `spec/features/skills/specify/README.md` — add at least one link to the `reviewer-gates` Feature (likely in the new `### Reviewer gate` topic written in Task 5; coordinate to avoid duplicate linking). (4) `spec/features/README.md` — verify the existing index row for `reviewer-gates` (added when this Feature was approved) has a one-line description; if not, edit to include one. Confirm `specscore spec lint` passes.

### Task 8: Grade increment — grade-as-verdict-currency

**Verifies:** reviewer-gates#ac:grade-band-by-blocker-count, reviewer-gates#ac:threshold-default-reproduces-today, reviewer-gates#ac:threshold-resolution-order, reviewer-gates#ac:invalid-threshold-refused, reviewer-gates#ac:lenient-threshold-tolerates-blocker, reviewer-gates#ac:worst-wins-union-across-reviewers, reviewer-gates#ac:within-band-letter-derivation, reviewer-gates#ac:ba-lens-problem-traceability-blocker, reviewer-gates#ac:grade-recorded-on-release
**Status:** done
**Depends-On:** 2

The grade increment added by the grade-as-verdict-currency revision (see the Feature's `### Grade and threshold` topic and `spec/research/reviewer-gates-grade-design.md`). It is a distinct, later implementation pass and is **not** part of the original Batches 1–4 — it begins only after the binary gate (Tasks 1–7) is shipped. Scope: (a) extend the gate runner (Task 2) to compute an A–F grade from the worst-wins union of `Blocker` findings across reviewers and lenses, mapping count→band (`C`=1, `D`=2–3, `F`=4+) and zero-Blocker→pass band (`A`/`B` by within-band judgment) per `grade-band-mapping` and `grade-aggregation`; (b) resolve the Approve threshold per `threshold-config` (per-stage `gates.<stage>.threshold` → top-level `grade.threshold` → default `B`) and validate it at load time per Task 1's loader, deriving the verdict `Approved` iff `grade ≥ threshold` per `threshold-derived-verdict`; (c) author the multi-role reviewer prompt (BA/dev/QA lenses, per-lens sub-assessment, within-band letter, and the BA problem→requirements traceability `Blocker` category) per `multi-role-reviewer`. Test via fixtures: blocker-count→grade mapping, default-threshold migration parity, threshold resolution order, invalid-threshold refusal, lenient-threshold release, worst-wins union, and a mocked multi-role reviewer that emits the BA traceability Blocker. The judgment-quality portion of `ba-lens-problem-traceability-blocker` is validated at the assumption layer, not as a deterministic fixture.

Implementation status (done): the protocol/prose changes are complete — `skills/shared/reviewer-gates/loader.md` (Step 2.5 threshold resolution + validation), `skills/shared/reviewer-gates/runner.md` (Step 2.7 threshold-aware halt, Step 2.8 grade computation + grade-derived verdict), and `skills/specify/references/reviewer-prompt.md` (multi-role BA/dev/QA lenses, within-band letter, problem-traceability Blocker). Rehearse scenario stubs for all eight grade ACs are authored at `spec/features/reviewer-gates/_tests/<ac-slug>.md` (`**Status:** pending`, with Given/When/Then steps); executing them against live fixtures is downstream of this plan.

### Task 9: Event-keyed gates revision

**Verifies:** reviewer-gates#ac:legacy-command-key-rejected, reviewer-gates#ac:deterministic-verdict-from-exit, reviewer-gates#ac:noop-always-approves, reviewer-gates#ac:pre-commit-gate-fires-per-occurrence
**Status:** done
**Depends-On:** 1, 2

Implements the event-keyed revision of the contract added after the original MVP shipped: migrate gate keys from command names to events (`gates.feature.approved`, plus the gate-point events `implementation.pre_commit` / `implementation.pre_push`), reject legacy command-keyed gates with a migration error, and add the `deterministic` (tool-backed; verdict derived from exit code, diagnostics captured as `Blocker`s) and `noop` (always-approve placeholder) reviewer types in the loader/validator (Task 1) and the gate runner (Task 2). Register the gate-point events `implementation.pre_commit` / `implementation.pre_push` in the canonical `skills/shared/events.md` catalog (currently only lifecycle events are listed). Evaluate multi-occurrence gate-point events independently per occurrence (no single-shot caching). Test via fixtures: legacy-command-key rejection, deterministic exit-code→verdict mapping, noop always-approves, and per-occurrence multi-fire of `implementation.pre_commit`. Wiring an actual `implement` run to *fire* these gate-points is a downstream Feature (the implement-autonomy layer), not this task.

## Open Questions

None at this time.

---
*This document follows the https://specscore.md/plan-specification*
