---
format: https://specscore.md/feature-specification
status: Stable
---

# Feature: Issue Artifact Type

> [SpecScore.**Studio**](https://specscore.studio): | [Explore](https://specscore.studio/app/github.com/specscore/specstudio-skills/spec/features/issue-artifact-type?op=explore) | [Edit](https://specscore.studio/app/github.com/specscore/specstudio-skills/spec/features/issue-artifact-type?op=edit) | [Ask question](https://specscore.studio/app/github.com/specscore/specstudio-skills/spec/features/issue-artifact-type?op=ask) | [Request change](https://specscore.studio/app/github.com/specscore/specstudio-skills/spec/features/issue-artifact-type?op=request-change) |
**Status:** Stable
**Date:** 2026-05-22
**Owner:** alexandertrakhimenok
**Source Ideas:** sidekick-issue-tracker-destinations
**Supersedes:** —

## Summary

Introduces `issue` as a top-level SpecScore artifact type for reported observations of broken behavior. Defines the frontmatter schema, required body sections, dual-location path convention (root + Feature-scoped), the four-state lifecycle, and the `I-` lint rule namespace. Foundation Feature — no GitHub dependency, no sidekick changes; scope is the artifact itself.

## Problem

SpecScore today has no first-class artifact for reported issues. The closest neighbor — `sidekick-seed` under `spec/ideas/seeds/` — is idea-shaped: deliberated by the consilium, promoted to Ideas, then to Features. Reported issues have an opposite lifecycle: investigated, resolved, or rejected. Forcing them through the idea pipeline either rubber-stamps them (waste of consilium tokens) or worse, votes them down on cost-benefit grounds (catastrophic — broken behavior isn't a feature request). This Feature creates the right-shaped artifact and lets the rest of the system route to it.

## Behavior

### Artifact Schema

Every `issue` artifact carries a fixed YAML frontmatter block. The schema is split into required-always, required-on-transition, optional, and reserved fields. Lint enforces presence, types, and value constraints.

#### REQ: issue-frontmatter-required-fields

The system MUST require these frontmatter fields on every `issue` artifact at every status: `type` (string, value `issue`), `slug` (string matching the filename slug), `status` (one of `open|investigating|resolved|rejected`), `captured_at` (RFC 3339 / ISO 8601 timestamp), `captured_by` (string).

#### REQ: issue-frontmatter-optional-fields

The system MUST accept these optional frontmatter fields with the documented shape. Absence is valid; presence requires the shape:

- `severity` — string, one of `low|medium|high|critical|unset`. Any other value is a violation.
- `affected_component` — non-empty string (free-form in this MVP; cross-referenced to Features by REQ `issue-affected-component-validation`).
- `first_seen` — non-empty string. MVP accepts free-form (commit SHAs, version tags, dates); shape is not regex-validated. Future tightening is strictly-additive.
- `github_issue` — non-empty string. MVP accepts free-form (URLs like `https://github.com/<org>/<repo>/issues/123` or short forms like `#123`); shape is not regex-validated. Future tightening is strictly-additive.
- `rejection_reason` — string, one of the six values in REQ `issue-rejection-reason-required`.
- `rejection_notes` — free-form string.
- `bugs` — YAML list (possibly empty) whose every element is a string. Element-shape validation is out of MVP per REQ `issue-bugs-field-opaque`.

Any optional field present with a malformed shape (wrong type, empty string where non-empty is required, enum value outside the documented set) is a lint violation.

#### REQ: issue-severity-required-on-transition

The system MUST require `severity` to be set to a non-`unset` value (`low|medium|high|critical`) when `status` is `investigating`, `resolved`, or `rejected`. When `status: open`, `severity` MAY be absent or `unset`. This preserves the sidekick "write-and-continue" discipline at capture (you don't know severity yet) while ensuring triaged issues carry severity.

#### REQ: issue-rejection-reason-required

When `status: rejected`, the system MUST require `rejection_reason` to be set to one of: `not-a-defect`, `wont-fix`, `duplicate`, `not-reproducible`, `by-design`, `deferred`. `rejection_notes` is optional free-form context (e.g. `"duplicate of #142"` for `duplicate`, deferral target version for `deferred`). At any other status, `rejection_reason` and `rejection_notes` MUST be absent.

#### REQ: issue-affected-component-validation

When `affected_component` is present, the system MUST validate that its value matches the directory name of a Feature whose `README.md` exists at `spec/features/<value>/README.md`. If `spec/features/<value>/` exists but `README.md` is absent, the Feature is treated as nonexistent for the purposes of this validation (matching existing Source-Ideas reference-validation behavior). Absence of `affected_component` is valid (means the issue isn't yet triaged to a Feature). This catches typos at lint time and keeps the issue→Feature link real.

### Body Structure

The issue body has a fixed top-level shape: an `# Issue:` H1, then three required H2 sections. Additional H2 sections after these three are allowed (for triage notes, RCA hypotheses, etc.).

#### REQ: issue-h1-title

The H1 of every `issue` artifact MUST match the regex `^# Issue: .+$` — analogous to the existing `# Idea:` and `# Feature:` conventions. The text after `Issue: ` is the human-readable title.

#### REQ: issue-body-required-h2-sections

The system MUST require three H2 sections in every `issue` artifact body, in this order, immediately after the body-metadata block: `## Description`, `## Steps to Reproduce`, `## Expected vs Actual`. Each of the three required sections MUST appear exactly once (no duplicates) and MUST contain at least one non-whitespace character of content below its heading (no empty sections). Additional H2 sections after `## Expected vs Actual` are allowed and unconstrained.

### Path Convention

Issues live at one of exactly two locations. Lint rejects any other placement.

#### REQ: issue-dual-location

The system MUST accept `issue` artifacts at exactly these two location patterns: (a) root-level at `spec/issues/<slug>.md`, or (b) Feature-scoped at `spec/features/<feature-slug>/issues/<issue-slug>.md` where `<feature-slug>` is the directory name of an existing Feature. Any `issue` file matching `type: issue` outside these two patterns is a lint violation.

#### REQ: issue-slug-derivation

The system MUST derive issue slugs using the same algorithm as `sidekick-seed`: (1) lowercase Unicode default casefolding, (2) replace every character outside `[a-z0-9]` with `-`, (3) collapse runs of `-` into a single `-`, (4) trim leading and trailing `-`, (5) if length > 60, truncate to the nearest preceding `-` boundary that produces a slug ≤ 60 chars. The filename `<slug>.md` MUST match the `slug` frontmatter field exactly.

#### REQ: issue-slug-globally-unique

The system MUST enforce globally unique issue slugs across both location patterns combined. Two `issue` artifacts with the same slug — whether both at root, both Feature-scoped under different Features, or one at each — is a lint violation. This invariant makes manual relocation safe: `mv spec/issues/foo.md spec/features/x/issues/foo.md` never changes the slug and never collides.

### Lifecycle States

Issues move through exactly four states. The Feature documents the intended transition graph; in this MVP, lint validates only the *current state* and per-state required fields. Transition validity (e.g. "you can't go `resolved → open` directly") is enforced by the future `specscore issue change-status` CLI verb, out of scope here.

#### REQ: issue-lifecycle-state-values

The system MUST recognize exactly these four values for `status`: `open`, `investigating`, `resolved`, `rejected`. Any other value (including legacy values like `triaged`, `closed`, `fixed`) is a lint violation.

#### REQ: issue-lifecycle-default-state-on-capture

The system MUST allow `status: open` with no other state-dependent fields populated (no `severity`, no `rejection_reason`). This is the default state at sidekick capture time and MUST be lint-clean as a minimal valid issue.

### Indexes

Each location that contains issues has an index. Indexes follow the existing `index` artifact schema and are lint-validated.

#### REQ: issue-root-index-required

When `spec/issues/` contains one or more `issue` artifacts, the system MUST require `spec/issues/README.md` to exist and conform to the SpecScore Index Artifact schema (`type: index`, `**Status:** Stable`, a `## Contents` table, an `## Outstanding Questions` section). An empty `spec/issues/` directory does not require a README.

#### REQ: issue-feature-scoped-index-required

When `spec/features/<feature-slug>/issues/` contains one or more `issue` artifacts, the system MUST require `spec/features/<feature-slug>/issues/README.md` to exist and conform to the SpecScore Index Artifact schema. An empty Feature-scoped `issues/` directory does not require a README.

#### REQ: issue-index-contents-columns

Each `issues/README.md` Contents table MUST have these columns in this order: `Slug` (markdown link to the artifact), `Title` (the H1 text minus the `Issue: ` prefix), `Status`, `Severity` (or `—` when absent/unset), `Captured` (date portion of `captured_at`).

### Reserved Fields

The `bugs` field is reserved on the `issue` artifact for forward-compatibility with the deferred sibling Idea `bug-artifact-and-rca`.

#### REQ: issue-bugs-field-opaque

The system MUST accept a `bugs` frontmatter field on `issue` artifacts as an opaque array of strings. Lint MUST validate that the field, when present, is a YAML list and that every element is a string. Lint MUST NOT validate that the strings resolve to any existing artifact (no `bug` artifact exists yet in this Idea's MVP). This makes the future link strictly-additive: the sibling Idea will tighten the rule from "list of strings" to "list of valid bug slugs" without breaking existing issues.

## Acceptance Criteria

### AC: valid-minimal-open-issue (verifies REQ:issue-frontmatter-required-fields, REQ:issue-lifecycle-default-state-on-capture)

**Given** a file at `spec/issues/menu-crashes-on-empty-input.md` containing frontmatter with `type: issue`, `slug: menu-crashes-on-empty-input`, `status: open`, `captured_at: 2026-05-22T10:00:00Z`, `captured_by: alex`, an `# Issue: Menu crashes on empty input` H1, and the three required H2 sections each with non-empty content
**When** `specscore spec lint` runs
**Then** the file produces zero violations and `status: open` with no `severity` or `rejection_reason` is accepted as valid

### AC: missing-required-field-rejected (verifies REQ:issue-frontmatter-required-fields)

**Given** a `spec/issues/<slug>.md` issue artifact missing one of the required frontmatter fields (`type`, `slug`, `status`, `captured_at`, `captured_by`)
**When** `specscore spec lint` runs
**Then** a violation is emitted with rule id beginning `I-` and a message naming the missing field

### AC: valid-optional-fields-accepted (verifies REQ:issue-frontmatter-optional-fields)

**Given** an otherwise-valid `issue` artifact with `status: investigating`, `severity: high`, `affected_component: sidekick-capture` (where that Feature exists), `first_seen: abc1234`, `github_issue: "#42"`, and `bugs: []`
**When** `specscore spec lint` runs
**Then** zero violations are emitted

### AC: severity-invalid-enum-rejected (verifies REQ:issue-frontmatter-optional-fields)

**Given** an `issue` artifact with `severity: extreme` (a value outside the documented enum)
**When** `specscore spec lint` runs
**Then** a violation is emitted listing the five valid values (`low`, `medium`, `high`, `critical`, `unset`)

### AC: first-seen-wrong-type-rejected (verifies REQ:issue-frontmatter-optional-fields)

**Given** an `issue` artifact with `first_seen: 42` (an integer where a string is required)
**When** `specscore spec lint` runs
**Then** a violation is emitted stating `first_seen` must be a non-empty string

### AC: optional-field-empty-string-rejected (verifies REQ:issue-frontmatter-optional-fields)

**Given** an `issue` artifact with `affected_component: ""` (empty string for a non-empty-string optional field)
**When** `specscore spec lint` runs
**Then** a violation is emitted stating that optional fields, when present, must conform to the documented shape (non-empty string)

### AC: investigating-without-severity-rejected (verifies REQ:issue-severity-required-on-transition)

**Given** an `issue` artifact with `status: investigating` and no `severity` field (or `severity: unset`)
**When** `specscore spec lint` runs
**Then** a violation is emitted with rule id beginning `I-` and a message stating that `severity` is required when status is `investigating`, `resolved`, or `rejected`

### AC: resolved-without-severity-rejected (verifies REQ:issue-severity-required-on-transition)

**Given** an `issue` artifact with `status: resolved` and `severity: unset`
**When** `specscore spec lint` runs
**Then** a violation is emitted naming the same severity-required-on-transition rule

### AC: rejected-without-severity-rejected (verifies REQ:issue-severity-required-on-transition)

**Given** an `issue` artifact with `status: rejected`, `rejection_reason: wont-fix`, and no `severity` field
**When** `specscore spec lint` runs
**Then** a violation is emitted naming the same severity-required-on-transition rule (the missing-severity violation is independent of the rejection-reason-required check)

### AC: rejected-without-reason-rejected (verifies REQ:issue-rejection-reason-required)

**Given** an `issue` artifact with `status: rejected`, `severity: low`, and no `rejection_reason`
**When** `specscore spec lint` runs
**Then** a violation is emitted with rule id beginning `I-` and a message stating that `rejection_reason` is required when `status: rejected`

### AC: rejection-reason-outside-rejected-rejected (verifies REQ:issue-rejection-reason-required)

**Given** an `issue` artifact with `status: investigating` and `rejection_reason: duplicate`
**When** `specscore spec lint` runs
**Then** a violation is emitted stating that `rejection_reason` MUST be absent when `status` is not `rejected`

### AC: invalid-rejection-reason-value-rejected (verifies REQ:issue-rejection-reason-required)

**Given** an `issue` artifact with `status: rejected` and `rejection_reason: not-real-enough` (a string outside the documented enum)
**When** `specscore spec lint` runs
**Then** a violation is emitted listing the six valid values: `not-a-defect`, `wont-fix`, `duplicate`, `not-reproducible`, `by-design`, `deferred`

### AC: affected-component-references-real-feature (verifies REQ:issue-affected-component-validation)

**Given** an `issue` artifact at `spec/issues/foo.md` with `affected_component: nonexistent-feature` and no `spec/features/nonexistent-feature/README.md` exists
**When** `specscore spec lint` runs
**Then** a violation is emitted stating that `affected_component` must reference an existing Feature directory

### AC: optional-affected-component-absent-passes (verifies REQ:issue-affected-component-validation)

**Given** an otherwise-valid `issue` artifact at `spec/issues/foo.md` with no `affected_component` field
**When** `specscore spec lint` runs
**Then** zero violations are emitted

### AC: feature-scoped-issue-valid (verifies REQ:issue-dual-location)

**Given** an `issue` artifact at `spec/features/sidekick-capture/issues/bar.md` with a sibling `spec/features/sidekick-capture/README.md` Feature artifact and a sibling `spec/features/sidekick-capture/issues/README.md` index
**When** `specscore spec lint` runs
**Then** zero violations are emitted on the issue itself

### AC: issue-at-invalid-path-rejected (verifies REQ:issue-dual-location)

**Given** a file at `spec/random-dir/foo.md` containing valid `type: issue` frontmatter
**When** `specscore spec lint` runs
**Then** a violation is emitted stating that `issue` artifacts must live under `spec/issues/` or `spec/features/<slug>/issues/`

### AC: slug-mismatch-rejected (verifies REQ:issue-slug-derivation)

**Given** an `issue` artifact at `spec/issues/foo.md` whose frontmatter `slug` field is `bar` (mismatching the filename)
**When** `specscore spec lint` runs
**Then** a violation is emitted stating that the filename slug and the frontmatter `slug` field must match

### AC: slug-truncation-at-60-chars (verifies REQ:issue-slug-derivation)

**Given** a one-liner `"The application crashes intermittently when the user navigates between menus quickly"` whose naive lowercased-and-dashed form (`the-application-crashes-intermittently-when-the-user-navigates-between-menus-quickly`) exceeds 60 chars
**When** the slug-derivation algorithm runs
**Then** the resulting slug is `the-application-crashes-intermittently-when-the-user` — truncated at the nearest preceding `-` boundary that produces a slug ≤ 60 chars, with no trailing partial word and no trailing `-`

### AC: duplicate-slug-across-locations-rejected (verifies REQ:issue-slug-globally-unique)

**Given** an issue at `spec/issues/foo.md` AND an issue at `spec/features/sidekick-capture/issues/foo.md`, both lint-valid in isolation
**When** `specscore spec lint` runs
**Then** a violation is emitted naming both files as duplicate slugs and identifying `foo` as the colliding slug

### AC: invalid-status-value-rejected (verifies REQ:issue-lifecycle-state-values)

**Given** an `issue` artifact with `status: triaged` (a legacy value not in the four-state set)
**When** `specscore spec lint` runs
**Then** a violation is emitted listing the four valid status values: `open`, `investigating`, `resolved`, `rejected`

### AC: missing-required-h2-section-rejected (verifies REQ:issue-body-required-h2-sections)

**Given** an `issue` artifact whose body contains `## Description` and `## Steps to Reproduce` but lacks `## Expected vs Actual`
**When** `specscore spec lint` runs
**Then** a violation is emitted stating the required H2 section is missing

### AC: out-of-order-h2-sections-rejected (verifies REQ:issue-body-required-h2-sections)

**Given** an `issue` artifact whose body contains the three required H2 sections in the order `## Steps to Reproduce`, `## Description`, `## Expected vs Actual`
**When** `specscore spec lint` runs
**Then** a violation is emitted stating the required sections must appear in canonical order

### AC: empty-required-h2-section-rejected (verifies REQ:issue-body-required-h2-sections)

**Given** an `issue` artifact with all three required H2 sections present but `## Steps to Reproduce` contains only whitespace
**When** `specscore spec lint` runs
**Then** a violation is emitted stating that required sections must contain non-empty content

### AC: extra-h2-sections-allowed (verifies REQ:issue-body-required-h2-sections)

**Given** an otherwise-valid `issue` artifact with the three required H2 sections plus an additional `## Triage Notes` H2 section
**When** `specscore spec lint` runs
**Then** zero violations are emitted

### AC: invalid-h1-rejected (verifies REQ:issue-h1-title)

**Given** an `issue` artifact whose H1 is `# Bug: Menu crashes` (using `Bug:` instead of `Issue:`)
**When** `specscore spec lint` runs
**Then** a violation is emitted stating the H1 must match `^# Issue: .+$`

### AC: root-index-required-when-issues-present (verifies REQ:issue-root-index-required)

**Given** at least one `issue` artifact in `spec/issues/` and no `spec/issues/README.md`
**When** `specscore spec lint` runs
**Then** a violation is emitted stating that an index README is required

### AC: feature-scoped-index-required-when-issues-present (verifies REQ:issue-feature-scoped-index-required)

**Given** at least one `issue` artifact in `spec/features/sidekick-capture/issues/` and no `spec/features/sidekick-capture/issues/README.md`
**When** `specscore spec lint` runs
**Then** a violation is emitted stating that the Feature-scoped index README is required

### AC: index-missing-required-column-rejected (verifies REQ:issue-index-contents-columns)

**Given** a `spec/issues/README.md` whose `## Contents` table lacks the `Severity` column
**When** `specscore spec lint` runs
**Then** a violation is emitted listing the five required columns in order

### AC: bugs-field-empty-list-accepted (verifies REQ:issue-bugs-field-opaque)

**Given** an `issue` artifact with `bugs: []` in frontmatter
**When** `specscore spec lint` runs
**Then** zero violations are emitted

### AC: bugs-field-opaque-strings-accepted (verifies REQ:issue-bugs-field-opaque)

**Given** an `issue` artifact with `bugs: [some-future-bug-slug, another-unverified-slug]` and no `spec/bugs/` directory exists
**When** `specscore spec lint` runs
**Then** zero violations are emitted (lint does not validate the strings resolve to artifacts in this MVP)

### AC: bugs-field-wrong-type-rejected (verifies REQ:issue-bugs-field-opaque)

**Given** an `issue` artifact with `bugs: "single-string-not-list"` (a string instead of a list)
**When** `specscore spec lint` runs
**Then** a violation is emitted stating that `bugs` must be a YAML list of strings

### AC: bugs-field-non-string-element-rejected (verifies REQ:issue-bugs-field-opaque)

**Given** an `issue` artifact with `bugs: [123, "valid-slug"]` (a list whose first element is an integer)
**When** `specscore spec lint` runs
**Then** a violation is emitted stating that every element of `bugs` must be a string

### AC: duplicated-required-h2-section-rejected (verifies REQ:issue-body-required-h2-sections)

**Given** an `issue` artifact whose body contains `## Description` twice (a duplicate of one required section)
**When** `specscore spec lint` runs
**Then** a violation is emitted stating that each required H2 section MUST appear exactly once

## Architecture & Components

This Feature is implemented entirely inside the `specscore` CLI's lint engine. No new skill, no new artifact directory beyond the path conventions, no GitHub coupling.

- **Lint rule namespace `I-`**: A new family of rule ids (`I-001`, `I-002`, …) covering the requirements above. Rule ids are stable identifiers in lint output; the mapping from REQ to rule id is owned by the CLI implementation and listed in the CLI's rule registry.
- **Path-pattern matchers**: Two glob patterns the lint engine watches for `type: issue` artifacts — `spec/issues/*.md` and `spec/features/*/issues/*.md`. Files matching `type: issue` outside these patterns trigger the dual-location violation.
- **Cross-artifact validation**: The `affected_component` validator consults the existing Feature directory listing. Already a pattern in lint (Source Ideas validation uses the same mechanism). Re-uses existing infrastructure.
- **No new CLI verbs in this Feature**: `specscore issue new`, `specscore issue change-status`, `specscore issue relocate`, and `specscore issue list` are all explicit follow-ups (named in the source Idea's Open Questions). MVP creation path is `specstudio:sidekick --type issue` (specified in the separate `sidekick-destination-routing` Feature).

## Data Flow

1. A user (or sidekick on the user's behalf) writes a file at `spec/issues/<slug>.md` or `spec/features/<feature-slug>/issues/<issue-slug>.md`.
2. The user runs `specscore spec lint`.
3. The lint engine walks the spec tree, identifies `type: issue` files, and applies the `I-` rule family.
4. Violations are reported with rule id, file path, line, and message — same shape as existing lint output.
5. The lint engine separately re-checks index conformance for `spec/issues/README.md` and per-Feature `issues/README.md` files.

There is no runtime data flow beyond lint. The artifact is a passive file; downstream consumers (`specscore feature info`, future `specscore issue list`, the GH mirror in `sidekick-destination-routing`) read the same frontmatter.

## Error Handling & Failure Modes

- **Malformed YAML frontmatter**: caught by the existing frontmatter parser; reported with the existing generic frontmatter-parse rule, not an `I-` rule.
- **File present but `type: issue` absent**: file is ignored by the `I-` rule family (it's not an issue artifact). Other rule families may still apply if the file matches another type.
- **`affected_component` references a Feature directory that exists but lacks `README.md`**: treated as nonexistent (matches existing Source Ideas validation behavior).
- **Empty `spec/issues/` directory with no README**: lint passes (per `issue-root-index-required` — the README is only required when issues are present).
- **Slug collision detection across both location patterns**: requires the lint engine to maintain a slug-to-paths map across the issue type. This is incremental work over the existing per-directory uniqueness check; surface as an enhancement in the implementation plan, not a separate Feature.

## Testing Strategy

Every AC above is testable via the `specscore spec lint` CLI on a fixture-tree input. Per-AC Rehearse stubs are scaffolded under `_tests/` with `**Status:** pending`. The stubs name the fixture-tree shape and the expected lint output. Implementation will satisfy these stubs as the `I-` rule family lands.

No Rehearse stub is skipped — every AC has a fixture-tree shape that can be set up and a lint-output assertion that can be checked.

## Not Doing / Out of Scope

- **The `bug` artifact type and the RCA pipeline.** Deferred to sibling Idea `bug-artifact-and-rca`. The `issue.bugs[]` reserved field is forward-compatibility only; lint validates shape (list of strings) but not link integrity.
- **`specscore issue new` CLI scaffolder.** In MVP, sidekick (`--type issue`) is the only blessed creator. Manual file creation works but the user must conform to the schema. CLI scaffolder is a clean follow-up.
- **`specscore issue change-status` CLI with transition enforcement.** Lint in this Feature validates only the *current state* and per-state required fields. Manual frontmatter edits between states are unconstrained by lint; the CLI verb (when it ships) will enforce the transition graph.
- **`specscore issue relocate` CLI.** Manual `mv` + lint is the MVP path. The global-slug-uniqueness invariant makes manual relocation safe.
- **`specscore issue list` aggregate view.** Discovery is via `ls spec/issues/` and `ls spec/features/*/issues/`. The CLI verb is a clean follow-up.
- **GitHub Issue mirroring.** Specified in the separate `sidekick-destination-routing` Feature. This Feature defines the artifact; routing is orthogonal.
- **Auto-managed "Known issues" section in parent Feature README.** Open Question in the source Idea; deferred to a future lint `--fix` rule.

## Assumption Carryover

From the source Idea `sidekick-issue-tracker-destinations`:

- **Surviving / strengthened by this Feature:**
  - *Issues and ideas need distinct lifecycles* — codified by the four-state `open|investigating|resolved|rejected` plus `rejection_reason` enum, distinct from the consilium-bound idea lifecycle.
  - *Dual-location earns its keep* — codified by the `issue-dual-location` REQ; the actual validation (≥ 20% relocation, ≥ 20% root-stay) happens during two-week dogfood.
  - *The issue-vs-bug taxonomy is the right split* — codified by the reserved `bugs: []` field. The dogfood signal (≥ 20% of issues get a manually-filled bugs list) is collected via grep over `spec/issues/**/*.md`.

- **Answered or removed by this Feature:**
  - The Idea's Open Question on `rejection_reason` taxonomy is now settled: enum of six values plus optional free-form `rejection_notes`.
  - The Idea's Open Question on `affected_component` validation is now settled: lint-enforced Feature-slug reference when present.
  - The Idea's Open Question on severity-required-at-capture is now settled: optional at capture, required on transition to non-`open`.

- **Deferred to other Features / future Ideas:**
  - GH-issue-title formatting, repo issue-template detection, GH auth failure policy — all live in `sidekick-destination-routing`.
  - The `bug` artifact, the `issue.bugs[]` link integrity, and the RCA pipeline — all in the deferred sibling Idea `bug-artifact-and-rca`.

## Rehearse Integration

Every AC above corresponds to a `_tests/<req-slug>-<ac-slug>.md` Rehearse stub. All stubs are scaffolded `**Status:** pending` — they describe the fixture tree and the expected lint output but do not contain implementations. Implementation lands as the `I-` rule family is built per the implementation plan.

## Open Questions

- **Should `severity: unset` be a distinct value, or just `severity` absent?** Current spec allows both as semantically equivalent for `status: open`. A future cleanup could choose one.
- **Lint perf at scale.** Slug-uniqueness across the full issue tree requires a global scan. For repos with hundreds of issues this is fine; at thousands, it might warrant an index file. Not a problem at MVP scale.

---
*This document follows the https://specscore.md/feature-specification*
