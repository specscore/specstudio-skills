---
captured_by: user
status: queued
---
# specstudio lint-rules contract is stale against specscore CLI 0.18.0 — F-003/F-004 do not exist and their stated AC semantics invert canonical SpecScore

## Problem

`skills/shared/specscore-lint-rules.md` and `skills/specify/SKILL.md` assert lint rules that **do not exist** and whose semantics **invert** canonical SpecScore. An agent following `specstudio:specify` literally produces a non-canonical Feature the real linter flags.

Verified against specscore CLI 0.18.0 and `specscore/specscore@main`:

- **`F-004`** ("every AC uses Given/When/Then", error) — `lint --rules F-004` → `Unknown rule "F-004"`. Canonical `acceptance-criteria#req:abstract-not-concrete` and its scenario `_tests/ac-contains-given-when-then.md` assert lint **warns** when an AC contains GIVEN/WHEN/THEN. Inverted.
- **`F-003`** ("every REQ has ≥1 AC", error) — `lint --rules F-003` → `Unknown rule "F-003"`. Canonical `acceptance-criteria#req:optional-grouping`: "ACs are OPTIONAL."
- **`U-002`** (requires `**Date:**`, `**Owner:**`) — `specscore feature new` scaffolds `**Status:**` + `**Source Ideas:**` only.
- **Preamble** ("no YAML front-matter") — the CLI scaffold emits `format:`/`status:` front-matter.

`specify/SKILL.md` repeats the AC error in three places: `## Acceptance Criterion Format`, the `## Verification` checklist, and `## Red Flags`.

The contract's escape hatch ("when this contract diverges, that repo wins") only helps a reader who already suspects divergence and reads the reference repo. The default path is to trust the skill and write the wrong artifact.

## Suggested direction

Reconcile both files with canonical SpecScore: ACs are abstract prose bundling ≥2 REQs via `**Requirements:**`; GIVEN/WHEN/THEN belongs to `_tests/` Scenarios with `**Validates:**`. Fix specify's AC-format section, verification checklist, and red flags; correct the `U-002` and front-matter claims.

Longer term, derive the rule table from the CLI's rule registry — or drop rule IDs and point at the canonical spec — rather than hand-maintaining a mirror that rots silently. The fabricated `F-003`/`F-004` IDs show the mirror has no verification pressure on it.

## Provenance

Surfaced 2026-07-15 dogfooding `specstudio:specify` in `sneat-co/sneat-cli` (SpecStudio 0.0.13, CLI 0.18.0). Followed the reference repo instead of the skill; `lint --severity info` passes clean with prose ACs. Full evidence and repro: specscore/specstudio-skills#38.
