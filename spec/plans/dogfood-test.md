# Plan: Dogfood Test

**Status:** Completed
**Source Feature:** dogfood-test
**Date:** 2026-05-19
**Owner:** alexander.trakhimenok
**Supersedes:** —
**Mode:** full

## Summary

Synthetic three-task Plan exercising `specstudio:implement`'s parallel-dispatch, conflict-detection, Status-write, and stage-only mechanics against the [`dogfood-test`](../features/dogfood-test/README.md) Feature. Mode is `full` so subagent prompts include authored task bodies. Tasks 1 and 2 are independent; task 3 depends on both.

## Approach

Two batches:

- **Batch 1** (parallel, no dependencies): Task 1 creates `CONTRIBUTING.md` (verifies `dogfood-test#ac:contributing-doc-exists`); Task 2 creates `CHANGELOG.md` (verifies `dogfood-test#ac:changelog-doc-exists`). Both dispatch as concurrent subagents.
- **Batch 2** (sequential, after batch 1 done): Task 3 edits root `README.md` to add a `## Contributing` section linking both new docs (verifies `dogfood-test#ac:readme-cross-references-both`).

The Plan does not depend on any cross-task file overlap, so the post-batch conflict-detection check (`REQ:conflict-detection-line-overlap`) should pass cleanly.

## Tasks

### Task 1: Create CONTRIBUTING.md

**Verifies:** dogfood-test#ac:contributing-doc-exists
**Status:** done
**Depends-On:** —

Create a new file `CONTRIBUTING.md` at repo root with a top-level `# Contributing` heading, an `## Overview` H2 section briefly explaining the SpecScore Studio contribution model (one-paragraph), and a `## Development workflow` H2 section listing the lifecycle skills (`ideate ⇒ specify ⇒ plan ⇒ implement ⇒ verify ⇒ recap ⇒ review ⇒ ship`) and pointing readers at `PRINCIPLES.md` and `skills/shared/philosophy.md` as required reading.

### Task 2: Create CHANGELOG.md

**Verifies:** dogfood-test#ac:changelog-doc-exists
**Status:** done
**Depends-On:** —

Create a new file `CHANGELOG.md` at repo root with a top-level `# Changelog` heading and a `## 0.0.5` H2 section summarizing this session's user-visible changes: PRINCIPLES.md added, plan Feature revised (Mode/Status/Depends-On), implement skill shipped. One-sentence bullets per item; no version comparison links yet (Keep-a-Changelog format is overkill for v0.0.5).

### Task 3: Add Contributing section to README

**Verifies:** dogfood-test#ac:readme-cross-references-both
**Status:** done
**Depends-On:** 1, 2

Add a new `## Contributing` H2 section to root `README.md`, placed after the existing `## Status` section and before the `## License` section. Body text: one paragraph linking to both `CONTRIBUTING.md` (for the contribution workflow) and `CHANGELOG.md` (for release history) via relative markdown links. Keep it brief — a discoverability pointer, not a manifesto.

## Open Questions

- **Fixture cleanup after dogfood.** This Plan and the dogfood-test Feature are synthetic — they exist to test `implement`, not to persist. After the dogfood completes and findings are recorded, both should be transitioned to `**Status:** Deprecated` (or moved to an archived/ subdirectory). The actual `CONTRIBUTING.md` / `CHANGELOG.md` / README updates produced by the test are useful and stay.

---
*This document follows the https://specscore.md/plan-specification*
