# Feature: Verify Skill

> [SpecScore.**Studio**](https://specscore.studio): | [Explore](https://specscore.studio/app/github.com/synchestra-io/specstudio-skills/spec/features/skills/verify?op=explore) | [Edit](https://specscore.studio/app/github.com/synchestra-io/specstudio-skills/spec/features/skills/verify?op=edit) | [Ask question](https://specscore.studio/app/github.com/synchestra-io/specstudio-skills/spec/features/skills/verify?op=ask) | [Request change](https://specscore.studio/app/github.com/synchestra-io/specstudio-skills/spec/features/skills/verify?op=request-change) |

**Status:** Approved
**Date:** 2026-05-22
**Owner:** alex
**Source Ideas:** specstudio-verify-skill
**Supersedes:** —

## Summary

The `specstudio:verify` skill consumes an approved SpecScore Feature and produces a machine-checkable per-AC verdict report. For each acceptance criterion in the Feature, the skill walks `git log` for the `Verifies:` commit-message trailer that `specstudio:implement` writes, dispatches a built-in AI subagent with the AC text plus the matching commits and diffs, and aggregates the subagent's verdicts into a Markdown report with a grep-friendly YAML summary block at the top. The skill closes the SpecStudio pipeline's verify slot so downstream skills (`recap`, `review`, `ship`) have an honest gate to consume. Implementation lives at `skills/verify/`.

## Problem

`specstudio:implement` stages AC-traceable diffs and emits `Verifies: <feature-slug>#ac:<ac-slug>` commit trailers on every commit, but nothing in the pipeline closes the loop by *checking* whether those commits actually satisfy the ACs they claim to. Today the check is a manual exercise: a human reads the spec, reads the diff, and decides per AC. That manual step blocks `recap`, `review`, and `ship` from being honest gates — they have no structured per-AC verdict to consume.

The Idea (`spec/ideas/specstudio-verify-skill.md`) names the MVP shape: one built-in AI-subagent verifier, trailer-driven scoping, Markdown report with YAML summary, serial dispatch, no pluggability. This Feature pins that shape into an enforceable contract with concrete REQ/AC.

## Behavior

### Invocation and pre-flight

#### REQ: invocation-triggers

The skill is invoked via the `specstudio:verify <feature-slug>` skill name or as the explicit downstream transition from `specstudio:implement`'s `transition-to-verify` REQ. The skill MUST accept exactly one positional argument — the Feature slug — and resolve it to `spec/features/<feature-slug>/README.md`.

#### REQ: requires-approved-feature

The skill MUST refuse to run when the resolved Feature has `**Status:**` outside the set `{Approved, Implementing, Stable}`. On refusal, the skill MUST print the Feature's current Status and recommend re-approving via `specstudio:specify` before retrying.

#### REQ: requires-feature-in-git-head

The skill MUST confirm the Feature exists at git HEAD via `git cat-file -e HEAD:spec/features/<feature-slug>/README.md`. A Feature that exists only in the working tree (uncommitted) MUST cause the skill to refuse with an instruction to commit the Feature first. This mirrors `implement`'s pre-flight discipline.

### Hard gate

#### REQ: hard-gate

The skill MUST NOT invoke `specstudio:recap`, `specstudio:review`, `specstudio:ship`, `writing-plans`, `frontend-design`, `mcp-builder`, or ANY downstream skill until the report has been written, lint passes on the repo, and the report has been staged. The only permitted transition after a successful run is `specstudio:recap` (or hand back to the user if `recap` is unshipped).

### Reading the Feature

#### REQ: feature-parse

The skill MUST parse the Feature via the `specscore` CLI's Feature parser (delegated; do not re-implement). The parse MUST surface the ordered list of AC IDs in the form `<feature-slug>#ac:<ac-slug>` along with each AC's `Given / When / Then` text. Parse failures MUST stop the skill with the CLI's lint-rule citation surfaced to the user.

### Trailer-driven AC ↔ commit mapping

#### REQ: trailer-grep-per-ac

For each AC ID parsed from the Feature, the skill MUST walk the current branch's git history with `git log --grep='^Verifies:.*<feature-slug>#ac:<ac-slug>'` (extended regexp). The skill MUST collect, per AC: the list of matching commit SHAs in chronological order and their commit messages. The skill MUST NOT pre-fetch full commit diffs — the subagent fetches diffs on demand via its own Bash tool (see `subagent-prompt`).

#### REQ: unmapped-detection

An AC with zero matching commits MUST be recorded as `unmapped`. The skill MUST NOT dispatch a subagent for an unmapped AC and MUST NOT treat `unmapped` as a failure for exit-code purposes (see `exit-code-semantics`). The orchestrator MUST be the sole producer of the `unmapped` verdict; the value `unmapped` MUST NOT appear in the allowed-verdict set named in any subagent prompt (see `subagent-verdict-contract`).

### Subagent dispatch

#### REQ: subagent-dispatch-serial

The skill MUST dispatch one AI subagent at a time, in the Feature's AC order. Parallel dispatch is explicitly out of scope for the MVP (see `## Not Doing`); the skill MUST NOT spawn more than one verifier subagent concurrently. The next AC's subagent is dispatched only after the previous AC's subagent has returned a verdict.

#### REQ: subagent-prompt

The subagent prompt MUST include, in this order: (1) the AC text in full `Given / When / Then` form with its AC ID, (2) the list of matching commit SHAs (chronological order) paired with their commit messages, (3) the verdict contract from `subagent-verdict-contract` (verbatim, so the subagent knows the required output shape), and (4) an explicit instruction to fetch commit diffs and read source files on demand via the subagent's own Bash tool (e.g., `git show <sha>`, `git show <sha> -- <path>`, `cat <path>`). The orchestrator MUST NOT pre-fetch diffs into the prompt; the subagent decides which commits and which files to read at what depth.

#### REQ: subagent-verdict-contract

The subagent MUST return a verdict from the set `{pass, fail, error}`. The verdict line MUST be followed by a justification of at most 400 characters and a list of evidence references (file paths, commit SHAs, optional `_tests/` scenario filenames). Malformed verdicts (verdict outside the allowed set, missing justification, justification exceeding 400 characters) MUST be retried exactly once with a corrective prompt; a second malformed response MUST be recorded as `error` for that AC.

### Report generation

#### REQ: report-path

The skill MUST write the report to `spec/features/<feature-slug>/_verify/<sha>.md` where `<sha>` is the abbreviated git SHA of `HEAD` at run time. The `_verify/` directory MUST be created if absent.

#### REQ: report-yaml-summary

The report MUST begin with a fenced YAML block (delimited by ` ```yaml ` and ` ``` `) listing, in Feature AC order, each AC ID with its verdict and a one-line justification snippet. This block is the grep-target downstream skills consume. Example shape:

```yaml
feature: <feature-slug>
revision: <sha>
verdicts:
  - ac: <feature-slug>#ac:<ac-slug-1>
    verdict: pass
    justification: "<one-line snippet>"
  - ac: <feature-slug>#ac:<ac-slug-2>
    verdict: unmapped
    justification: "no commits reference this AC"
```

#### REQ: report-body

After the YAML block, the report MUST contain one `## AC: <ac-slug>` section per AC, each with the verdict, full justification, commit list, and evidence references. The body is human-readable; the YAML block is machine-readable.

#### REQ: report-staged

After writing the report, the skill MUST stage it with `git add spec/features/<feature-slug>/_verify/<sha>.md`. The skill MUST stage the `_verify/README.md` index from `REQ:report-index-readme` in the same staging set. The skill MUST NOT run `git commit` — committing the report is the user's call (mirrors the `ideate` / `specify` / `plan` / `implement` discipline).

#### REQ: report-index-readme

The skill MUST create `spec/features/<feature-slug>/_verify/README.md` if absent and MUST append a row for the current run to its `## Contents` table on every run. The README is the directory's index — without it, the project's `readme-exists` lint rule fails on the newly created `_verify/` directory. The README MUST follow SpecScore's scenarios-index conventions: an H1, a one-paragraph description, a `## Contents` table with columns `Report | Run revision | Verdict summary`, an `## Open Questions` section (set to `None at this time.` when no questions are tracked), and the `*This document follows the https://specscore.md/index-specification*` footer. New runs append rows newest-first or newest-last consistently — pick one and document the choice in `## Approach` of the Plan, not as a Feature-level REQ.

### Exit semantics

#### REQ: exit-code-semantics

The skill MUST exit non-zero if and only if at least one AC has verdict `fail` or `error`. The verdicts `pass` and `unmapped` MUST NOT contribute to a non-zero exit. The exit-code semantics make `unmapped` an informational state at the verify layer; `ship` is the eventual gate that escalates `unmapped` to blocking.

#### REQ: no-commits-edge-case

When a Feature has zero commits matching ANY of its ACs' trailers (e.g., a Feature that was specified but not yet implemented), the skill MUST still generate a complete report with every AC marked `unmapped`, MUST stage the report, and MUST exit zero. The report itself communicates that nothing has been implemented yet.

### Event emission

#### REQ: verify-completed-event

On a successful run (report written, regardless of verdict outcomes), the skill MUST emit a `verify.completed` event via the convention in `skills/shared/synchestra-events.md`. Payload MUST include the following integer count fields, each ≥ 0, summing to the Feature's total AC count: `passed_count`, `failed_count`, `unmapped_count`, `errored_count`. Payload MUST also include: `feature_slug`, `revision` (the HEAD SHA), and `report_path` (relative to repo root). Additional payload fields MAY be added in the future without breaking this contract; the four count fields and the three identity fields are the minimum. The event MUST be emitted exactly once per successful run. Per-AC verdict details are NOT in the event payload — consumers read them from the report file at `report_path`.

### Transition

#### REQ: transition-to-recap

After the report is written, staged, and the event is emitted, the skill MUST transition only to `specstudio:recap` (or, while `recap` is unshipped, hand back to the user with the report path and a recommendation to review verdicts manually). The skill MUST NOT invoke `ideate`, `specify`, `plan`, `implement`, `review`, `ship`, `frontend-design`, `mcp-builder`, or any other skill on transition.

## Architecture

- **Orchestrator (the skill body):** Pre-flight, Feature parse, AC iteration, subagent dispatch coordination, verdict aggregation, report write, event emission.
- **Verifier subagent (single built-in):** Receives the per-AC prompt, fetches commit diffs and reads source files on demand via its own Bash tool (per `subagent-prompt`), returns the verdict line per `subagent-verdict-contract`.
- **Inputs:** Approved Feature at `spec/features/<feature-slug>/README.md`; git history with `Verifies:` trailers.
- **Outputs:** Markdown report at `spec/features/<feature-slug>/_verify/<sha>.md` (staged, not committed); one `verify.completed` event in `.synchestra/events.jsonl`.
- **Dependencies:** `specstudio:implement` shipped (✅); `Verifies:` trailer convention holds in commit history; `specscore` CLI Feature parser available.

## Interaction with Other Features

| Feature | Relationship |
|---|---|
| [Implement Skill](../implement/README.md) | `implement` is the upstream gate. `implement`'s `Verifies:` trailers are the AC↔commit linkage `verify` consumes. `implement` transitions to `verify` via its `transition-to-verify` REQ; `verify` never invokes `implement`. |
| [Recap Skill](../recap/README.md) | `verify` is the upstream gate of `recap`. `recap` consumes the report's YAML summary block to compare what-was-built vs. what-was-specified. While `recap` is unshipped, `verify` hands back to the user. |
| [Third-Party Integration](../../third-party-integration/README.md) | The `Verifies:` commit-trailer convention is lint-enforceable and is what makes pluggable verifier backends possible *later*. Pluggability is out of scope for this Feature (see `## Not Doing`); the seed in `## Sidekick Seeds Generated` points at the future direction. |

## Acceptance Criteria

ACs are grouped here with explicit REQ back-references, mirroring the sibling `implement` Feature's style.

### AC: refuses-draft-feature (verifies REQ:requires-approved-feature)

**Given** a Feature at `spec/features/<slug>/README.md` with `**Status:** Draft`,
**When** the user runs `specstudio:verify <slug>`,
**Then** the skill MUST refuse to run, print the current Status, recommend `specstudio:specify` to re-approve, MUST NOT write any report, and MUST exit non-zero.

### AC: refuses-uncommitted-feature (verifies REQ:requires-feature-in-git-head)

**Given** a Feature with `**Status:** Approved` that exists in the working tree but has not been committed (`git cat-file -e HEAD:spec/features/<slug>/README.md` exits non-zero),
**When** the user runs `specstudio:verify <slug>`,
**Then** the skill MUST refuse, instruct the user to commit the Feature first, MUST NOT write any report, and MUST exit non-zero.

### AC: per-ac-trailer-grep (verifies REQ:trailer-grep-per-ac, REQ:feature-parse)

**Given** an approved Feature with three ACs `slug#ac:a`, `slug#ac:b`, `slug#ac:c`, where commits in the branch contain `Verifies: slug#ac:a` (one commit) and `Verifies: slug#ac:b` (two commits) but no commit references `slug#ac:c`,
**When** the user runs `specstudio:verify slug`,
**Then** the skill MUST collect one commit for AC `a`, two commits for AC `b`, and zero commits for AC `c`, and MUST dispatch subagents only for ACs `a` and `b`.

### AC: unmapped-not-fail (verifies REQ:unmapped-detection, REQ:exit-code-semantics)

**Given** an approved Feature where AC `slug#ac:c` has zero matching `Verifies:` trailers and all other ACs return `pass` from their subagents,
**When** the user runs `specstudio:verify slug`,
**Then** the report MUST mark AC `c` as `unmapped`, the skill MUST exit zero, and the YAML summary's `verdicts` list MUST include AC `c` with `verdict: unmapped`.

### AC: subagent-serial-dispatch (verifies REQ:subagent-dispatch-serial)

**Given** an approved Feature with four ACs each having at least one matching commit,
**When** the user runs `specstudio:verify slug`,
**Then** the skill MUST dispatch exactly one verifier subagent at a time in AC order, and at no point during the run MUST more than one verifier subagent be concurrently in flight.

### AC: subagent-prompt-shape (verifies REQ:subagent-prompt, REQ:subagent-verdict-contract)

**Given** AC `slug#ac:a` with two matching commits and a `Given/When/Then` body,
**When** the orchestrator dispatches the verifier subagent for AC `a`,
**Then** the dispatched prompt MUST contain the AC's full G/W/T text, both commit SHAs paired with their commit messages, the verbatim verdict contract (allowed values, required justification length, evidence-reference requirement), and an explicit instruction to fetch diffs and read source files on demand via the subagent's own Bash tool. The dispatched prompt MUST NOT contain pre-fetched commit diffs.

### AC: malformed-verdict-retried-once (verifies REQ:subagent-verdict-contract)

**Given** a verifier subagent that returns a malformed verdict on first call (e.g., a verdict outside `{pass, fail, error}` or a missing justification),
**When** the orchestrator parses the response,
**Then** the orchestrator MUST re-dispatch the subagent with a corrective prompt exactly once; if the second response is also malformed, the orchestrator MUST record the AC's verdict as `error` and MUST NOT call the subagent a third time.

### AC: report-path-and-staging (verifies REQ:report-path, REQ:report-staged)

**Given** an approved Feature and a verify run that completes,
**When** the run finishes,
**Then** a Markdown report MUST exist at `spec/features/<feature-slug>/_verify/<sha>.md` where `<sha>` is the abbreviated git SHA of `HEAD`, the file MUST be staged via `git add`, and the skill MUST NOT have invoked `git commit`.

### AC: report-yaml-block-grep-target (verifies REQ:report-yaml-summary, REQ:report-body)

**Given** a completed verify run,
**When** a downstream consumer reads the report file,
**Then** the report's first content MUST be a fenced YAML block (delimited by ` ```yaml ` and ` ``` `) listing every AC with `ac`, `verdict`, and `justification` fields in Feature AC order; below the YAML block, the report MUST contain one `## AC: <ac-slug>` section per AC.

### AC: report-index-readme-created-and-updated (verifies REQ:report-index-readme, REQ:report-staged)

**Given** an approved Feature whose `_verify/` directory does not yet exist,
**When** the user runs `specstudio:verify <slug>` for the first time,
**Then** the skill MUST create `spec/features/<slug>/_verify/README.md` containing an H1, a one-paragraph description, a `## Contents` table with exactly one row referencing the just-written `<sha>.md`, an `## Open Questions` section set to `None at this time.`, and the `*This document follows the https://specscore.md/index-specification*` footer; the README MUST be in the same staged set as the per-run report; and `specscore spec lint` MUST exit zero after the run.

**Given** an approved Feature whose `_verify/README.md` already exists from a prior run with N rows in its `## Contents` table,
**When** the user runs `specstudio:verify <slug>` again,
**Then** the skill MUST append one row to the table for the current `<sha>.md` (preserving the existing rows in their order) and MUST stage the README alongside the new per-run report; the resulting table MUST have N+1 rows; lint MUST exit zero.

### AC: exit-non-zero-on-fail-or-error (verifies REQ:exit-code-semantics)

**Given** an approved Feature where the verifier subagent returns `fail` for at least one AC,
**When** the run completes and the report is written,
**Then** the skill MUST exit non-zero, and the YAML summary MUST mark the failing AC(s) with `verdict: fail`.

**Given** an approved Feature where at least one AC ends with verdict `error` (e.g., because the subagent returned malformed responses on both attempts per `REQ:subagent-verdict-contract`),
**When** the run completes and the report is written,
**Then** the skill MUST exit non-zero, and the YAML summary MUST mark the affected AC(s) with `verdict: error`.

### AC: no-commits-still-reports (verifies REQ:no-commits-edge-case)

**Given** an approved Feature whose ACs have zero matching `Verifies:` trailers in the entire branch history,
**When** the user runs `specstudio:verify <slug>`,
**Then** the skill MUST write a complete report at the canonical path with every AC marked `unmapped`, MUST stage the report, MUST emit the `verify.completed` event with `unmapped_count` equal to the AC count and the other three counts equal to zero, and MUST exit zero.

### AC: verify-completed-event-emitted (verifies REQ:verify-completed-event)

**Given** a verify run that completes (report written, regardless of verdict outcomes) on a Feature with N total ACs,
**When** the orchestrator finishes the report write,
**Then** exactly one `verify.completed` event MUST be appended to `.synchestra/events.jsonl` (or emitted via `synchestra emit` when the CLI is available), with payload fields `feature_slug`, `revision`, `report_path`, `passed_count`, `failed_count`, `unmapped_count`, and `errored_count`,
**And** each of the four count fields MUST be a non-negative integer,
**And** `passed_count + failed_count + unmapped_count + errored_count` MUST equal N,
**And** the payload MUST NOT include per-AC verdict details.

### AC: transition-to-recap-only (verifies REQ:transition-to-recap, REQ:hard-gate)

**Given** a successful verify run,
**When** the skill prepares to transition,
**Then** the skill MUST offer transition only to `specstudio:recap` (or, when `recap` is unshipped, hand back to the user with the report path), and MUST NOT invoke `ideate`, `specify`, `plan`, `implement`, `review`, `ship`, `writing-plans`, `frontend-design`, or `mcp-builder`.

## Rehearse Integration

Per the rehearse-heuristic, every AC above is testable via filesystem fixtures (mock repo with seeded git history + Feature files) and event-payload inspection. The non-testable cases (e.g., the verifier subagent's judgment quality) are validated at the assumption-validation layer of the source Idea, not as Rehearse scenarios.

Rehearse stubs for each AC are scaffolded at `_tests/<ac-slug>.md` with `**Status:** pending`; authoring scenario steps follows the implementation plan.

## Not Doing

Inherited from the source Idea and pinned here:

- **Pluggable verifier backends** — deferred to a dedicated future Idea once a real second backend (pytest adapter, Rehearse runtime, superpowers TDD) is on the table. Designing the contract with N=1 implementation overfits to the AI-subagent shape.
- **`specscore.yaml verify:` block** — not added in MVP; would prematurely commit to a backend-naming schema.
- **JSONL stdin/stdout contract in shared/** — same overfit risk as above.
- **Rehearse executor** — there is no Rehearse runtime in this repo today; `_tests/` Markdown scenarios are read as evidence text by the subagent, not executed.
- **Whole-repo input scoping** — trailer-driven scoping is tighter and rewards the discipline `implement` enforces. Hand-edits without `Verifies:` trailers are invisible to verify — which is the correct behavior.
- **Single-AC isolated runs** — MVP runs the full Feature; per-AC mode lands in a follow-on Idea if a real workflow needs it.
- **Parallel subagent dispatch** — MVP is serial. Parallelism lands in a follow-on Idea once typical AC counts and token-burst behavior are observed.
- **Drift detection against `implement`'s claimed ACs** — useful (a commit's `Verifies:` trailer could claim ACs the verifier judges `fail`) but separable; lands when pluggability lands.
- **Multi-branch verification** — verify always operates on the current branch's `HEAD`. Cross-branch verification is out of scope.

## Open Questions

- The role of `_tests/` Markdown scenarios in verification (ground truth, hints, or out-of-scope) is deliberately unresolved here. A separate Idea (captured as a sidekick seed during this Feature's specify session) will define how the subagent treats `_tests/` files. Until that Idea ships, `_tests/` files are not part of the subagent prompt and the subagent has no special instruction to read them.

## Sidekick Seeds Generated

- [future-review-skill-could-discover-available-claude-code](../../../ideas/seeds/future-review-skill-could-discover-available-claude-code.md) — captured 2026-05-22 by specstudio:specify
- [define-the-role-of-tests-markdown-scenarios-in-the-verify](../../../ideas/seeds/define-the-role-of-tests-markdown-scenarios-in-the-verify.md) — captured 2026-05-22 by specstudio:specify

---
*This document follows the https://specscore.md/feature-specification*
