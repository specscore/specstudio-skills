---
format: https://specscore.md/feature-specification
status: Archived
---

# Feature: Review Skill

> [SpecScore.**Studio**](https://specscore.studio): | [Explore](https://specscore.studio/app/github.com/specscore/specstudio-skills/spec/features/skills/review?op=explore) | [Edit](https://specscore.studio/app/github.com/specscore/specstudio-skills/spec/features/skills/review?op=edit) | [Ask question](https://specscore.studio/app/github.com/specscore/specstudio-skills/spec/features/skills/review?op=ask) | [Request change](https://specscore.studio/app/github.com/specscore/specstudio-skills/spec/features/skills/review?op=request-change) |

**Status:** Archived
**Source Ideas:** —
**Archive Reason:** Superseded by reviewer-gates — reviews are stage-internal under each producer's gate; no standalone review skill is required.

## Summary

The `review` skill is intended to perform multi-axis code review (correctness, readability, architecture, security, performance) of an implementation against the Feature it claims to satisfy. The skill is not yet defined — no Idea has been written, and the scope below is a placeholder to be refined via `specstudio:ideate`.

## Problem

Generic code-review skills exist (`agent-skills:code-review-and-quality`, `code-review:code-review`), but they evaluate code against general principles, not against the specific Feature contract the code claims to satisfy. As a result, reviews can pass even when the implementation drifts from the spec — and reviews can fail on stylistic grounds even when the implementation is faithful.

A spec-aware `review` skill exists to evaluate code against its source Feature first, and against general engineering quality second.

## Behavior

The skill is *intended* to: load the Feature, the Plan, the commits, the `verify` report, and the `recap` artifact, then perform a structured review with the Feature as the primary reference point. Findings should be categorized by axis (correctness, readability, architecture, security, performance) and by severity, with each finding referencing either an AC ID or a general principle.

This is intent, not contract. The actual scope, axes, severity scale, and gate semantics need to be defined through `specstudio:ideate`.

## Acceptance Criteria

Not defined yet.

## Open Questions

- Acceptance criteria not yet defined for this feature.
- Does `review` produce a verdict (pass/fail/conditional) or only findings (leaving the verdict to a human)?
- How does `review` interact with human reviewers — replace, augment, or always defer to?
- Should `review` block `ship` automatically on a failing axis, or only flag and let the user decide?
- How does `review` handle findings on code that satisfies the spec but is poorly written (or vice versa)?
- Does `review` run as a single pass or as one pass per axis, with axis-specific sub-skills?

---
*This document follows the https://specscore.md/feature-specification*
