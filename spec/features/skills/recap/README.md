# Feature: Recap Skill

> [SpecScore.**Studio**](https://specscore.studio): | [Explore](https://specscore.studio/app/github.com/synchestra-io/specstudio-skills/spec/features/skills/recap?op=explore) | [Edit](https://specscore.studio/app/github.com/synchestra-io/specstudio-skills/spec/features/skills/recap?op=edit) | [Ask question](https://specscore.studio/app/github.com/synchestra-io/specstudio-skills/spec/features/skills/recap?op=ask) | [Request change](https://specscore.studio/app/github.com/synchestra-io/specstudio-skills/spec/features/skills/recap?op=request-change) |

**Status:** Approved
**Date:** 2026-05-22
**Owner:** alex
**Source Ideas:** specstudio-recap-skill
**Supersedes:** —

## Summary

The `specstudio:recap` skill consumes an approved SpecScore Feature plus the latest `specstudio:verify` report at HEAD and produces a per-AC drift report: for each acceptance criterion, it dispatches a built-in AI subagent that compares what the spec asks for against what the commits actually delivered, and classifies the divergence as `no-drift`, `spec-tighter-than-code`, `code-tighter-than-spec`, or `contradiction`. The aggregated report lands at `spec/features/<feature-slug>/_recap/<sha>.md` with a grep-friendly YAML summary block at the top, mirroring verify's layout. The skill closes the pipeline's recap slot so `review` and `ship` have an honest drift gate to consume. Implementation lives at `skills/recap/`.

## Problem

`specstudio:verify` answers "does the code satisfy each AC?" but does not answer "does the code as built match what the spec said it should?" An AC can return `pass` while the implementation took a different route than the AC named — that drift goes to review unflagged. Conversely, an implementation may enforce constraints the spec never named, or fail to enforce constraints the spec did name — both shapes of divergence either waste reviewer attention or get silently absorbed into the Feature without an explicit Proposal.

The Idea (`spec/ideas/specstudio-recap-skill.md`) pins the MVP shape: AC-level drift only, one built-in AI-subagent narrator, baseline = Feature at HEAD, flag-only disposition (never edit the Feature), explicit invocation. This Feature pins that shape into an enforceable contract with concrete REQ/AC.

## Behavior

### Invocation and pre-flight

#### REQ: invocation-triggers

The skill is invoked via the `specstudio:recap <feature-slug>` skill name or as the explicit downstream transition from `specstudio:verify`'s `transition-to-recap` REQ. The skill MUST accept exactly one positional argument — the Feature slug — and resolve it to `spec/features/<feature-slug>/README.md`.

#### REQ: requires-approved-feature

The skill MUST refuse to run when the resolved Feature has `**Status:**` outside the set `{Approved, Implementing, Stable}`. On refusal, the skill MUST print the Feature's current Status and recommend re-approving via `specstudio:specify` before retrying.

#### REQ: requires-feature-in-git-head

The skill MUST confirm the Feature exists at git HEAD via `git cat-file -e HEAD:spec/features/<feature-slug>/README.md`. A Feature that exists only in the working tree (uncommitted) MUST cause the skill to refuse with an instruction to commit the Feature first. This mirrors `verify`'s and `implement`'s pre-flight discipline.

#### REQ: requires-verify-report

The skill MUST refuse to run when `spec/features/<feature-slug>/_verify/` does not exist or contains zero `<sha>.md` report files reachable at HEAD. On refusal, the skill MUST print a recommendation to run `specstudio:verify <feature-slug>` first. Recap-without-verify is a category error: there is nothing to recap against.

### Hard gate

#### REQ: hard-gate

The skill MUST NOT invoke `specstudio:review`, `specstudio:ship`, `writing-plans`, `frontend-design`, `mcp-builder`, or ANY downstream skill until the report has been written, lint passes on the repo, and the report has been staged. The only permitted transition after a successful run is `specstudio:review` (or hand back to the user if `review` is unshipped).

### Reading inputs

#### REQ: feature-parse

The skill MUST parse the Feature via the `specscore` CLI's Feature parser (delegated; do not re-implement). The parse MUST surface the ordered list of AC IDs in the form `<feature-slug>#ac:<ac-slug>` along with each AC's `Given / When / Then` text. Parse failures MUST stop the skill with the CLI's lint-rule citation surfaced to the user.

#### REQ: verify-report-resolution

The skill MUST auto-resolve the latest `_verify/<sha>.md` report at HEAD — defined as the report file whose `<sha>` matches `git rev-parse --short HEAD` if present, else the report whose embedded `revision:` YAML field is most recent in the branch's git history. The skill MUST parse the resolved report's top-of-file YAML summary block and surface, per AC ID, the `verdict` and `justification` fields. The skill MUST NOT accept an explicit `--report <path>` argument in the MVP (deferred — see `## Not Doing`). Parse failures on the verify report's YAML block MUST stop the skill with a clear error identifying the unparseable file.

### Trailer-driven AC ↔ commit mapping

#### REQ: trailer-grep-per-ac

For each AC ID parsed from the Feature, the skill MUST walk the current branch's git history with `git log --grep='^Verifies:.*<feature-slug>#ac:<ac-slug>'` (extended regexp). The skill MUST collect, per AC: the list of matching commit SHAs in chronological order and their commit messages. The skill MUST NOT pre-fetch full commit diffs — the drift-narrator subagent fetches diffs on demand via its own Bash tool (see `subagent-prompt`).

#### REQ: unmapped-detection

An AC with zero matching commits MUST be recorded with drift verdict `unmapped`. The skill MUST NOT dispatch a subagent for an unmapped AC and MUST NOT treat `unmapped` as a contradiction for exit-code purposes (see `exit-code-semantics`). The orchestrator MUST be the sole producer of the `unmapped` verdict; the value `unmapped` MUST NOT appear in the allowed-verdict set named in any subagent prompt (see `subagent-drift-contract`).

### Subagent dispatch

#### REQ: subagent-dispatch-serial

The skill MUST dispatch one drift-narrator subagent at a time, in the Feature's AC order. Parallel dispatch is explicitly out of scope for the MVP (see `## Not Doing`); the skill MUST NOT spawn more than one narrator subagent concurrently. The next AC's subagent is dispatched only after the previous AC's subagent has returned a verdict.

#### REQ: subagent-prompt

The drift-narrator subagent prompt MUST include, in this order: (1) the AC text in full `Given / When / Then` form with its AC ID; (2) the AC's verify verdict and verify justification snippet as parsed from the resolved `_verify/<sha>.md` report; (3) the list of matching commit SHAs (chronological order) paired with their commit messages; (4) the drift verdict contract from `subagent-drift-contract` (verbatim, so the subagent knows the required output shape); and (5) an explicit instruction to fetch commit diffs and read source files on demand via the subagent's own Bash tool (e.g., `git show <sha>`, `git show <sha> -- <path>`, `cat <path>`). The orchestrator MUST NOT pre-fetch diffs into the prompt; the subagent decides which commits and which files to read at what depth.

#### REQ: subagent-drift-contract

The drift-narrator subagent MUST return a drift verdict from the set `{no-drift, spec-tighter-than-code, code-tighter-than-spec, contradiction}`. The verdict line MUST be followed by a narrative of at most 500 characters and a list of evidence references (file paths, commit SHAs, optional `_tests/` scenario filenames). The four verdicts MUST be interpreted as: `no-drift` = the code matches what the AC requires; `spec-tighter-than-code` = the spec requires more than the code delivers; `code-tighter-than-spec` = the code enforces stricter behavior than the spec named; `contradiction` = the code does something the spec actively disallows or does not name as a valid alternative. Malformed verdicts (verdict outside the allowed set, missing narrative, narrative exceeding 500 characters) MUST be retried exactly once with a corrective prompt; a second malformed response MUST be recorded with drift verdict `error` for that AC. The orchestrator MUST be the sole producer of `error`; `error` MUST NOT appear in the allowed-verdict set named in any subagent prompt.

### Report generation

#### REQ: report-path

The skill MUST write the report to `spec/features/<feature-slug>/_recap/<sha>.md` where `<sha>` is the abbreviated git SHA of `HEAD` at run time. The `_recap/` directory MUST be created if absent.

#### REQ: report-yaml-summary

The report MUST begin with a fenced YAML block (delimited by ` ```yaml ` and ` ``` `) listing, in Feature AC order, each AC ID with its drift verdict and a one-line narrative snippet. The YAML block MUST also include a `verify_revision:` top-level field naming the `<sha>` of the resolved `_verify/<sha>.md` report. This block is the grep-target downstream skills consume. Example shape:

```yaml
feature: <feature-slug>
revision: <sha>
verify_revision: <verify-report-sha>
drift:
  - ac: <feature-slug>#ac:<ac-slug-1>
    verdict: no-drift
    narrative: "<one-line snippet>"
  - ac: <feature-slug>#ac:<ac-slug-2>
    verdict: spec-tighter-than-code
    narrative: "<one-line snippet>"
  - ac: <feature-slug>#ac:<ac-slug-3>
    verdict: unmapped
    narrative: "no commits reference this AC"
```

#### REQ: report-body

After the YAML block, the report MUST contain one `## AC: <ac-slug>` section per AC, each with: the drift verdict, the full narrative, the verify verdict and justification for the same AC (carried over verbatim from the resolved verify report), the commit list, and evidence references. The body is human-readable; the YAML block is machine-readable.

#### REQ: report-staged

After writing the report, the skill MUST stage it with `git add spec/features/<feature-slug>/_recap/<sha>.md`. The skill MUST stage the `_recap/README.md` index from `REQ:report-index-readme` in the same staging set. The skill MUST NOT run `git commit` — committing the report is the user's call (mirrors the `ideate` / `specify` / `plan` / `implement` / `verify` discipline).

#### REQ: report-index-readme

The skill MUST create `spec/features/<feature-slug>/_recap/README.md` if absent and MUST append a row for the current run to its `## Contents` table on every run. The README is the directory's index — without it, the project's `readme-exists` lint rule fails on the newly created `_recap/` directory. The README MUST follow SpecScore's scenarios-index conventions: an H1, a one-paragraph description, a `## Contents` table with columns `Report | Run revision | Verify revision | Drift summary`, an `## Open Questions` section (set to `None at this time.` when no questions are tracked), and the `*This document follows the https://specscore.md/index-specification*` footer. New runs append rows newest-first or newest-last consistently — pick one and document the choice in `## Approach` of the Plan, not as a Feature-level REQ.

### Exit semantics

#### REQ: exit-code-semantics

The skill MUST exit non-zero if and only if at least one AC has drift verdict `contradiction` or `error`. The verdicts `no-drift`, `spec-tighter-than-code`, `code-tighter-than-spec`, and `unmapped` MUST NOT contribute to a non-zero exit. The exit-code semantics make the three non-contradiction drift verdicts informational at the recap layer; `review` and `ship` are the eventual gates that MAY escalate `spec-tighter-than-code`, `code-tighter-than-spec`, or `unmapped` to blocking based on their own policies.

### Event emission

#### REQ: recap-completed-event

On a successful run (report written, regardless of drift outcomes), the skill MUST emit a `recap.completed` event via the convention in `skills/shared/synchestra-events.md`. Payload MUST include the following integer count fields, each ≥ 0, summing to the Feature's total AC count: `no_drift_count`, `spec_tighter_count`, `code_tighter_count`, `contradiction_count`, `unmapped_count`, `errored_count`. Payload MUST also include: `feature_slug`, `revision` (the HEAD SHA), `report_path` (relative to repo root), and `verify_report_path` (the path of the resolved `_verify/<sha>.md` the recap was compared against). Additional payload fields MAY be added in the future without breaking this contract; the six count fields and the four identity fields are the minimum. The event MUST be emitted exactly once per successful run. Per-AC drift details are NOT in the event payload — consumers read them from the report file at `report_path`.

### Transition

#### REQ: transition-to-review

After the report is written, staged, and the event is emitted, the skill MUST transition only to `specstudio:review` (or, while `review` is unshipped, hand back to the user with the report path and a recommendation to inspect drift items manually). The skill MUST NOT invoke `ideate`, `specify`, `plan`, `implement`, `verify`, `ship`, `frontend-design`, `mcp-builder`, or any other skill on transition.

## Architecture

- **Orchestrator (the skill body):** Pre-flight, Feature parse, verify-report resolution and parse, AC iteration, subagent dispatch coordination, drift aggregation, report write, event emission, transition.
- **Drift-narrator subagent (single built-in):** Receives the per-AC prompt, fetches commit diffs and reads source files on demand via its own Bash tool (per `subagent-prompt`), returns the drift verdict line per `subagent-drift-contract`.
- **Inputs:** Approved Feature at `spec/features/<feature-slug>/README.md`; latest `_verify/<sha>.md` report at HEAD; git history with `Verifies:` trailers.
- **Outputs:** Markdown report at `spec/features/<feature-slug>/_recap/<sha>.md` (staged, not committed); one `recap.completed` event in `.synchestra/events.jsonl`.
- **Dependencies:** `specstudio:verify` shipped; the `_verify/<sha>.md` report format (YAML head + body) stable enough to parse; the `Verifies:` trailer convention holds in commit history; `specscore` CLI Feature parser available.

## Interaction with Other Features

| Feature | Relationship |
|---|---|
| [Verify Skill](../verify/README.md) | `verify` is the upstream gate. Recap reads `_verify/<sha>.md` and refuses if no verify report exists. `verify`'s `transition-to-recap` REQ names recap as the only permitted transition; recap never invokes `verify`. |
| [Review Skill](../review/README.md) | `recap` is the upstream gate of `review`. `review` consumes `_recap/<sha>.md`'s YAML summary block to incorporate drift findings into its multi-axis review. While `review` is unshipped, recap hands back to the user. |
| [Implement Skill](../implement/README.md) | `implement`'s `Verifies:` commit trailers are the AC↔commit linkage recap consumes (same dependency verify has). Recap does not call implement; it relies on the trailer convention being preserved upstream. |
| [Third-Party Integration](../../third-party-integration/README.md) | Pluggable drift detectors are out of scope (mirrors verify's stance). The MVP ships one built-in narrator; a future Idea can add pluggability once a real second backend (e.g., a deterministic AST-diff detector) is on the table. |

## Acceptance Criteria

ACs are grouped here with explicit REQ back-references, mirroring the sibling `verify` Feature's style.

### AC: refuses-draft-feature (verifies REQ:requires-approved-feature)

**Given** a Feature at `spec/features/<slug>/README.md` with `**Status:** Draft`,
**When** the user runs `specstudio:recap <slug>`,
**Then** the skill MUST refuse to run, print the current Status, recommend `specstudio:specify` to re-approve, MUST NOT write any report, and MUST exit non-zero.

### AC: refuses-uncommitted-feature (verifies REQ:requires-feature-in-git-head)

**Given** a Feature with `**Status:** Approved` that exists in the working tree but has not been committed (`git cat-file -e HEAD:spec/features/<slug>/README.md` exits non-zero),
**When** the user runs `specstudio:recap <slug>`,
**Then** the skill MUST refuse, instruct the user to commit the Feature first, MUST NOT write any report, and MUST exit non-zero.

### AC: refuses-when-no-verify-report (verifies REQ:requires-verify-report)

**Given** an Approved, committed Feature whose `spec/features/<slug>/_verify/` directory either does not exist or contains zero `<sha>.md` report files reachable at HEAD,
**When** the user runs `specstudio:recap <slug>`,
**Then** the skill MUST refuse, recommend running `specstudio:verify <slug>` first, MUST NOT write any report, and MUST exit non-zero.

### AC: verify-report-resolved-to-latest-at-head (verifies REQ:verify-report-resolution)

**Given** an Approved Feature whose `_verify/` directory contains three reports `<sha-A>.md`, `<sha-B>.md`, `<sha-C>.md` where `<sha-C>` is the abbreviated SHA of `HEAD`,
**When** the user runs `specstudio:recap <slug>`,
**Then** the skill MUST resolve the verify report to `<sha-C>.md`, MUST parse its top-of-file YAML summary block, MUST surface per-AC verify verdicts and justifications to the subagent prompts, and MUST record `verify_revision: <sha-C>` in the resulting recap report's YAML summary.

**Given** an Approved Feature whose `_verify/` directory contains reports for older SHAs but no report at `HEAD` exactly,
**When** the user runs `specstudio:recap <slug>`,
**Then** the skill MUST resolve to the verify report whose embedded `revision:` field is most recent in the branch's git history, MUST parse its YAML summary, and MUST record that report's `revision` as `verify_revision:` in the recap report.

### AC: per-ac-trailer-grep (verifies REQ:trailer-grep-per-ac, REQ:feature-parse)

**Given** an Approved Feature with three ACs `slug#ac:a`, `slug#ac:b`, `slug#ac:c`, a resolvable verify report at HEAD, and commits in the branch containing `Verifies: slug#ac:a` (one commit) and `Verifies: slug#ac:b` (two commits) but no commit referencing `slug#ac:c`,
**When** the user runs `specstudio:recap slug`,
**Then** the skill MUST collect one commit for AC `a`, two commits for AC `b`, and zero commits for AC `c`, and MUST dispatch drift-narrator subagents only for ACs `a` and `b`.

### AC: unmapped-not-contradiction (verifies REQ:unmapped-detection, REQ:exit-code-semantics)

**Given** an Approved Feature where AC `slug#ac:c` has zero matching `Verifies:` trailers and all other ACs return `no-drift` from their drift-narrator subagents,
**When** the user runs `specstudio:recap slug`,
**Then** the recap report MUST mark AC `c` with `verdict: unmapped`, the skill MUST exit zero, and the YAML summary's `drift` list MUST include AC `c` with `verdict: unmapped`.

### AC: subagent-serial-dispatch (verifies REQ:subagent-dispatch-serial)

**Given** an Approved Feature with four ACs each having at least one matching commit and a resolvable verify report,
**When** the user runs `specstudio:recap slug`,
**Then** the skill MUST dispatch exactly one drift-narrator subagent at a time in AC order, and at no point during the run MUST more than one narrator subagent be concurrently in flight.

### AC: subagent-prompt-shape (verifies REQ:subagent-prompt, REQ:subagent-drift-contract)

**Given** AC `slug#ac:a` with two matching commits, a resolved verify report listing AC `a` with `verdict: pass` and a one-line justification snippet, and a `Given/When/Then` body for AC `a`,
**When** the orchestrator dispatches the drift-narrator subagent for AC `a`,
**Then** the dispatched prompt MUST contain (in order) the AC's full G/W/T text, the AC's verify verdict and verify justification snippet carried over verbatim from the resolved verify report, both commit SHAs paired with their commit messages, the verbatim drift verdict contract (allowed values, required narrative length, evidence-reference requirement), and an explicit instruction to fetch diffs and read source files on demand via the subagent's own Bash tool. The dispatched prompt MUST NOT contain pre-fetched commit diffs and MUST NOT include `unmapped` or `error` in the allowed-verdict set named to the subagent.

### AC: malformed-drift-verdict-retried-once (verifies REQ:subagent-drift-contract)

**Given** a drift-narrator subagent that returns a malformed verdict on first call (e.g., a verdict outside `{no-drift, spec-tighter-than-code, code-tighter-than-spec, contradiction}`, or a missing narrative, or a narrative exceeding 500 characters),
**When** the orchestrator parses the response,
**Then** the orchestrator MUST re-dispatch the subagent with a corrective prompt exactly once; if the second response is also malformed, the orchestrator MUST record the AC's drift verdict as `error` and MUST NOT call the subagent a third time.

### AC: report-path-and-staging (verifies REQ:report-path, REQ:report-staged)

**Given** an Approved Feature, a resolvable verify report, and a recap run that completes,
**When** the run finishes,
**Then** a Markdown report MUST exist at `spec/features/<feature-slug>/_recap/<sha>.md` where `<sha>` is the abbreviated git SHA of `HEAD`, the file MUST be staged via `git add`, and the skill MUST NOT have invoked `git commit`.

### AC: report-yaml-block-grep-target (verifies REQ:report-yaml-summary, REQ:report-body)

**Given** a completed recap run,
**When** a downstream consumer reads the report file,
**Then** the report's first content MUST be a fenced YAML block (delimited by ` ```yaml ` and ` ``` `) listing every AC with `ac`, `verdict`, and `narrative` fields in Feature AC order, AND the YAML block MUST include `feature:`, `revision:`, and `verify_revision:` top-level fields; below the YAML block, the report MUST contain one `## AC: <ac-slug>` section per AC, each carrying the drift verdict, the full narrative, the verify verdict and justification for the same AC (verbatim from the verify report), the commit list, and evidence references.

### AC: report-index-readme-created-and-updated (verifies REQ:report-index-readme, REQ:report-staged)

**Given** an Approved Feature whose `_recap/` directory does not yet exist,
**When** the user runs `specstudio:recap <slug>` for the first time,
**Then** the skill MUST create `spec/features/<slug>/_recap/README.md` containing an H1, a one-paragraph description, a `## Contents` table with exactly one row referencing the just-written `<sha>.md`, an `## Open Questions` section set to `None at this time.`, and the `*This document follows the https://specscore.md/index-specification*` footer; the README MUST be in the same staged set as the per-run report; and `specscore spec lint` MUST exit zero after the run.

**Given** an Approved Feature whose `_recap/README.md` already exists from a prior run with N rows in its `## Contents` table,
**When** the user runs `specstudio:recap <slug>` again,
**Then** the skill MUST append one row to the table for the current `<sha>.md` (preserving the existing rows in their order) and MUST stage the README alongside the new per-run report; the resulting table MUST have N+1 rows; lint MUST exit zero.

### AC: exit-non-zero-on-contradiction-only (verifies REQ:exit-code-semantics)

**Given** an Approved Feature where the drift-narrator subagent returns `contradiction` for at least one AC,
**When** the run completes and the report is written,
**Then** the skill MUST exit non-zero, and the YAML summary MUST mark the affected AC(s) with `verdict: contradiction`.

**Given** an Approved Feature where every AC returns one of `{no-drift, spec-tighter-than-code, code-tighter-than-spec}` and at least one AC is `unmapped`,
**When** the run completes and the report is written,
**Then** the skill MUST exit zero, regardless of how many ACs carry `spec-tighter-than-code`, `code-tighter-than-spec`, or `unmapped` verdicts.

**Given** an Approved Feature where at least one AC ends with drift verdict `error` (e.g., because the subagent returned malformed responses on both attempts per `REQ:subagent-drift-contract`),
**When** the run completes and the report is written,
**Then** the skill MUST exit non-zero, and the YAML summary MUST mark the affected AC(s) with `verdict: error`.

### AC: recap-completed-event-emitted (verifies REQ:recap-completed-event)

**Given** a recap run that completes (report written, regardless of drift outcomes) on a Feature with N total ACs,
**When** the orchestrator finishes the report write,
**Then** exactly one `recap.completed` event MUST be appended to `.synchestra/events.jsonl` (or emitted via `synchestra emit` when the CLI is available), with payload fields `feature_slug`, `revision`, `report_path`, `verify_report_path`, `no_drift_count`, `spec_tighter_count`, `code_tighter_count`, `contradiction_count`, `unmapped_count`, and `errored_count`,
**And** each of the six count fields MUST be a non-negative integer,
**And** `no_drift_count + spec_tighter_count + code_tighter_count + contradiction_count + unmapped_count + errored_count` MUST equal N,
**And** the payload MUST NOT include per-AC drift details.

### AC: transition-to-review-only (verifies REQ:transition-to-review, REQ:hard-gate)

**Given** a successful recap run,
**When** the skill prepares to transition,
**Then** the skill MUST offer transition only to `specstudio:review` (or, when `review` is unshipped, hand back to the user with the report path), and MUST NOT invoke `ideate`, `specify`, `plan`, `implement`, `verify`, `ship`, `writing-plans`, `frontend-design`, or `mcp-builder`.

## Rehearse Integration

Per the rehearse-heuristic, every AC above is testable via filesystem fixtures (mock repo with seeded git history + Feature files + a stub `_verify/<sha>.md` report) and event-payload inspection. The non-testable cases (e.g., the drift-narrator subagent's judgment quality) are validated at the assumption-validation layer of the source Idea, not as Rehearse scenarios.

Rehearse stubs for each AC are scaffolded at `_tests/<ac-slug>.md` with `**Status:** pending`; authoring scenario steps follows the implementation plan.

## Not Doing

Inherited from the source Idea and pinned here:

- **Auto-running after verify** — explicit invocation only. `verify`'s `transition-to-recap` REQ already names recap as the recommended next step but does not auto-invoke; preserving the user's deliberate gate is part of the pipeline contract.
- **Editing the Feature on the user's behalf** — recap flags drift; humans author Feature edits. Auto-mutation is a separable design surface that should not be smuggled in via recap.
- **Auto-drafting Proposals or seeding `_proposals/<slug>.md` stubs** — separable next Idea once recap is shipped and the human drift-resolution workflow is observed in practice.
- **Architectural or stylistic drift detection** — AC-level only. Style and architecture are `review`'s concern, not recap's.
- **Comparing against the Feature at Plan-approval time** — HEAD only. Multi-baseline diffing is overfitting before we know the single baseline is wrong.
- **Pluggable drift detectors or classification schemes** — same N=1 overfit trap verify's Idea named. Deferred to a dedicated future Idea once a real second backend (e.g., a deterministic AST-diff detector) is on the table.
- **Parallel subagent dispatch** — match verify's serial discipline for MVP. Revisit if real runtimes hurt.
- **Plan-completeness checks** (every Plan task committed; unplanned trailers) — separable; widens scope past AC-level drift. A focused plan-recap Idea can land later if the need is real.
- **`specscore.yaml recap:` block** — premature backend-naming schema (same overfit risk verify named).
- **Auto-promotion of drift items into the Feature's Open Questions section** — recap stages a separate artifact; the human decides what to back-port.
- **Explicit `--report <path>` argument** — MVP auto-resolves the latest verify report at HEAD. A `--report` flag lands in a follow-on Idea if a real workflow needs to recap against an older verify run.
- **Stale-verify-report detection** — MVP resolves "latest" by SHA/recency without checking whether new commits have landed since verify ran. Stale-report handling is separable (see `## Open Questions`).

## Open Questions

- The role of the verify report's evidence references in the drift-narrator subagent prompt — currently passed verbatim alongside the verify verdict snippet. If subagents over-anchor to verify's narrative and miss drift verify itself missed, we may need to truncate, paraphrase, or omit the verify justification in the prompt. Defer the decision to dogfood observation; the AC for `subagent-prompt-shape` requires the verbatim carry-over today.
- Whether recap should warn when the resolved `_verify/<sha>.md` is stale relative to HEAD (verify ran at SHA X, but HEAD is now SHA Y with new commits not seen by verify). MVP auto-resolves "latest" by SHA/recency without staleness detection; the report's `verify_revision:` field makes the staleness visible to humans but does not block. A separable Idea may add a staleness gate later.
- Whether the report body should include the resolved verify report's full evidence references per AC, or only the verify verdict + one-line justification snippet. MVP requires verbatim carry-over of verdict + justification only; full evidence-reference duplication is deferred to keep recap reports compact.

---
*This document follows the https://specscore.md/feature-specification*
