# Idea: Seed-to-Idea Promotion

**Status:** Specifying
**Date:** 2026-06-04
**Owner:** alexander.trakhimenok
**Promotes To:** seed-to-idea-promotion
**Supersedes:** —
**Related Ideas:** —

## Problem Statement

How might we promote a reviewed seed into a lint-clean Idea while preserving its provenance and not leaving redundant artifacts behind?

## Context

Surfaced via /ideate: when a sidekick seed (spec/ideas/seeds/<slug>.md) becomes a full Idea (spec/ideas/<slug>.md), should we git mv the seed into place, or deprecate the seed and create a new Idea? Today the consilium explicitly defers this ('No auto-promotion at this layer (Phase 2)'). Seeds and Ideas are different schemas (YAML-frontmatter seed vs bold-prefix-metadata Idea), so a pure move never yields a lint-clean Idea — content must be transformed either way. The real choice is a provenance/bloat strategy, complicated by sidekick's back-links and consilium's verdict, both of which live on the seed.

## Recommended Direction

Treat promotion as transform-then-relocate, not a pure move and not deprecate-and-recreate. DEFAULT (same-repo): git mv spec/ideas/seeds/<slug>.md -> spec/ideas/<slug>.md, rewrite the body into the Idea schema in the same commit (git rename-detection keeps the raw seed in history — provenance with zero standalone artifact), and reconcile the '## Sidekick Seeds Generated' back-links in source artifacts to point at the new Idea path. FALLBACK (cross-repo): when any back-link originates in another repo, a single-repo git mv cannot fix it, so the seed STAYS in place and is marked status: promoted with a forward pointer to the new Idea. Terminal seed states are distinct and never conflated: 'promoted' (became an Idea — the implemented analogue) vs 'deprecated' (consilium blocked it). The promoted Idea carries the consilium verdict forward via a one-line provenance pointer by DEFAULT, configurable to copy the full verdict section or drop it.

Consilium review is the normal *triage-time* path (the consilium draining the seed queue), and promotion does NOT hard-require a prior verdict. But a manually-picked seed is effectively pulling from the queue out of band, so when a user promotes an *unreviewed* seed, promotion first OFFERS to run the consilium before transforming it into an Idea — an offer the user can decline. Because `sidekick` writes cross-repo back-links (in a repo-qualified, machine-distinguishable form), cross-repo references are first-class and detectable at promotion time — which is precisely why a cross-repo seed must stay in place rather than move.

## Alternatives Considered

- **Pure `git mv`, no transform.** Tempting because it's one command and preserves history, but it loses: a seed's YAML frontmatter + free prose is not a lint-clean Idea (`# Idea:` heading, bold-prefix metadata, nine required sections, I-002 Not Doing). The result fails idea lint, so a transform is unavoidable regardless of how the file is relocated.
- **Always deprecate seed + create new Idea (two files).** This was the status-quo assumption in the request. Rejected as the *default* because it bloats `spec/`, which is exactly the concern raised — for same-repo promotions, git rename-detection already preserves the raw seed in history, so a second standalone artifact earns nothing. It survives only as the cross-repo fallback (renamed to `promoted`, since the seed was consumed, not blocked).
- **Always move, accept broken back-links, repair later via `lint --fix`.** Rejected as the default: it ships known-dangling links and bets on a reconciliation rule that doesn't exist yet. Back-link reconciliation at promotion time is cheap and bounded, so do it then.

## MVP Scope

Two-week spike on the same-repo path only: a promotion operation (specscore idea promote <seed-slug>, or ideate-driven) that git-mv's the seed, scaffolds the Idea schema carrying the seed's body, rewrites same-repo back-links, and stamps a verdict provenance line. Cross-repo guard: if any back-link is cross-repo, refuse the move and fall back to keep + status: promoted. Good enough if it promotes one real same-repo seed end-to-end, lint-clean, with back-links intact and the seed recoverable via git log --follow.

## Not Doing (and Why)

- Cross-repo back-link rewriting at promotion time — a single-repo git op cannot mutate sibling repos; keep + promoted instead (note: sidekick *writing* cross-repo back-links at capture time is in scope, as a dependency)
- Auto-promotion from a consilium verdict — promotion stays user-initiated (consilium already states 'no auto-promotion at this layer')
- A lint --fix rule that retro-reconciles already-broken back-links — separate, deferred concern
- Reverse demotion (Idea back to seed) — not a real workflow

## Key Assumptions to Validate

| Tier | Assumption | How to validate |
|------|------------|-----------------|
| Must-be-true | Git rename-detection preserves the raw seed across a transform commit, so "git history only" is real provenance. | `git mv` a seed then heavily rewrite the body in the same commit; confirm `git log --follow` / `git blame` recovers the original seed text. |
| Must-be-true | The `## Sidekick Seeds Generated` back-link bullets have a stable, machine-rewritable format and can be located in source artifacts. | Parse the section across existing Features/Plans that spawned seeds; confirm the bullet/relative-path shape is reliably matchable. |
| Should-be-true | Cross-repo back-links are written by sidekick in a repo-qualified form that promotion can classify by origin repo. | Define the cross-repo back-link format in `sidekick-capture`; confirm promotion can split same-repo (reconcile) from cross-repo (keep) reliably. |
| Might-be-true | Users want verdict carry-forward configurable rather than always-on (pointer). | Defaults usage feedback once promotion ships; revisit if nobody changes the default. |


## SpecScore Integration

- **New Features this would create:** a seed-promotion capability, likely split between a `specscore` CLI verb (e.g. `idea promote <seed-slug>`) that owns the git mv + schema scaffold + back-link rewrite, and skill wiring (ideate/consilium) that invokes it.
- **Existing Features affected:** `sidekick-capture` (the `## Sidekick Seeds Generated` back-link format becomes a load-bearing contract, AND it must be extended to write cross-repo back-links in a repo-qualified form); `sidekick-consilium` (this is its deferred "Phase 2 promotion"; defines verdict carry-forward, and the promotion-offers-consilium handshake for unreviewed manually-picked seeds); `relocate-idea` (shares move + link-reconcile mechanics — possible code reuse); `ideate` (gains "promote an existing seed" as an entry path).
- **Dependencies:** a consilium review typically precedes promotion (open question below); a `specscore` CLI version that ships the promote verb.

## Open Questions

- What repo-qualified format should `sidekick` write for cross-repo back-links (e.g. `<repo-slug>:spec/ideas/seeds/<slug>.md`) so promotion can classify each back-link's origin repo? (Decision taken: sidekick *will* write them; format is open.)
- The promotion-offers-consilium handshake for an unreviewed, manually-picked seed: offer by default — but is the offer suppressible via config, and what is the default answer if the user just hits enter?
- What is the config surface for verdict carry-forward — a `specscore.yaml` block, a per-invocation flag, or both?
- In the cross-repo keep case, does the retained `promoted` seed stay in `spec/ideas/seeds/` or move to `spec/ideas/archived/`?

---
*This document follows the https://specscore.md/idea-specification*
