# SpecScore Lint Rules Assumed by SDD Skills

**Status:** Contract (skills depend on these; changes require a coordinated update)

Both `specstudio:ideate` and `specstudio:specify` assume `specscore spec lint` enforces the rules below. Skills invoke lint as a verification step — if any rule here changes, the corresponding skill checklist step may need to update.

The rules below are SpecStudio's **expectation** of canonical SpecScore lint behavior — they describe the canonical SpecScore artifact shape (no YAML front-matter; bold-prefixed body metadata; title-prefix dispatch keys). The authoritative spec is the SpecScore feature tree at [`specscore/specscore`](https://github.com/specscore/specscore); when this contract diverges, that repo wins.

## Lint CLI Contract

- `specscore spec lint <file>` — single-file mode. Returns 0 on pass, non-zero on fail; machine-readable output on stderr (`--json` flag).
- `specscore spec lint` (no arg) — whole-tree mode, scans `spec/**`.
- Dispatches per-artifact rule sets based on the **title prefix** (`# Idea: …`, `# Feature: …`, `# Plan: …`). The artifact's canonical id is the filename slug (no separate `id:` field).

## Universal Rules (all SpecScore artifacts)

| ID | Rule | Severity |
|---|---|---|
| U-001 | Title is `# Idea: <name>` / `# Feature: <name>` / `# Plan: <name>` (the prefix is the dispatch key) | error |
| U-002 | Required body-metadata lines present immediately after the title: `**Status:**`, `**Date:**`, `**Owner:**` (in that order) | error |
| U-003 | `**Status:**` value is valid for the artifact kind (see kind-specific rules below) | error |
| U-004 | `**Date:**` value is ISO-8601 and not in the future | warn |
| U-005 | No unresolved placeholders (`TBD`, `TODO`, `???`, `FIXME`) in required sections | error |
| U-006 | No broken cross-references (`**Promotes To:**`, `**Supersedes:**`, `**Source Ideas:**`, `**Related Ideas:**`) | error |
| U-007 | File location matches canonical path for the artifact kind (see [`path-conventions.md`](path-conventions.md)) | error |
| U-008 | Adherence footer present as the last line of the document | error |

## Idea-Specific Rules (`# Idea: …`)

| ID | Rule | Severity |
|---|---|---|
| I-001 | Required sections present: `Problem Statement`, `Recommended Direction`, `MVP Scope`, `Not Doing`, `Key Assumptions to Validate` | error |
| I-002 | `Not Doing` section is non-empty (explicit exclusions required) | error |
| I-003 | `Key Assumptions to Validate` lists at least one Must-be-true assumption | error |
| I-004 | `Problem Statement` contains exactly one "How Might We" framing | warn |
| I-005 | `**Status:**` ∈ {`Draft`, `Under Review`, `Approved`, `Implementing`, `Specified`, `Archived`} | error |
| I-006 | If `**Status:** Specified`, `**Promotes To:**` must be non-empty (and not `—`) | error |
| I-007 | If `**Status:** ∈ {Draft, Under Review, Approved}`, `**Promotes To:**` may be `—` (empty) | info |
| I-008 | `**Promotes To:**`, `**Supersedes:**`, `**Related Ideas:**` lines are present (value `—` when empty) | error |
| I-009 | `**Archive Reason:**` line is present when (and only when) `**Status:** Archived` | error |

## Feature-Specific Rules (`# Feature: …`)

| ID | Rule | Severity |
|---|---|---|
| F-001 | `README.md` present at `spec/features/<slug>/README.md` | error |
| F-002 | `## Behavior` contains at least one `#### REQ: <slug>` requirement | error |
| F-003 | Each requirement has at least one acceptance criterion | error |
| F-004 | Each acceptance criterion uses `Given / When / Then` format | error |
| F-005 | `**Status:**` ∈ {`Draft`, `Under Review`, `Approved`, `Implementing`, `Stable`, `Deprecated`} | error |
| F-006 | If `**Supersedes:**` references another Feature, that Feature exists and has `**Status:** Deprecated` | error |
| F-007 | Required sections present per the SpecScore Feature spec (Summary, Problem, Behavior, Acceptance Criteria, Open Questions) | error |
| oq-section | The canonical questions heading is `## Open Questions`; the legacy `## Outstanding Questions` is rejected and migrated by `specscore spec lint --fix` | error |

## Plan-Specific Rules (`# Plan: …`)

Out of scope for this document — defined by the planning skill (see `spec/ideas/specstudio-plan-skill.md`).

## What Skills Do on Lint Failure

1. **Run `specscore spec lint --fix <file>` exactly once.** Trust the CLI to repair what is safely auto-fixable.
2. **Re-run `specscore spec lint <file>`.** If the result is now clean, continue and tell the user what was auto-fixed.
3. **If lint still fails, surface the remaining violations to the user** with rule IDs and affected sections.
4. **Do not loop `--fix`.** One attempt only.
5. **Do not bypass lint.** Skills treat a passing `specscore spec lint` as a precondition for user-review and event emission.

### CLI owns the fix policy

Skills do NOT carry their own list of which rules are safely auto-fixable. That policy belongs to the `specscore` CLI's `--fix` implementation. If `--fix` silently repairs a violation that should require human input (e.g., inserts a placeholder for a missing `Not Doing` list), that is a CLI bug to file against `specscore` — not a workaround to encode in any skill. This keeps the contract single-sourced and prevents skill-vs-CLI drift.

A future enhancement to the CLI's lint output may include a per-violation `auto_fixable: true/false` metadata field, which skills MAY use to produce richer surfaced messages ("these 2 could be auto-fixed by re-running with `--fix`; these 3 require your input"). Skills MUST NOT depend on this metadata for the basic recovery flow above.

## Versioning

Lint rule IDs are stable. Adding rules is backward-compatible; removing or renaming requires a skill update.
