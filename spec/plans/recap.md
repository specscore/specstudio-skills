# Plan: Recap Skill MVP

**Status:** Completed
**Source Feature:** skills/recap
**Date:** 2026-05-22
**Owner:** alex
**Supersedes:** —

## Summary

Decomposes the [recap Feature](../features/skills/recap/README.md) into seven linearly-ordered tasks that ship the MVP AI-subagent drift narrator — pre-flight guards (including the recap-specific verify-report-presence check), Feature parse + verify-report resolution + per-AC commit collection, serial drift-narrator dispatch with the four-bucket verdict contract, report writer, exit-code rollup with `unmapped` handling, event emission, and the hard-gated transition to review. All 15 source-Feature ACs are covered by tasks; none deferred.

## Approach

The seven tasks mirror the seven topic groups in the Feature's `## Behavior` section, with one bundled task per topic that covers every AC inside that topic. Each downstream task depends on artifacts produced by the previous one (pre-flight gates the run; the parse + verify-report resolution feeds dispatch; dispatch verdicts feed the report; the report path feeds the event), which forces a strict linear order with no parallel branches. ACs that span multiple REQs (e.g., `unmapped-not-contradiction` covers both `unmapped-detection` and `exit-code-semantics`) are placed in the task that owns the dominant REQ to keep `**Verifies:**` lines clean. The structure deliberately parallels the [verify Plan](verify.md) since recap is verify's architectural twin; the only structural addition is the `requires-verify-report` AC inside Task 1 and the verify-report-resolution responsibility inside Task 2.

## Tasks

### Task 1: Pre-flight guards

**Verifies:** skills/recap#ac:refuses-draft-feature, skills/recap#ac:refuses-uncommitted-feature, skills/recap#ac:refuses-when-no-verify-report

Implement the three pre-dispatch guards: (1) resolve the input slug to `spec/features/<slug>/README.md` and refuse when `**Status:**` is outside `{Approved, Implementing, Stable}`; (2) refuse when `git cat-file -e HEAD:<feature-path>` exits non-zero (Feature exists only in the working tree); (3) refuse when `spec/features/<slug>/_verify/` either does not exist or contains zero `<sha>.md` report files reachable at HEAD, recommending `specstudio:verify <slug>` first. All three refusals print a helpful message, write no report, and exit non-zero.

### Task 2: Feature parse, verify-report resolution, and per-AC commit collection

**Verifies:** skills/recap#ac:verify-report-resolved-to-latest-at-head, skills/recap#ac:per-ac-trailer-grep

Parse the Feature via the `specscore` CLI to extract the ordered list of AC IDs with their `Given/When/Then` text. Resolve the verify report by preferring `_verify/<sha>.md` whose `<sha>` matches `git rev-parse --short HEAD`, else falling back to the report whose embedded `revision:` YAML field is most recent in the branch's git history; parse the resolved report's top-of-file YAML summary block and build a per-AC map of `{verify_verdict, verify_justification}`. For each AC, walk `git log --grep='^Verifies:.*<feature-slug>#ac:<ac-slug>'` on the current branch and collect matching commit SHAs (chronological) paired with their commit messages. Do NOT pre-fetch commit diffs. Record the resolved verify report's path and SHA for downstream use in the report YAML's `verify_revision:` field and the event payload's `verify_report_path`.

### Task 3: Drift-narrator subagent dispatch and verdict contract

**Verifies:** skills/recap#ac:subagent-serial-dispatch, skills/recap#ac:subagent-prompt-shape, skills/recap#ac:malformed-drift-verdict-retried-once

Implement the serial-dispatch loop that calls one drift-narrator subagent per mapped AC in Feature order, never two concurrently. Build the per-AC prompt with the five required parts in order (AC G/W/T text + AC ID, the AC's verify verdict and justification snippet carried over verbatim from the resolved verify report, commit SHAs + messages, the verbatim drift verdict contract naming only the four subagent-allowed verdicts `{no-drift, spec-tighter-than-code, code-tighter-than-spec, contradiction}` — never `unmapped` or `error`, and a fetch-on-demand Bash instruction) and zero pre-fetched diffs. Parse the subagent's response per the verdict contract (verdict line, ≤500-char narrative, evidence references); on malformed response, re-dispatch exactly once with a corrective prompt; on a second malformed response, record the AC with drift verdict `error` (orchestrator-only verdict, never named to the subagent).

### Task 4: Report generation and staging

**Verifies:** skills/recap#ac:report-path-and-staging, skills/recap#ac:report-yaml-block-grep-target, skills/recap#ac:report-index-readme-created-and-updated

Write the per-run report to `spec/features/<feature-slug>/_recap/<sha>.md` (create `_recap/` if absent) where `<sha>` is the abbreviated HEAD SHA. The report MUST open with a fenced ```yaml summary block listing every AC in Feature order with `ac`, `verdict`, and `narrative` fields, plus top-level `feature:`, `revision:`, and `verify_revision:` fields. Below the YAML, write one `## AC: <ac-slug>` body section per AC carrying the drift verdict, full narrative, the verify verdict and justification for the same AC (verbatim from the verify report), the commit list, and evidence references. Create `_recap/README.md` if absent — with H1, one-paragraph description, `## Contents` table with columns `Report | Run revision | Verify revision | Drift summary`, `## Open Questions` set to `None at this time.`, and the index-spec footer — or append a row to its `## Contents` table if present (preserving prior rows in order; pick newest-first or newest-last and document the choice in this Approach section if revisited). Stage both the per-run report and the index README via `git add` in the same staging set; never invoke `git commit`.

### Task 5: Exit semantics, unmapped handling, and drift-verdict rollup

**Verifies:** skills/recap#ac:unmapped-not-contradiction, skills/recap#ac:exit-non-zero-on-contradiction-only

Implement the verdict-tally rollup: count `no_drift_count`, `spec_tighter_count`, `code_tighter_count`, `contradiction_count`, `unmapped_count`, `errored_count`. ACs with zero matching commits are recorded as `unmapped` (orchestrator-only, info) — the skill MUST NOT dispatch a subagent for an unmapped AC and MUST NOT treat `unmapped` as a contradiction. The skill exits non-zero if and only if `contradiction_count + errored_count > 0`; `no-drift`, `spec-tighter-than-code`, `code-tighter-than-spec`, and `unmapped` are informational at the recap layer and never contribute to the exit code.

### Task 6: recap.completed event emission

**Verifies:** skills/recap#ac:recap-completed-event-emitted

After the report is written and staged, emit exactly one `recap.completed` event via the convention in `skills/shared/events.md`. Payload MUST include `feature_slug`, `revision` (HEAD SHA), `report_path` (relative to repo root), `verify_report_path` (the resolved `_verify/<sha>.md` path), and the six flat count fields (`no_drift_count`, `spec_tighter_count`, `code_tighter_count`, `contradiction_count`, `unmapped_count`, `errored_count`) each ≥ 0 and summing to the Feature's total AC count. Payload MUST NOT include per-AC drift details — consumers read those from `report_path`.

### Task 7: Hard gate and transition to review

**Verifies:** skills/recap#ac:transition-to-review-only

Enforce the hard gate: the skill MUST NOT invoke `ideate`, `specify`, `plan`, `implement`, `verify`, `ship`, `writing-plans`, `frontend-design`, or `mcp-builder` at any point. On a successful run, transition only to `specstudio:review`; while `specstudio:review` is unshipped, hand back to the user with the report path and a recommendation to inspect drift items manually.

## Open Questions

- Token-budget guardrail for the drift-narrator subagent prompt: an AC with many matched commits plus the verbatim carry-over of the verify verdict and justification could blow context even without pre-fetched diffs. Implementation should track per-AC prompt size and surface a warning if it exceeds a sensible cap (e.g., 100KB). Mirrors verify's open question; surfaces at Task 3 implementation time.
- Newest-first vs newest-last row ordering for `_recap/README.md`'s `## Contents` table is left to Task 4 implementation. Pick one, document the choice inline in `## Approach` here if it's worth pinning, otherwise let the implementer decide and stay consistent across runs.
- The drift-narrator subagent receives the verify verdict + justification snippet verbatim; if dogfood reveals the subagent over-anchors to verify's narrative and misses drift verify itself missed, the carry-over may need to be truncated, paraphrased, or omitted. The Feature's `AC:subagent-prompt-shape` locks verbatim carry-over today; revisiting requires an AC edit. Surfaces at dogfood-validation time, not at Plan time.

---
*This document follows the https://specscore.md/plan-specification*
