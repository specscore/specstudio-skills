# Feature: Recap Skill

> [View in SpecScore Studio](https://specscore.studio/project/features?id=specstudio-skills@specscore@github.com&path=spec%2Ffeatures%2Fskills%2Frecap) — graph, discussions, approvals

**Status:** Draft

## Summary

The `recap` skill is intended to summarize what was actually built against what was specified, surfacing spec↔code drift before review. The skill is not yet defined — no Idea has been written, and the scope below is a placeholder to be refined via `specstudio:ideate`.

## Problem

By the time a Feature has been planned, built, and verified, drift between the spec and the implementation has often crept in: a requirement that got reinterpreted, an AC that was satisfied differently than written, an undocumented decision made under pressure. Without an explicit drift-surfacing step, that drift either goes to review unflagged (wasting reviewer attention) or gets silently absorbed into the Feature without a Proposal.

A `recap` step exists to make drift visible as a first-class artifact, before review and before ship.

## Behavior

The skill is *intended* to: load a Feature, the Plan that satisfied it, the commits associated with the Plan's tasks, and the `verify` report — then produce a structured recap that lists what was built, what was specified, and where the two diverge. Drift items should be surfaced as either Proposals (to be incorporated into the Feature) or as risks (to be flagged for review).

This is intent, not contract. The actual scope, drift-detection rules, and output format need to be defined through `specstudio:ideate`.

## Acceptance Criteria

Not defined yet.

## Outstanding Questions

- Acceptance criteria not yet defined for this feature.
- What constitutes "drift" precisely — only behavioral divergence, or also stylistic/architectural choices?
- Does `recap` compare against the Feature at Plan-approval time, or against the current Feature (which may have been updated mid-implementation)?
- Should `recap` automatically draft Proposals for incorporated drift, or always require human authorship?
- How does `recap` handle implementation choices that were never specified (under-specification) vs. choices that contradict the spec (over-specification)?
- Should `recap` run automatically after `build`/`verify`, or always be invoked explicitly?

---
*This document follows the https://specscore.md/feature-specification*
