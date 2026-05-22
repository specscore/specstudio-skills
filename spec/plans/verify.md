# Plan: Verify Skill MVP

**Status:** Completed
**Source Feature:** skills/verify
**Date:** 2026-05-22
**Owner:** alex
**Supersedes:** —

## Summary

Decomposes the [verify Feature](../features/skills/verify/README.md) into seven linearly-ordered tasks that ship the MVP AI-subagent verifier — pre-flight guards, Feature parse + per-AC commit collection, serial subagent dispatch with verdict contract, report writer, exit-code rollup, event emission, and the hard-gated transition to recap. All 13 source-Feature ACs are covered by tasks; none deferred.

## Approach

The seven tasks mirror the seven topic groups in the Feature's `## Behavior` section, with one bundled task per topic that covers every AC inside that topic. Each downstream task depends on artifacts produced by the previous one (the parse output feeds dispatch; dispatch verdicts feed the report; the report path feeds the event), which forces a strict linear order with no parallel branches. ACs that span multiple REQs (e.g., `unmapped-not-fail` covers both `unmapped-detection` and `exit-code-semantics`) are placed in the task that owns the dominant REQ to keep `**Verifies:**` lines clean.

## Tasks

### Task 1: Pre-flight guards

**Verifies:** skills/verify#ac:refuses-draft-feature, skills/verify#ac:refuses-uncommitted-feature

Implement the two pre-dispatch guards: (1) resolve the input slug to `spec/features/<slug>/README.md` and refuse when `**Status:**` is outside `{Approved, Implementing, Stable}`; (2) refuse when `git cat-file -e HEAD:<feature-path>` exits non-zero (Feature exists only in the working tree). Both refusals print a helpful message, write no report, and exit non-zero.

### Task 2: Feature parse and per-AC commit collection

**Verifies:** skills/verify#ac:per-ac-trailer-grep

Parse the Feature via the `specscore` CLI to extract the ordered list of AC IDs with their `Given/When/Then` text. For each AC, walk `git log --grep='^Verifies:.*<feature-slug>#ac:<ac-slug>'` on the current branch and collect matching commit SHAs (chronological) paired with their commit messages. Do NOT pre-fetch commit diffs.

### Task 3: Verifier subagent dispatch + verdict contract

**Verifies:** skills/verify#ac:subagent-serial-dispatch, skills/verify#ac:subagent-prompt-shape, skills/verify#ac:malformed-verdict-retried-once

Implement the serial-dispatch loop that calls one verifier subagent per mapped AC in Feature order, never two concurrently. Build the per-AC prompt with the four required parts (AC G/W/T text, commit SHAs + messages, verbatim verdict contract, fetch-on-demand Bash instruction) and zero pre-fetched diffs. Parse the subagent's response per the verdict contract; on malformed response, re-dispatch exactly once with a corrective prompt; on a second malformed response, record the AC with `verdict: error`.

### Task 4: Report generation and staging

**Verifies:** skills/verify#ac:report-path-and-staging, skills/verify#ac:report-yaml-block-grep-target, skills/verify#ac:report-index-readme-created-and-updated

Write the per-run report to `spec/features/<feature-slug>/_verify/<sha>.md` (create `_verify/` if absent) where `<sha>` is the abbreviated HEAD SHA. The report MUST open with a fenced ```yaml summary block listing each AC in Feature order with `ac`, `verdict`, and `justification` fields, followed by one `## AC: <ac-slug>` body section per AC with verdict, full justification, commit list, and evidence references. Create `_verify/README.md` if absent or append a row to its `## Contents` table if present (preserving prior rows in order). Stage both the per-run report and the index README via `git add` in the same staging set; never invoke `git commit`.

### Task 5: Exit semantics, unmapped handling, and no-commits edge case

**Verifies:** skills/verify#ac:unmapped-not-fail, skills/verify#ac:exit-non-zero-on-fail-or-error, skills/verify#ac:no-commits-still-reports

Implement the verdict-tally rollup: count `passed_count`, `failed_count`, `unmapped_count`, `errored_count`. ACs with zero matching commits are recorded as `unmapped` (info, not failure). The skill exits non-zero if and only if `failed_count + errored_count > 0`. When every AC is `unmapped` (e.g., a Feature with no implementation commits yet), the skill still writes the report, stages it, emits the event, and exits zero.

### Task 6: verify.completed event emission

**Verifies:** skills/verify#ac:verify-completed-event-emitted

After the report is written and staged, emit exactly one `verify.completed` event via the convention in `skills/shared/synchestra-events.md`. Payload MUST include `feature_slug`, `revision`, `report_path`, and the four flat count fields (`passed_count`, `failed_count`, `unmapped_count`, `errored_count`) summing to the Feature's total AC count, with no per-AC verdict details.

### Task 7: Hard gate and transition to recap

**Verifies:** skills/verify#ac:transition-to-recap-only

Enforce the hard gate: the skill MUST NOT invoke `ideate`, `specify`, `plan`, `implement`, `review`, `ship`, `writing-plans`, `frontend-design`, or `mcp-builder` at any point. On a successful run, transition only to `specstudio:recap`; while `recap` is unshipped, hand back to the user with the report path and a recommendation to review verdicts manually.

## Open Questions

- Token-budget guardrail for the verifier subagent prompt: an AC with many matched commits could theoretically blow context even without pre-fetched diffs, if the commit-message list itself is large. Implementation should track per-AC prompt size and surface a warning if it exceeds a sensible cap (e.g., 100KB). Not blocking the Plan — surfaces at Task 3 implementation time.
- The `_tests/` Markdown scenarios' role in verification remains unresolved (deferred per the Feature's Open Question to a future Idea). The MVP Plan does NOT include any task that reads `_tests/` files; the subagent prompt does not reference them. This is consistent with the Feature spec but worth tracking until the deferred Idea ships.

---
*This document follows the https://specscore.md/plan-specification*
