---
type: sidekick-seed
slug: implement-preflight-should-require-source-feature-in-git-h
captured_at: 2026-05-19T19:01:10Z
captured_by: user
captured_during: spec/features/skills/implement
trigger: explicit
status: completed
synchestra_task: null
---

# implement pre-flight should require the Plan's Source Feature to exist in git HEAD, not just in the working tree

## Observed problem (dogfood finding #1)

During the implement dogfood, the synthetic `dogfood-test` Feature was created but never committed before `implement` ran. Pre-flight read the Feature from the working tree, passed all validity checks, and dispatched subagents. The resulting batch 1 commit would have carried `Verifies: dogfood-test#ac:...` trailers pointing at a Feature absent from git history — except I caught the gap inline and split into two commits (a354482 for fixture setup, 52c4a80 for batch output).

## Root cause

`REQ:requires-approved-source-feature` says the Source Feature must be `**Status:** ∈ {Approved, Implementing, Stable}` — but says nothing about it being committed to git. Implement's parse step reads the working tree directly.

## Suggested fix

Tighten the REQ to also require the Source Feature exists at HEAD (`git cat-file -e HEAD:spec/features/<feature-slug>/README.md` succeeds). Add a pre-flight check that refuses to dispatch if the Feature exists only in the working tree.

Add an AC: `Given a Plan whose Source Feature exists only in the working tree (uncommitted), When implement runs pre-flight, Then the skill refuses and instructs the user to commit it first.`

## Why this matters

The `Verifies:` trailer is the spec↔code coherence guarantee. If the Feature it references doesn't exist at HEAD, downstream verify/recap/review can't resolve ACs, and P-002 will flag stale references. Surfacing at pre-flight prevents the silent break.

Captured during the implement dogfood; see commits a354482 + 52c4a80.
