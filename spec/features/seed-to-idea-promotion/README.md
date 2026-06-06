---
format: https://specscore.md/feature-specification
status: Approved
---

# Feature: Seed-to-Idea Promotion

> [SpecScore.**Studio**](https://specscore.studio): | [Explore](https://specscore.studio/app/github.com/specscore/specstudio-skills/spec/features/seed-to-idea-promotion?op=explore) | [Edit](https://specscore.studio/app/github.com/specscore/specstudio-skills/spec/features/seed-to-idea-promotion?op=edit) | [Ask question](https://specscore.studio/app/github.com/specscore/specstudio-skills/spec/features/seed-to-idea-promotion?op=ask) | [Request change](https://specscore.studio/app/github.com/specscore/specstudio-skills/spec/features/seed-to-idea-promotion?op=request-change) |

**Status:** Approved
**Date:** 2026-06-04
**Owner:** alexander.trakhimenok
**Source Ideas:** seed-to-idea-promotion
**Supersedes:** —
**Grade:** B

## Summary

A deterministic `specscore idea promote <slug>` CLI verb that turns a sidekick seed (`spec/ideas/seeds/<slug>.md`) into a lint-clean Idea skeleton (`spec/ideas/<slug>.md`), plus a skill-side flow that offers consilium review for an unreviewed, manually-picked seed before promoting. Serves SpecStudio maintainers who triage the seed queue and want a seed to become an Idea without redundant artifacts or dangling back-links.

## Problem

Today a seed and an Idea are different schemas (YAML-frontmatter seed vs bold-prefix-metadata Idea), and there is no defined path from one to the other — the consilium explicitly defers promotion to "Phase 2." A naive `git mv` produces a file that fails idea lint, and moving a seed silently breaks the `## Sidekick Seeds Generated` back-links that `sidekick-capture` writes into source artifacts. The consilium verdict and queue state also live on the seed, so an undisciplined promotion orphans them. This Feature defines the single, deterministic promotion contract so a reviewed (or deliberately un-reviewed) seed becomes an Idea with provenance preserved and links intact.

## Behavior

The Feature ships two surfaces: a deterministic CLI verb that owns the file mutation, and a skill-side handshake that adds the interactive consilium offer. The CLI verb is non-interactive and idempotent-by-refusal; the skill layers the human-in-the-loop pieces on top.

### The `specscore idea promote` CLI verb

Narrative: `promote` is a deterministic mutation in the family of `idea new` and `idea relocate` — it scaffolds and relocates, it does not author prose. Its output is a lint-clean Idea *skeleton* carrying the seed's content, ready for `specstudio:ideate` to fill.

#### REQ: promote-resolves-seed

The verb MUST accept a bare seed slug and resolve it to `spec/ideas/seeds/<slug>.md` in the current repo. When no such seed exists, the verb MUST exit non-zero with a message naming the missing path and MUST NOT create, move, or modify any file.

#### REQ: promote-collision-refusal

When `spec/ideas/<slug>.md` already exists, the verb MUST refuse and exit non-zero unless `--force` is supplied, mirroring `idea new --force`. On refusal, no file is created, moved, or modified.

#### REQ: same-repo-move-and-transform

When every back-link to the seed is same-repo (or the seed has no back-links), the verb MUST `git mv spec/ideas/seeds/<slug>.md spec/ideas/<slug>.md` (preserving history via rename detection), then transform the moved file in place: replace the seed's YAML frontmatter with Idea body-metadata (`**Status:** Draft`, `**Date:**`, `**Owner:**`, `**Promotes To:** —`, `**Supersedes:** —`, `**Related Ideas:** —`), retitle the body to `# Idea: <title>`, fold the seed's existing prose into `## Context`, and insert HTML-comment prompts for every unfilled canonical Idea section. The verb MUST run lint-fix so the resulting file is lint-clean by construction.

#### REQ: same-repo-back-link-reconcile

In the same-repo path, the verb MUST rewrite every `## Sidekick Seeds Generated` back-link entry in same-repo source artifacts that pointed at `spec/ideas/seeds/<slug>.md` so it points at the new `spec/ideas/<slug>.md` path. No same-repo back-link may dangle after a successful promotion.

#### REQ: cross-repo-keep-and-mark

When any back-link to the seed originates in a different repo, the verb MUST NOT rewrite the sibling repo's back-link (a single-repo git operation cannot reach another repo). It MUST: create the new Idea at `spec/ideas/<slug>.md` by copying and transforming the seed body (same transform as `same-repo-move-and-transform`); `git mv` the seed to `spec/ideas/archived/<slug>.md` with its frontmatter `status` set to `promoted`; and write the forward pointer as a frontmatter key `promoted_to: <slug>` on the archived seed identifying the created Idea. The archived seed and the new Idea both exist after this path. Reconciling the cross-repo back-link to the moved seed is delegated to the lint/UI cross-repo reference resolution (see `## Open Questions`), not to this verb.

#### REQ: promoted-vs-deprecated-distinct

A consumed seed has two distinct terminal states that MUST NOT be conflated: `promoted` (the seed became an Idea — set by this verb) and `deprecated` (the seed was blocked — owned by the consilium, out of scope here). This verb only ever sets `promoted`; it MUST NOT mark a seed `deprecated`.

#### REQ: verdict-carry-forward

By default, the created Idea MUST carry the seed's consilium verdict forward as a single-line provenance pointer. The behavior MUST be selectable with both a project default and a per-invocation override: a `specscore.yaml` `promote.verdict_carry_forward` key (`pointer` | `full` | `drop`, default `pointer`) sets the project default, and a `--verdict=<pointer|full|drop>` flag overrides it for a single run (the flag wins when both are set). `full` copies the entire `## Consilium Verdict` section into the Idea; `drop` omits it. When the seed has no verdict, the verb omits the pointer regardless of the setting and proceeds normally.

### Skill-side promotion flow

Narrative: the CLI verb is non-interactive, so the human-in-the-loop consilium offer and the post-promote authoring live in the invoking skill (`specstudio:ideate`).

#### REQ: offer-consilium-on-unreviewed

When a user manually promotes a seed that has no `## Consilium Verdict` section, the skill MUST offer to run the consilium before promoting, defaulting to "yes" on an empty user response (offer-and-default-to-yes). The user MAY decline; on decline, promotion proceeds. The offer MUST be suppressible via configuration — a `specscore.yaml` `promote.offer_consilium: false` skips the offer entirely and promotes directly without asking. The skill MUST NOT hard-require a verdict and MUST NOT block promotion when the user declines.

#### REQ: skill-invokes-cli-then-fills

The skill MUST perform promotion by invoking `specscore idea promote <slug>` (not by hand-moving files), and then fill the returned lint-clean skeleton via its normal Idea-authoring flow. The skill MUST NOT bypass the CLI verb's transform and reconcile guarantees.

## Acceptance Criteria

### AC: missing-seed-errors (verifies REQ:promote-resolves-seed)

**Given** no file at `spec/ideas/seeds/ghost.md`
**When** `specscore idea promote ghost` runs
**Then** the command exits non-zero, names the missing `spec/ideas/seeds/ghost.md` path, and no file is created, moved, or modified.

### AC: collision-refused-without-force (verifies REQ:promote-collision-refusal)

**Given** a seed at `spec/ideas/seeds/foo.md` and an existing `spec/ideas/foo.md`
**When** `specscore idea promote foo` runs without `--force`
**Then** the command exits non-zero and leaves both files unchanged.

### AC: same-repo-promotes-by-move (verifies REQ:same-repo-move-and-transform)

**Given** a seed at `spec/ideas/seeds/bar.md` whose only back-links are same-repo
**When** `specscore idea promote bar` runs
**Then** `spec/ideas/seeds/bar.md` no longer exists, `spec/ideas/bar.md` exists with a `# Idea:` title and Idea body-metadata, `git log --follow spec/ideas/bar.md` reaches the original seed commit, and `specscore spec lint` passes.

### AC: same-repo-backlinks-reconciled (verifies REQ:same-repo-back-link-reconcile)

**Given** a same-repo Feature whose `## Sidekick Seeds Generated` section links to `spec/ideas/seeds/bar.md`
**When** `specscore idea promote bar` completes via the same-repo path
**Then** that back-link points at `spec/ideas/bar.md` and no entry still references `spec/ideas/seeds/bar.md`.

### AC: cross-repo-keeps-seed (verifies REQ:cross-repo-keep-and-mark)

**Given** a seed at `spec/ideas/seeds/baz.md` with at least one cross-repo back-link
**When** `specscore idea promote baz` runs
**Then** `spec/ideas/baz.md` exists as a lint-clean Idea skeleton, the seed has moved to `spec/ideas/archived/baz.md` with frontmatter `status: promoted` and `promoted_to: baz`, no file remains at `spec/ideas/seeds/baz.md`, and the verb did not rewrite the sibling repo's back-link.

### AC: never-marks-deprecated (verifies REQ:promoted-vs-deprecated-distinct)

**Given** any seed being promoted
**When** the verb sets a terminal state on a retained seed
**Then** the state is `promoted` and never `deprecated`.

### AC: verdict-pointer-default (verifies REQ:verdict-carry-forward)

**Given** a seed carrying a `## Consilium Verdict` section and default configuration
**When** the seed is promoted
**Then** the created Idea contains a single-line provenance pointer to that verdict; setting `promote.verdict_carry_forward: full` (or `--verdict=full`) instead copies the full section, `drop` omits it, and the `--verdict` flag overrides the `specscore.yaml` default when both are present.

### AC: unreviewed-offer-consilium (verifies REQ:offer-consilium-on-unreviewed)

**Given** a user manually promoting a seed with no `## Consilium Verdict`
**When** the promotion flow starts
**Then** the skill offers to run the consilium first, defaulting to yes on an empty response; on the user declining, promotion proceeds without a verdict; and when `promote.offer_consilium` is `false`, no offer is shown and promotion proceeds directly.

### AC: skill-delegates-to-cli (verifies REQ:skill-invokes-cli-then-fills)

**Given** the skill is asked to promote a seed
**When** it performs the promotion
**Then** it invokes `specscore idea promote <slug>` for the move/transform/reconcile and only afterward fills the skeleton, rather than hand-moving files.

## Rehearse Integration

No `_tests/` stubs are scaffolded in this repo. The CLI ACs (`missing-seed-errors`, `collision-refused-without-force`, `same-repo-promotes-by-move`, `same-repo-backlinks-reconciled`, `cross-repo-keeps-seed`, `never-marks-deprecated`, `verdict-pointer-default`) exercise the `specscore idea promote` verb, which is implemented and rehearsed in the `specscore-cli` repo — that is where the testable surface lives. The two skill ACs (`unreviewed-offer-consilium`, `skill-delegates-to-cli`) are conversational/dispatch behaviors of `specstudio:ideate` whose observer is the skill flow itself; they are verified through the skill's own checks rather than a Rehearse scenario file. Revisit if a CLI shim for `promote` is added to this repo.

## Open Questions

- **Cross-repo back-link reconciliation (dependency).** After the seed moves to `spec/ideas/archived/<slug>.md`, the cross-repo back-link in the sibling repo still points at the old `spec/ideas/seeds/<slug>.md`. Promotion cannot reach that repo, so reconciliation is delegated to lint/UI cross-repo reference resolution. That mechanism is a separate, relied-upon capability (cf. the Idea's `Not Doing` on retro-reconcile rules): confirm it resolves a moved/archived seed reference and decide what it ultimately points at (the archived seed vs the promoted Idea).
- Resolved and folded into the REQs above: cross-repo back-link format (`<repo-slug>:` prefix, defined in `sidekick-capture`); verdict carry-forward surface (config default + `--verdict` flag, flag wins); retained-seed location (`spec/ideas/archived/`); consilium-offer suppression (`promote.offer_consilium`) and empty-response default (yes).

---
*This document follows the https://specscore.md/feature-specification*
