# Plan: Issue Artifact Type

**Status:** Approved
**Source Feature:** issue-artifact-type
**Date:** 2026-05-22
**Owner:** alexandertrakhimenok
**Supersedes:** —

## Summary

Decomposes the [issue-artifact-type Feature](../features/issue-artifact-type/README.md) into eleven linearly-ordered tasks that introduce the `issue` top-level artifact to the `specscore` CLI lint engine — discovery, schema validation, lifecycle rules, body structure, path conventions, slug discipline, cross-artifact validation, and index requirements. All 33 source-Feature ACs are covered by tasks; none deferred.

## Approach

Tasks are grouped by lint-rule family. Each family corresponds to one or more contiguous REQs in the Feature and lives in one PR-shaped task. Bundling rules: a task references no more than 5 ACs (Task 7 — body sections — touches 5 because they're all variants of one rule); tasks that share infrastructure (file discovery, frontmatter parse) are sequenced so the foundation lands first. Order respects two inferable dependencies: (1) discovery + required-fields validation (Task 1) is prerequisite for every later rule because every rule reads frontmatter, and (2) slug derivation (Task 9) precedes affected-component cross-validation (Task 10) because the latter relies on a stable slug identity for issue files. ACs that touch multiple REQs (e.g., `valid-minimal-open-issue` covers both `issue-frontmatter-required-fields` and `issue-lifecycle-default-state-on-capture`) land in the task that owns the dominant REQ.

## Tasks

### Task 1: Bootstrap `issue` artifact type, required-fields validation, and status enum

**Verifies:** issue-artifact-type#ac:valid-minimal-open-issue, issue-artifact-type#ac:missing-required-field-rejected, issue-artifact-type#ac:invalid-status-value-rejected

Register `issue` as a new artifact type in the `specscore` CLI's type registry. Add the two path-pattern matchers (`spec/issues/*.md` and `spec/features/*/issues/*.md`) so the lint engine discovers `type: issue` files. Implement the first `I-` rule family covering the always-required frontmatter fields (`type`, `slug`, `status`, `captured_at`, `captured_by`), the four-state `status` enum (`open|investigating|resolved|rejected`), and the lint-clean minimal `status: open` case (no other state-dependent fields populated). Issue rule IDs in this task: `I-001` (required fields), `I-002` (status enum). The "minimal open issue" AC anchors the positive test for the whole family.

### Task 2: Optional-fields shape validation

**Verifies:** issue-artifact-type#ac:valid-optional-fields-accepted, issue-artifact-type#ac:severity-invalid-enum-rejected, issue-artifact-type#ac:first-seen-wrong-type-rejected, issue-artifact-type#ac:optional-field-empty-string-rejected

Implement per-optional-field shape validation: `severity` enum (`low|medium|high|critical|unset`), `affected_component` / `first_seen` / `github_issue` as non-empty strings (MVP accepts free-form; no regex tightening), `rejection_reason` / `rejection_notes` shape, and presence-with-malformed-shape as a violation. Absence of any optional field is valid; presence requires the documented shape. New rule: `I-003` (optional-field shape family). Cross-field validation (e.g. `rejection_reason` requires `status: rejected`) is out of this task — see Tasks 4 and 5.

### Task 3: Reserved `bugs` field opaque validation

**Verifies:** issue-artifact-type#ac:bugs-field-empty-list-accepted, issue-artifact-type#ac:bugs-field-opaque-strings-accepted, issue-artifact-type#ac:bugs-field-wrong-type-rejected, issue-artifact-type#ac:bugs-field-non-string-element-rejected

Add lint validation that, when present, `bugs` is a YAML list whose every element is a string. Empty list is valid. Non-empty list with opaque slug strings is valid (no resolution check — the `bug` artifact type does not exist in this Idea's MVP). String-not-list (`bugs: "single"`) and list-with-non-string-element (`bugs: [42, "valid"]`) are violations. New rule: `I-004` (bugs-field opaque). This is the forward-compatibility surface for the deferred `bug-artifact-and-rca` sibling Idea; lint will tighten from "list of strings" to "list of valid bug slugs" without breaking existing issues.

### Task 4: Severity required on non-`open` transitions

**Verifies:** issue-artifact-type#ac:investigating-without-severity-rejected, issue-artifact-type#ac:resolved-without-severity-rejected, issue-artifact-type#ac:rejected-without-severity-rejected

Implement the state-conditional rule: when `status` is `investigating`, `resolved`, or `rejected`, `severity` MUST be set to a value in `{low, medium, high, critical}` (not absent, not `unset`). When `status: open`, severity is optional. The `rejected` variant tests must pair with a valid `rejection_reason` so the violation is provably attributable to severity, not to rejection-reason absence (the AC fixture explicitly uses this pattern). New rule: `I-005` (severity-required-on-transition).

### Task 5: Rejection reason presence, scope, and enum

**Verifies:** issue-artifact-type#ac:rejected-without-reason-rejected, issue-artifact-type#ac:rejection-reason-outside-rejected-rejected, issue-artifact-type#ac:invalid-rejection-reason-value-rejected

Implement the three rejection-reason rules: (a) `status: rejected` requires `rejection_reason` to be set; (b) `rejection_reason` MUST be absent when `status` is not `rejected` (no orphan rejection reasons on non-rejected issues); (c) `rejection_reason` value MUST be one of `not-a-defect`, `wont-fix`, `duplicate`, `not-reproducible`, `by-design`, `deferred`. `rejection_notes` is optional free-form and MUST be absent when `rejection_reason` is absent. New rule: `I-006` (rejection-reason family).

### Task 6: H1 title regex

**Verifies:** issue-artifact-type#ac:invalid-h1-rejected

Validate that the first H1 in every `issue` artifact body matches the regex `^# Issue: .+$`. Files using `# Bug:` or `# Idea:` or any other prefix fail. Empty title (`# Issue: `) fails. New rule: `I-007` (h1-title). The match is on the first H1; any subsequent H1s in the body are unconstrained (though practically issue bodies have only one H1).

### Task 7: Required body H2 sections — presence, order, content, no duplicates

**Verifies:** issue-artifact-type#ac:missing-required-h2-section-rejected, issue-artifact-type#ac:out-of-order-h2-sections-rejected, issue-artifact-type#ac:empty-required-h2-section-rejected, issue-artifact-type#ac:extra-h2-sections-allowed, issue-artifact-type#ac:duplicated-required-h2-section-rejected

Implement body-structure validation: the issue body MUST contain exactly three required H2 sections in this order immediately after the body-metadata block — `## Description`, `## Steps to Reproduce`, `## Expected vs Actual`. Each MUST appear exactly once (no duplicates) and contain ≥1 non-whitespace character of content below the heading. Additional H2 sections after `## Expected vs Actual` are unconstrained. New rule: `I-008` (body-sections family). Implementation reads the parsed markdown tree from the existing lint engine; no new parser.

### Task 8: Dual-location path validation

**Verifies:** issue-artifact-type#ac:feature-scoped-issue-valid, issue-artifact-type#ac:issue-at-invalid-path-rejected

Validate that every `type: issue` file lives at one of exactly two patterns: `spec/issues/<slug>.md` (root) or `spec/features/<feature-slug>/issues/<issue-slug>.md` (Feature-scoped, where `<feature-slug>` is the directory name of any directory at `spec/features/<feature-slug>/`). A file matching `type: issue` outside these patterns triggers a violation. This task contains the engine integration that makes both location patterns discoverable; the Feature-scoped variant additionally requires the parent Feature directory to exist (but the Feature's `README.md` presence is checked by Task 10, not here — this task validates path shape only). New rule: `I-009` (dual-location).

### Task 9: Slug derivation, filename match, and global uniqueness

**Verifies:** issue-artifact-type#ac:slug-mismatch-rejected, issue-artifact-type#ac:slug-truncation-at-60-chars, issue-artifact-type#ac:duplicate-slug-across-locations-rejected

Implement the slug-derivation algorithm (lowercase Unicode casefolding → `[^a-z0-9]→-` → collapse `-` runs → trim → truncate-at-≤60-at-nearest-`-`-boundary) as a pure function in the slug-utilities package, reused from the existing `sidekick-seed` slug code. Add the rule that the filename slug MUST equal the frontmatter `slug` field. Add a corpus-pass that builds a slug→paths map across both location patterns and emits a violation for any slug appearing more than once. The truncation AC uses the deterministic fixture from the Feature (`"The application crashes…" → "the-application-crashes-intermittently-when-the-user"`). New rules: `I-010` (slug-mismatch), `I-011` (slug-global-uniqueness).

### Task 10: Affected component cross-artifact validation

**Verifies:** issue-artifact-type#ac:affected-component-references-real-feature, issue-artifact-type#ac:optional-affected-component-absent-passes

When `affected_component` is present, validate that `spec/features/<value>/README.md` exists. Directory present without `README.md` is treated as nonexistent (matching the existing Source-Ideas reference-validation behavior). Absence of `affected_component` is valid. Reuse the existing Feature-reference lookup helper from the lint engine (the same one used by Source Ideas / Promotes To resolution). New rule: `I-012` (affected-component-feature-ref). This task is intentionally ordered after Task 9 because slug-uniqueness establishes the per-issue identity invariant that this cross-reference relies on.

### Task 11: Index rules — root, Feature-scoped, and contents columns

**Verifies:** issue-artifact-type#ac:root-index-required-when-issues-present, issue-artifact-type#ac:feature-scoped-index-required-when-issues-present, issue-artifact-type#ac:index-missing-required-column-rejected

Three index rules in one task because they share index-artifact validation infrastructure: (a) when `spec/issues/` contains ≥1 issue artifact, `spec/issues/README.md` MUST exist and conform to the SpecScore Index Artifact schema (`type: index`, `**Status:** Stable`, a `## Contents` table, an `## Open Questions` section); (b) when `spec/features/<feature-slug>/issues/` contains ≥1 issue, the analog Feature-scoped `issues/README.md` MUST exist and conform; (c) each `issues/README.md` Contents table MUST have these columns in this order: `Slug`, `Title`, `Status`, `Severity`, `Captured`. Empty directories don't require a README. New rules: `I-013` (root-index-required), `I-014` (feature-scoped-index-required), `I-015` (index-columns).

## Open Questions

- The Feature's two Outstanding Questions are inherited and not blocking: (1) `severity: unset` vs absent equivalence — Task 2 implements both as semantically equivalent for `status: open` (matches the AC); cleanup to a single canonical form is a follow-up. (2) Lint perf at slug-uniqueness corpus scan — Task 9 implements a single-pass scan; if perf becomes an issue at thousands of issues, a slug index file is the follow-up.
- Which release of the `specscore` CLI ships these rules — out of plan scope; coordinate with the CLI's release schedule.

---
*This document follows the https://specscore.md/plan-specification*
