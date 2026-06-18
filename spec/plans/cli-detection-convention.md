# Plan: CLI-Required Artifact Creation Rollout

**Status:** Implemented
**Source Feature:** cli-detection-convention
**Date:** 2026-06-05
**Owner:** alexander.trakhimenok
**Supersedes:** —

## Summary

Roll out the CLI-required artifact-creation mandate added to the `cli-detection-convention` Feature: revise the shared detection reference to add the creation class, then convert each producer skill to create artifacts only via `specscore … new` (no embedded schema, install/upgrade-then-retry, CLI as a black box). The detection ACs already satisfied by the prior cycle are deferred with that reason.

## Approach

Linear order: the shared `cli-detection.md` revision lands first because every producer conversion cites its new creation row. The three producers whose scaffold verb already exists — `ideate` (`idea new`), `specify` (`feature new`), `init` (`init`) — convert next and carry the generic creation ACs between them. `sidekick` and `plan` come last as blocked tasks: their scaffold verbs do not yet exist in the `specscore` CLI (`plan new` is in progress separately; the seed-scaffold verb is tracked by a sidekick seed in the `specscore-cli` repo). The six detection ACs satisfied during the original cli-detection-convention cycle are deferred rather than re-implemented.

## Tasks

### Task 1: Revise `skills/shared/cli-detection.md` for the creation class

**Verifies:** cli-detection-convention#ac:doc-exists, cli-detection-convention#ac:four-outcomes

Add a **creation** row to the response table (on `127` → install-then-retry; on exit `8` → upgrade-then-retry; any other non-zero → surface; never a direct-write fallback) and update the documented class set to creation / mandatory / capability-gated / wizard, retiring optional-with-fallback for the creation path. Preserve the existing four-outcome detection mechanism unchanged.

### Task 2: Convert `skills/ideate/SKILL.md` to CLI-required creation

**Verifies:** cli-detection-convention#ac:creation-uses-scaffold, cli-detection-convention#ac:no-schema-block, cli-detection-convention#ac:no-command-v, cli-detection-convention#ac:spec-url-not-fetched, cli-detection-convention#ac:producers-cite

Remove the embedded authoritative schema block and the Step 3c direct-write fallback; require `specscore idea new`; on `127` emit the install message and offer install-then-retry, on exit `8` upgrade-then-retry, other non-zero surface; add the read-only `https://specscore.md/idea-specification` pointer; cite the `cli-detection.md` creation row. This is the reference conversion that establishes the generic creation behavior.

### Task 3: Convert `skills/specify/SKILL.md` to CLI-required creation

**Verifies:** cli-detection-convention#ac:creation-127-install-retry, cli-detection-convention#ac:creation-nonzero-no-fallback, cli-detection-convention#ac:producers-cite

Apply the same creation policy to `specify` using `specscore feature new` (verb confirmed present): delete the embedded Feature schema block and direct-write fallback, wire install/upgrade-then-retry and the surface-on-other-error branch, and cite `cli-detection.md`.

### Task 4: Convert `skills/init/SKILL.md` to CLI-required creation

**Verifies:** cli-detection-convention#ac:cli-blackbox-stated, cli-detection-convention#ac:producers-cite

Convert `init` to scaffold the project via `specscore init` only — remove the AI-agent bootstrap fallback, wire install-then-retry, state that the CLI is treated as a black box (no template-sourcing assumptions), and keep any specification-page pointer read-only. Validate the cold-start path (brand-new repo, no CLI) is acceptable under install-then-retry.

### Task 5: Convert `skills/sidekick/SKILL.md` to CLI-required creation

**Verifies:** cli-detection-convention#ac:producers-cite

**Notes:** Implemented — `specscore sidekick new` shipped in CLI v0.7.2, and the `--slug` override (needed for the skill's `-N` collision disambiguation) shipped in v0.7.3. `skills/sidekick/SKILL.md` converted to required-CLI creation: it now derives the slug, resolves the `-N`-free slug, and delegates the write to `specscore sidekick new "<one-liner>" --slug <final-slug> --project <dest> …`, keeping destination resolution, back-link, and event emission as skill orchestration.

Replace the hand-written seed assembly with the CLI seed-scaffold call, remove the embedded seed template, wire install/upgrade-then-retry, and cite `cli-detection.md`.

### Task 6: Convert `skills/plan/SKILL.md` to CLI-required creation

**Verifies:** cli-detection-convention#ac:producers-cite

**Notes:** Implemented — `specscore plan new` shipped in `specscore` CLI v0.7.0 (`--feature`/`--idea` source flags, gallery template with embedded fallback). `skills/plan/SKILL.md` converted to required-CLI creation.

Replace the embedded Plan schema and direct write with `specscore plan new`, wire install/upgrade-then-retry and the surface-on-other-error branch, and cite `cli-detection.md`.

## Deferred AC Coverage

- cli-detection-convention#ac:mandatory-absent-install — Already satisfied by the prior cli-detection-convention detection cycle (`relocate-idea` converted at commit a0102db); no new work — re-confirmed at verify time.
- cli-detection-convention#ac:capability-upgrade — Already satisfied by the prior cycle (`consilium` converted at commit a0102db); no new work — re-confirmed at verify time.
- cli-detection-convention#ac:capability-precheck-order — Already satisfied by the prior cycle (`consilium` pre-check ordering); no new work — re-confirmed at verify time.
- cli-detection-convention#ac:relocate-cites — Already satisfied by the prior cycle (`relocate-idea` cites `cli-detection.md`); no new work — re-confirmed at verify time.
- cli-detection-convention#ac:consilium-cites — Already satisfied by the prior cycle (`consilium` cites `cli-detection.md`); no new work — re-confirmed at verify time.

## Open Questions

- Task 6 (`plan`) was implemented once `specscore plan new` shipped in CLI v0.7.0. Task 5 (`sidekick`) was implemented once `specscore sidekick new` (v0.7.2) and its `--slug` override (v0.7.3) shipped. All tasks in this Plan are now complete.

---
*This document follows the https://specscore.md/plan-specification*
