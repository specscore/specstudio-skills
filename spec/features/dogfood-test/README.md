# Feature: Dogfood Test

> [SpecScore.**Studio**](https://specscore.studio): | [Explore](https://specscore.studio/app/github.com/synchestra-io/specstudio-skills/spec/features/dogfood-test?op=explore) | [Edit](https://specscore.studio/app/github.com/synchestra-io/specstudio-skills/spec/features/dogfood-test?op=edit) | [Ask question](https://specscore.studio/app/github.com/synchestra-io/specstudio-skills/spec/features/dogfood-test?op=ask) | [Request change](https://specscore.studio/app/github.com/synchestra-io/specstudio-skills/spec/features/dogfood-test?op=request-change) |
**Status:** Deprecated
**Source Ideas:** —
**Supersedes:** —

## Summary

Synthetic test fixture exercising `specstudio:implement`'s end-to-end mechanics. NOT a real product Feature — exists solely to drive a tiny three-task Plan through implement so the parallel-dispatch, conflict-detection, Status-write, and staging-discipline machinery can be observed running. Archive (`Status: Deprecated`) after the dogfood completes.

## Problem

`specstudio:implement` was specified, reviewed, and shipped without ever being invoked end-to-end. The fixture defined here lets implement run against a real-but-trivial plan, surfacing any spec-vs-runtime gaps before the skill is trusted on real product Plans.

## Behavior

### Output artifacts

The Feature requires three output artifacts at the repo root, each verifying that implement's mechanism produced what the Plan specified.

#### REQ: contributing-doc

A `CONTRIBUTING.md` file MUST exist at repo root containing, at minimum, a top-level heading `# Contributing`, a `## Overview` section, and a `## Development workflow` section. Content is illustrative; the test verifies presence and structure.

#### REQ: changelog-doc

A `CHANGELOG.md` file MUST exist at repo root containing, at minimum, a top-level heading `# Changelog` and a `## 0.0.5` section heading (matching the current `plugin.json` version).

#### REQ: readme-cross-references

The root `README.md` MUST contain a section heading `## Contributing` with body text that links to both `CONTRIBUTING.md` and `CHANGELOG.md` via relative markdown links.

## Acceptance Criteria

### AC: contributing-doc-exists (verifies REQ:contributing-doc)

**Given** the repo at HEAD,
**When** I read `CONTRIBUTING.md` at repo root,
**Then** the file exists, starts with `# Contributing`, and contains both `## Overview` and `## Development workflow` H2 headings.

### AC: changelog-doc-exists (verifies REQ:changelog-doc)

**Given** the repo at HEAD,
**When** I read `CHANGELOG.md` at repo root,
**Then** the file exists, starts with `# Changelog`, and contains a `## 0.0.5` H2 heading.

### AC: readme-cross-references-both (verifies REQ:readme-cross-references)

**Given** the repo at HEAD,
**When** I read root `README.md`,
**Then** the file contains a `## Contributing` H2 section, and its body text contains both `CONTRIBUTING.md` and `CHANGELOG.md` as link targets (relative markdown link syntax).

## Open Questions

- **Fixture lifecycle.** This Feature exists for one-shot dogfooding. Should it be `**Supersedes:**`-ed by a follow-on, or simply transitioned to `**Status:** Deprecated` after the test completes? Recommendation: Deprecated, with a note linking to the dogfood findings.

---
*This document follows the https://specscore.md/feature-specification*
