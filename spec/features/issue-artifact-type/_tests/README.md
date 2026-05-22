# Rehearse Tests: Issue Artifact Type

**Status:** Stable
**Date:** 2026-05-22
**Owner:** alexandertrakhimenok

## Contents

| Stub | REQ | AC |
|------|-----|-----|
| [valid-minimal-open-issue.md](valid-minimal-open-issue.md) | issue-frontmatter-required-fields, issue-lifecycle-default-state-on-capture | valid-minimal-open-issue |
| [missing-required-field-rejected.md](missing-required-field-rejected.md) | issue-frontmatter-required-fields | missing-required-field-rejected |
| [valid-optional-fields-accepted.md](valid-optional-fields-accepted.md) | issue-frontmatter-optional-fields | valid-optional-fields-accepted |
| [severity-invalid-enum-rejected.md](severity-invalid-enum-rejected.md) | issue-frontmatter-optional-fields | severity-invalid-enum-rejected |
| [first-seen-wrong-type-rejected.md](first-seen-wrong-type-rejected.md) | issue-frontmatter-optional-fields | first-seen-wrong-type-rejected |
| [optional-field-empty-string-rejected.md](optional-field-empty-string-rejected.md) | issue-frontmatter-optional-fields | optional-field-empty-string-rejected |
| [investigating-without-severity-rejected.md](investigating-without-severity-rejected.md) | issue-severity-required-on-transition | investigating-without-severity-rejected |
| [resolved-without-severity-rejected.md](resolved-without-severity-rejected.md) | issue-severity-required-on-transition | resolved-without-severity-rejected |
| [rejected-without-severity-rejected.md](rejected-without-severity-rejected.md) | issue-severity-required-on-transition | rejected-without-severity-rejected |
| [rejected-without-reason-rejected.md](rejected-without-reason-rejected.md) | issue-rejection-reason-required | rejected-without-reason-rejected |
| [rejection-reason-outside-rejected-rejected.md](rejection-reason-outside-rejected-rejected.md) | issue-rejection-reason-required | rejection-reason-outside-rejected-rejected |
| [invalid-rejection-reason-value-rejected.md](invalid-rejection-reason-value-rejected.md) | issue-rejection-reason-required | invalid-rejection-reason-value-rejected |
| [affected-component-references-real-feature.md](affected-component-references-real-feature.md) | issue-affected-component-validation | affected-component-references-real-feature |
| [optional-affected-component-absent-passes.md](optional-affected-component-absent-passes.md) | issue-affected-component-validation | optional-affected-component-absent-passes |
| [feature-scoped-issue-valid.md](feature-scoped-issue-valid.md) | issue-dual-location | feature-scoped-issue-valid |
| [issue-at-invalid-path-rejected.md](issue-at-invalid-path-rejected.md) | issue-dual-location | issue-at-invalid-path-rejected |
| [slug-mismatch-rejected.md](slug-mismatch-rejected.md) | issue-slug-derivation | slug-mismatch-rejected |
| [slug-truncation-at-60-chars.md](slug-truncation-at-60-chars.md) | issue-slug-derivation | slug-truncation-at-60-chars |
| [duplicate-slug-across-locations-rejected.md](duplicate-slug-across-locations-rejected.md) | issue-slug-globally-unique | duplicate-slug-across-locations-rejected |
| [invalid-status-value-rejected.md](invalid-status-value-rejected.md) | issue-lifecycle-state-values | invalid-status-value-rejected |
| [missing-required-h2-section-rejected.md](missing-required-h2-section-rejected.md) | issue-body-required-h2-sections | missing-required-h2-section-rejected |
| [out-of-order-h2-sections-rejected.md](out-of-order-h2-sections-rejected.md) | issue-body-required-h2-sections | out-of-order-h2-sections-rejected |
| [empty-required-h2-section-rejected.md](empty-required-h2-section-rejected.md) | issue-body-required-h2-sections | empty-required-h2-section-rejected |
| [extra-h2-sections-allowed.md](extra-h2-sections-allowed.md) | issue-body-required-h2-sections | extra-h2-sections-allowed |
| [duplicated-required-h2-section-rejected.md](duplicated-required-h2-section-rejected.md) | issue-body-required-h2-sections | duplicated-required-h2-section-rejected |
| [invalid-h1-rejected.md](invalid-h1-rejected.md) | issue-h1-title | invalid-h1-rejected |
| [root-index-required-when-issues-present.md](root-index-required-when-issues-present.md) | issue-root-index-required | root-index-required-when-issues-present |
| [feature-scoped-index-required-when-issues-present.md](feature-scoped-index-required-when-issues-present.md) | issue-feature-scoped-index-required | feature-scoped-index-required-when-issues-present |
| [index-missing-required-column-rejected.md](index-missing-required-column-rejected.md) | issue-index-contents-columns | index-missing-required-column-rejected |
| [bugs-field-empty-list-accepted.md](bugs-field-empty-list-accepted.md) | issue-bugs-field-opaque | bugs-field-empty-list-accepted |
| [bugs-field-opaque-strings-accepted.md](bugs-field-opaque-strings-accepted.md) | issue-bugs-field-opaque | bugs-field-opaque-strings-accepted |
| [bugs-field-wrong-type-rejected.md](bugs-field-wrong-type-rejected.md) | issue-bugs-field-opaque | bugs-field-wrong-type-rejected |
| [bugs-field-non-string-element-rejected.md](bugs-field-non-string-element-rejected.md) | issue-bugs-field-opaque | bugs-field-non-string-element-rejected |

## Open Questions

None at this time.

---
*This document follows the https://specscore.md/scenarios-index-specification*
