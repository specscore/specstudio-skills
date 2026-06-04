# Plan: Seed-to-Idea Promotion

**Status:** Under Review
**Source Feature:** seed-to-idea-promotion
**Date:** 2026-06-04
**Owner:** alexander.trakhimenok
**Supersedes:** —

## Summary

Decomposes the `seed-to-idea-promotion` Feature into six linear tasks: a `specscore idea promote` CLI verb (resolution/refusal, same-repo move+transform, back-link reconcile, cross-repo keep-and-mark, verdict carry-forward) followed by the skill-side flow that invokes the verb and offers consilium for unreviewed seeds. All nine ACs are covered; none deferred.

## Approach

Tasks follow the verb's control flow and the dependency that the same/cross-repo back-link classifier (built in Task 2) underpins both the same-repo reconcile path (Task 3) and the cross-repo keep path (Task 4). The CLI verb is built first (Tasks 1–5) since the skill-side flow (Task 6) delegates to it. Verdict carry-forward (Task 5) is independent of the move/keep branch and lands after the two paths exist. Cross-repo classification relies on the `<repo-slug>:` back-link discriminator defined by the parallel `sidekick-capture` revision.

## Tasks

### Task 1: `promote` command scaffold — seed resolution and collision refusal

**Verifies:** seed-to-idea-promotion#ac:missing-seed-errors, seed-to-idea-promotion#ac:collision-refused-without-force

Scaffold `specscore idea promote <slug>`: resolve the slug to `spec/ideas/seeds/<slug>.md`, exit non-zero naming the missing path when absent, and refuse (exit non-zero, no file touched) when `spec/ideas/<slug>.md` already exists unless `--force` is passed. No move or transform yet.

### Task 2: Same-repo move and transform to a lint-clean Idea skeleton

**Verifies:** seed-to-idea-promotion#ac:same-repo-promotes-by-move

Build the back-link discovery + same/cross-repo classifier (using the `<repo-slug>:` discriminator), and implement the same-repo path: `git mv` the seed, swap frontmatter for Idea body-metadata, retitle to `# Idea:`, fold seed prose into `## Context`, insert HTML-comment prompts for unfilled sections, and run lint-fix so the result is lint-clean with history preserved (`git log --follow`).

### Task 3: Same-repo back-link reconciliation

**Verifies:** seed-to-idea-promotion#ac:same-repo-backlinks-reconciled

Using the classifier from Task 2, rewrite every same-repo `## Sidekick Seeds Generated` back-link entry that pointed at `spec/ideas/seeds/<slug>.md` to point at `spec/ideas/<slug>.md`, leaving no same-repo entry referencing the old seed path. Runs after the move completes.

### Task 4: Cross-repo keep-and-mark path

**Verifies:** seed-to-idea-promotion#ac:cross-repo-keeps-seed, seed-to-idea-promotion#ac:never-marks-deprecated

When the classifier finds any cross-repo back-link, take the no-move branch: create the new Idea by copying and transforming the seed body, leave the seed in place with frontmatter `status: promoted` and a forward pointer to the created Idea. The verb only ever sets `promoted` on a retained seed, never `deprecated`.

### Task 5: Verdict carry-forward

**Verifies:** seed-to-idea-promotion#ac:verdict-pointer-default

Carry the seed's `## Consilium Verdict` forward into the created Idea as a single-line provenance pointer by default, with configuration to instead copy the full section or drop it; omit the pointer when the seed has no verdict.

### Task 6: Skill-side promotion flow and consilium offer

**Verifies:** seed-to-idea-promotion#ac:skill-delegates-to-cli, seed-to-idea-promotion#ac:unreviewed-offer-consilium

Wire `specstudio:ideate` to perform promotion by invoking `specscore idea promote <slug>` (not hand-moving files) and then filling the returned skeleton; before promoting an unreviewed, manually-picked seed (no `## Consilium Verdict`), offer to run the consilium first, proceeding on decline.

## Open Questions

- Task 4's forward-pointer on-disk shape (frontmatter key vs body section) and whether the retained `promoted` seed stays in `spec/ideas/seeds/` or moves to `spec/ideas/archived/` — to be pinned before implementing Task 4 (tracked in the source Feature's Open Questions).
- Task 5's verdict carry-forward configuration surface (`specscore.yaml` block vs per-invocation flag vs both) — to be pinned before implementing Task 5.

---
*This document follows the https://specscore.md/plan-specification*
