---
name: recap
description: |
  Consumes an approved SpecScore Feature plus the latest `specstudio:verify`
  report at HEAD and produces a per-AC drift report at
  spec/features/<feature-slug>/_recap/<sha>.md. For each AC, dispatches one
  built-in AI subagent (serial, not parallel) that classifies divergence
  between spec and code into the 4-bucket verdict set
  {no-drift, spec-tighter-than-code, code-tighter-than-spec, contradiction}.
  Aggregates verdicts into a Markdown report opened by a grep-friendly YAML
  summary block, emits `recap.completed`, and transitions only to
  `specstudio:review`. Applies publication policy to the report checkpoint.
  Trigger: "recap", "/recap", "specstudio:recap", or the explicit downstream
  transition from `specstudio:verify`.
aliases: [recap]
---

# Recap

Turn an approved SpecScore Feature plus its latest verify report into a per-AC drift report. Close the SpecStudio pipeline's recap slot so `review` and `ship` have an honest drift gate to consume.

## Hard Gate

<HARD-GATE>
Do NOT invoke `specstudio:ideate`, `specstudio:specify`, `specstudio:plan`, `specstudio:implement`, `specstudio:verify`, `specstudio:ship`, `writing-plans`, `frontend-design`, `mcp-builder`, or ANY other skill until ALL of the following are true:
  1. Pre-flight passed: the input slug resolves to `spec/features/<feature-slug>/README.md`, the Feature's `**Status:**` is in `{Approved, Implementing, Stable}`, the Feature exists at git HEAD (`git cat-file -e HEAD:spec/features/<feature-slug>/README.md` exits zero), AND `spec/features/<feature-slug>/_verify/` exists and contains at least one `<sha>.md` report file reachable at HEAD.
  2. The Feature parsed cleanly via the `specscore` CLI's Feature parser; the ordered AC ID list is available.
  3. The verify report has been resolved (preferring the report whose `<sha>` matches `git rev-parse --short HEAD`; otherwise the report whose embedded `revision:` YAML field is most recent in branch history) and its top-of-file YAML summary block has been parsed into a per-AC `{verify_verdict, verify_justification}` map.
  4. The report has been written to `spec/features/<feature-slug>/_recap/<sha>.md` with a fenced ` ```yaml ` summary block (top-level `feature:`, `revision:`, `verify_revision:`, and `drift:` list with one entry per AC) followed by one `## AC: <ac-slug>` body section per AC.
  5. The `_recap/README.md` index has been created (if absent) or updated with the current run's row (if present), per `## Report Format → Index README`.
  6. Both the report and the `_recap/README.md` index are in the publication manifest and the `recap.completed` publication checkpoint has been resolved, disclosed, and applied.
  7. The `recap.completed` event has been emitted exactly once.

The only skill invoked after `specstudio:recap` is `specstudio:review` (or — while `review` is unshipped — a hand-back to the user with the report path and a recommendation to inspect drift items manually). The skill MUST NOT invoke `ideate`, `specify`, `plan`, `implement`, `verify`, `ship`, `writing-plans`, `frontend-design`, or `mcp-builder` on transition.
</HARD-GATE>

## When to Use

- An approved Feature at `spec/features/<feature-slug>/README.md` exists in git HEAD, `specstudio:verify` has already produced at least one report at `spec/features/<feature-slug>/_verify/<sha>.md`, and the user wants a per-AC drift verdict comparing what the spec asks for against what the commits actually delivered.
- `specstudio:verify` has just transitioned via its `transition-to-recap` REQ and the user wants to continue the pipeline.
- The user wants to re-recap a Feature after additional `Verifies:`-trailed commits and a fresh verify run have landed.

**Refuse and redirect when:**

- The Feature's `**Status:**` is `Draft` or `Under Review` → print the current Status and recommend `specstudio:specify` to re-approve. Write no report. Exit non-zero. (AC: `refuses-draft-feature`)
- The Feature exists only in the working tree (uncommitted) → instruct the user to commit the Feature first. Write no report. Exit non-zero. (AC: `refuses-uncommitted-feature`)
- The Feature's `_verify/` directory does not exist or contains zero `<sha>.md` report files reachable at HEAD → recommend running `specstudio:verify <feature-slug>` first. Write no report. Exit non-zero. Recap-without-verify is a category error: there is nothing to recap against. (AC: `refuses-when-no-verify-report`)
- The user asks the skill to bypass publication policy, commit unrelated paths, or push without branch-policy approval → refuse.
- The user asks the skill to invoke `ship`, `review` (when unshipped), or any non-`review` skill → refuse; the only permitted downstream transition is `specstudio:review`.

## Pre-Flight

1. **Resolve input.** The skill MUST accept exactly one positional argument — the Feature slug — and resolve it to `spec/features/<feature-slug>/README.md`.
2. **Status guard.** Read the Feature's body-metadata `**Status:**` line. Refuse if it is outside `{Approved, Implementing, Stable}`. On refusal, print the current Status, recommend `specstudio:specify` to re-approve, write no report, and exit non-zero.
3. **HEAD-existence guard.** Run `git cat-file -e HEAD:spec/features/<feature-slug>/README.md`. On non-zero exit, refuse with an instruction to commit the Feature first, write no report, and exit non-zero. The `Verifies:` trailer convention requires the Feature to exist in git history.
4. **Verify-report-presence guard.** Check that `spec/features/<feature-slug>/_verify/` exists AND contains at least one `<sha>.md` file reachable at HEAD. If the directory is absent, or it exists but contains zero `<sha>.md` reports reachable at HEAD, refuse with a recommendation to run `specstudio:verify <feature-slug>` first, write no report, and exit non-zero.
5. **Resolve HEAD SHA.** Capture `git rev-parse --short HEAD` once. This SHA is the `<sha>` in the report path and the `revision` in the YAML summary and the event payload.

## Checklist

Create a task for each and complete in order:

1. **Pre-flight** (steps above). Refusals exit immediately; do not proceed.
2. **Parse the Feature.** Delegate to the `specscore` CLI's Feature parser. Surface the ordered list of AC IDs in the form `<feature-slug>#ac:<ac-slug>` paired with each AC's full `Given / When / Then` text. Parse failures stop the skill with the CLI's lint-rule citation surfaced verbatim to the user.
3. **Resolve and parse the verify report.** Auto-resolve the latest `_verify/<sha>.md` report at HEAD:
   - **First preference:** the report file whose `<sha>` matches `git rev-parse --short HEAD` exactly.
   - **Fallback:** the report whose embedded top-of-file YAML `revision:` field is most recent in the branch's git history (use `git log` over `_verify/` paths to determine recency).

   Parse the resolved report's top-of-file fenced YAML summary block and build a per-AC map keyed by `<feature-slug>#ac:<ac-slug>` whose value is `{verify_verdict, verify_justification}`. Record the resolved report's path (e.g., `spec/features/<feature-slug>/_verify/<verify-sha>.md`) and its `revision:` value — both flow into the recap report's YAML (`verify_revision:`) and the event payload (`verify_report_path`). Parse failures on the verify report's YAML block MUST stop the skill with a clear error identifying the unparseable file. The skill MUST NOT accept a `--report <path>` override in MVP.
4. **Collect commits per AC.** For each AC ID in Feature order, run:

   ```
   git log --extended-regexp --grep='^Verifies:.*<feature-slug>#ac:<ac-slug>'  --format='%H%n%s%n%b%x00'
   ```

   Collect the matching commit SHAs in chronological (oldest-first) order paired with their commit messages (subject + body). Do NOT pre-fetch commit diffs — the drift-narrator subagent fetches diffs on demand. An AC with zero matching commits is recorded as `unmapped` (orchestrator-produced) and skipped for subagent dispatch.
5. **Dispatch drift-narrator subagents serially.** For each mapped AC (commit count ≥ 1) in Feature AC order, dispatch one drift-narrator subagent via the Agent tool with `subagent_type: general-purpose`. Construct the prompt per `## Drift-Narrator Subagent Contract` below. Wait for the subagent to return a parseable verdict before dispatching the next AC's subagent. At no point during the run MUST more than one narrator subagent be concurrently in flight.
6. **Parse the verdict.** Validate per `## Drift-Narrator Subagent Contract`. On a malformed first response, re-dispatch the same subagent exactly once with a corrective prompt that quotes the failure (verdict outside `{no-drift, spec-tighter-than-code, code-tighter-than-spec, contradiction}`, missing narrative, or narrative exceeding 500 characters) and reiterates the verdict contract. On a malformed second response, record the AC's drift verdict as `error` (orchestrator-produced) and MUST NOT call the subagent a third time.
7. **Tally counts.** Compute six non-negative integer counts over the full AC list: `no_drift_count`, `spec_tighter_count`, `code_tighter_count`, `contradiction_count`, `unmapped_count`, `errored_count`. Their sum equals the Feature's total AC count.
8. **Write the report.** Create `spec/features/<feature-slug>/_recap/` if absent. Write the report file to `spec/features/<feature-slug>/_recap/<sha>.md` per `## Report Format` below.
9. **Write or update the `_recap/README.md` index.** Per `## Report Format → Index README` below. If absent, create it with one row in `## Contents`. If present, append one row (newest-last) for the current run, preserving prior rows. Without this step the project's `readme-exists` lint rule will fail on the newly created `_recap/` directory.
10. **Publication checkpoint.** Add `spec/features/<feature-slug>/_recap/<sha>.md` and `spec/features/<feature-slug>/_recap/README.md` to the manifest. Apply [publication-policy.md](../shared/publication-policy.md) for `recap.completed`, staging only manifest paths and committing/pushing only when policy and safety allow.
11. **Emit `recap.completed`.** Per `## Event Emission` below, including `publication_result`. Exactly once per successful run.
12. **Transition.** Per `## Promotion Boundary` below. Only `specstudio:review` (or hand-back when `review` is unshipped).
13. **Determine exit code.** Per `## Exit Semantics` below. Non-zero iff `contradiction_count + errored_count > 0`. Otherwise zero.
14. **Throughout** — watch for sidekick ideas (e.g., a flaky drift-narrator shape, a recurrent verify-vs-recap disagreement pattern, a deferred plan-completeness check). When an out-of-scope improvement surfaces, invoke `specstudio:sidekick` with a one-liner, acknowledge in one line, and return to the current checklist step immediately. Do not derail.

## Drift-Narrator Subagent Contract

Each drift-narrator subagent is dispatched with an **isolated prompt** — it MUST NOT inherit the parent session's context. Construct the prompt freshly per AC.

### Prompt shape (five required parts, in this order)

1. **AC identification and full text.** The AC ID (`<feature-slug>#ac:<ac-slug>`) followed by its complete `Given / When / Then` text quoted verbatim from the source Feature.
2. **Verify verdict carry-over (verbatim).** The AC's verify verdict and verify justification snippet copied verbatim from the resolved `_verify/<sha>.md` report's YAML summary block for the same AC ID. Format:

   ```
   ## Verify result for this AC

   - verdict: <pass | fail | error | unmapped>
   - justification: <one-line snippet as recorded in the verify report>
   ```

   The carry-over is verbatim by contract; it gives the drift-narrator the orienting baseline of what verify itself concluded.
3. **Matching commits.** The ordered list of commit SHAs (chronological, oldest-first) paired with each commit's full message (subject + body). Format:

   ```
   ## Matching commits

   - <sha-1>
     <commit message subject + body>
   - <sha-2>
     <commit message subject + body>
   ```

   The orchestrator MUST NOT include pre-fetched commit diffs in the prompt. The subagent fetches diffs and reads source files on demand via its own Bash tool.
4. **Drift verdict contract (verbatim).** Quote the following block into the prompt verbatim, so the subagent knows the required output shape:

   ```
   Return exactly one drift verdict from the set:
     {no-drift, spec-tighter-than-code, code-tighter-than-spec, contradiction}.

   Required output shape:

     verdict: <no-drift | spec-tighter-than-code | code-tighter-than-spec | contradiction>
     narrative: <one-line snippet, MAXIMUM 500 characters>
     evidence:
       - <file path, commit SHA, or _tests/<ac-slug>.md reference>
       - ...

   Verdict semantics:
   - `no-drift`                  = the code matches what the AC requires.
   - `spec-tighter-than-code`    = the spec requires more than the code delivers.
   - `code-tighter-than-spec`    = the code enforces stricter behavior than the spec named.
   - `contradiction`             = the code does something the spec actively disallows
                                   or does not name as a valid alternative.

   The values `unmapped` and `error` are produced only by the orchestrator
   (`unmapped` for ACs with zero matching commits; `error` for ACs whose subagent
   returned malformed responses twice) and MUST NOT appear in your response.

   Malformed responses (verdict outside the allowed set, missing narrative, or
   narrative exceeding 500 characters) will be re-dispatched exactly once with a
   corrective prompt; a second malformed response is recorded as `error`.
   ```

5. **Fetch-on-demand instruction.** An explicit instruction that the subagent fetches commit diffs and reads source files on its own via Bash. Suggested commands to mention:

   ```
   git show <sha>
   git show <sha> -- <path>
   git show <sha> --stat
   cat <path>
   ```

   Include the guidance: "Read at the depth required to reach a drift verdict; do not enumerate every file in every commit if the verdict is obvious from one diff."

### Out of scope for the prompt

- **No pre-fetched diffs.** The orchestrator MUST NOT inline `git show` output into the prompt. Diff fetching is the subagent's responsibility.
- **No `_tests/` mandate.** The subagent MAY reference `_tests/<ac-slug>.md` scenarios in its `evidence:` list when relevant, but the prompt MUST NOT mandate reading them — Rehearse-scenario authoring is deferred per the source Feature's `## Rehearse Integration`.
- **No `unmapped` or `error` verdict.** Both are orchestrator-produced. The allowed-verdict set named in the prompt is `{no-drift, spec-tighter-than-code, code-tighter-than-spec, contradiction}` ONLY.

### Verdict parsing

A well-formed response contains:

- A `verdict:` line whose value is exactly one of `no-drift`, `spec-tighter-than-code`, `code-tighter-than-spec`, `contradiction`.
- A `narrative:` line whose value is non-empty and at most 500 characters.
- An `evidence:` list with zero or more entries (file paths, commit SHAs, `_tests/` scenario filenames, or other references).

Anything outside this shape is malformed. Apply the retry-once-then-error semantics per checklist step 6.

## Report Format

### Path

```
spec/features/<feature-slug>/_recap/<sha>.md
```

`<sha>` is the abbreviated git SHA of HEAD at run time (`git rev-parse --short HEAD`). Create the `_recap/` directory if absent.

### YAML summary block (grep target)

The report MUST open with a fenced YAML block (delimited by ` ```yaml ` and ` ``` `) listing every AC in Feature AC order. This block is the grep target downstream skills (`review`, eventually `ship`) consume. The block MUST include `feature:`, `revision:`, AND `verify_revision:` top-level fields; the per-AC `drift:` list entries MUST use `ac`, `verdict`, and `narrative` field names (note: `narrative`, not `justification` — the recap report's body section also carries verify's `justification` verbatim, so the two field names stay disambiguated).

```yaml
feature: <feature-slug>
revision: <sha>
verify_revision: <verify-report-sha>
drift:
  - ac: <feature-slug>#ac:<ac-slug-1>
    verdict: no-drift
    narrative: "<one-line snippet, ≤500 chars>"
  - ac: <feature-slug>#ac:<ac-slug-2>
    verdict: spec-tighter-than-code
    narrative: "<one-line snippet>"
  - ac: <feature-slug>#ac:<ac-slug-3>
    verdict: unmapped
    narrative: "no commits reference this AC"
  - ac: <feature-slug>#ac:<ac-slug-4>
    verdict: contradiction
    narrative: "<one-line snippet>"
  - ac: <feature-slug>#ac:<ac-slug-5>
    verdict: error
    narrative: "<one-line snippet>"
```

Every AC from the source Feature MUST appear exactly once, in Feature AC order. The six allowed `verdict` values are `no-drift`, `spec-tighter-than-code`, `code-tighter-than-spec`, `contradiction`, `unmapped`, and `error`. (Four are subagent-produced; `unmapped` and `error` are orchestrator-produced.)

### Body (per-AC sections)

After the YAML block, the report MUST contain one `## AC: <ac-slug>` section per AC, in Feature AC order. Each section contains:

```markdown
## AC: <ac-slug>

**Drift verdict:** <no-drift | spec-tighter-than-code | code-tighter-than-spec | contradiction | unmapped | error>

**Narrative:** <full narrative text — same as the YAML one-liner if short, or
expanded for human readability if the subagent supplied a richer rationale>

**Verify verdict:** <pass | fail | error | unmapped> (carried verbatim from the resolved verify report)

**Verify justification:** <one-line snippet, carried verbatim from the resolved verify report>

**Commits:**
- <sha-1> — <commit subject>
- <sha-2> — <commit subject>

(Or: "No commits reference this AC." for `unmapped` ACs.)

**Evidence:**
- <file path, commit SHA, _tests/<ac-slug>.md reference, or other reference>
- ...
```

The YAML block is the machine-readable surface; the body is the human-readable surface. The body MUST carry the verify verdict and verify justification verbatim from the resolved verify report so a reviewer can see, side-by-side per AC, what verify concluded and what recap concluded.

### Index README

The skill MUST create `spec/features/<feature-slug>/_recap/README.md` if absent and MUST append a row for the current run to its `## Contents` table on every run. The README is the directory's index — without it, the project's `readme-exists` lint rule fails on the newly created `_recap/` directory.

**Row ordering (Approach decision):** rows are appended **newest-last**, mirroring `verify`'s index ordering. The most recent run is the last row in the table.

**Shape (when creating from absent):**

```markdown
# Recap Reports — <feature-slug>

Per-run recap reports produced by `specstudio:recap`. Each report is named `<sha>.md` where `<sha>` is the abbreviated git SHA of `HEAD` at run time. Each row records which verify report the recap was compared against (`Verify revision`) so reviewers can trace the drift gate end-to-end.

## Contents

| Report | Run revision | Verify revision | Drift summary |
|---|---|---|---|
| [<sha>.md](<sha>.md) | <sha> | <verify-report-sha> | <one-line drift summary, e.g., "3 no-drift, 1 spec-tighter, 0 code-tighter, 0 contradiction, 1 unmapped, 0 errored"> |

## Open Questions

None at this time.

---
*This document follows the https://specscore.md/index-specification*
```

**Append rule (when README already exists):**

Locate the `## Contents` table. Insert a new row at the bottom of the table (newest-last). Preserve every prior row in its existing order — never reorder, never delete. The new row's `Report` cell is a Markdown link `[<sha>.md](<sha>.md)`; the `Run revision` cell is the abbreviated HEAD SHA; the `Verify revision` cell is the abbreviated SHA of the resolved verify report; the `Drift summary` cell is a one-line tally derived from the six flat counts.

If the table is missing entirely (e.g., README was hand-edited), the skill MUST refuse to write the row, surface the missing-table error to the user, and recommend manual repair before re-running.

### Publication discipline

After writing the report and the index README, the skill MUST add both to the checkpoint manifest and apply [publication-policy.md](../shared/publication-policy.md) for `recap.completed`. If the allowed actions include `stage`, stage only those manifest paths. If they include `commit` or `push`, run the shared unrelated-index and branch-safety checks first. The disclosure must name the resolved policy, executed actions, skipped actions, and manifest paths.

### No-mapped-commits edge case

When a Feature has at least one verify report at HEAD but zero `Verifies:` trailers for any AC in the entire branch history, the skill MUST still:

- Write the report at the canonical path.
- Mark every AC `unmapped` in both the YAML block and the body sections.
- Apply publication policy to the report and index manifest.
- Emit `recap.completed` with `unmapped_count` equal to the Feature's total AC count, the other five counts equal to zero, and `publication_result`.
- Exit zero.

The report itself communicates that nothing has been mapped yet; the absence of a report is not the right signal. (Note: this edge case requires a verify report to be present — recap's pre-flight refuses outright when `_verify/` is empty.)

## Exit Semantics

The skill MUST exit non-zero **if and only if** `contradiction_count + errored_count > 0`. Equivalently:

| Tally | Exit code |
|---|---|
| Every AC `no-drift`, `spec-tighter-than-code`, `code-tighter-than-spec`, or `unmapped` (any mix) | `0` |
| At least one AC `contradiction` | non-zero |
| At least one AC `error` | non-zero |
| At least one `contradiction` AND at least one `error` | non-zero |
| Every AC `unmapped` (no-mapped-commits edge case) | `0` |

`no-drift`, `spec-tighter-than-code`, `code-tighter-than-spec`, and `unmapped` are **informational** at the recap layer. `review` and `ship` are the eventual gates that MAY escalate `spec-tighter-than-code`, `code-tighter-than-spec`, or `unmapped` to blocking based on their own policies; recap does not.

A pre-flight refusal (Draft Feature, uncommitted Feature, no verify report) exits non-zero without writing a report — those refusals are distinct from the verdict-tally exit code above.

## Event Emission

After the report is written and publication policy has been applied (and only on a successful run — pre-flight refusals do NOT emit), the skill MUST emit exactly one `recap.completed` event via the convention in [events.md](../shared/events.md). Use `specscore event emit <event.yaml>` when the CLI is available; fall back to appending the event JSONL line to `.specscore/events.jsonl`.

Payload shape (flat counts — NOT a nested `drift_counts` object):

```yaml
event: recap.completed
version: 1
uuid: <generated>
timestamp: <ISO-8601>
actor:
  kind: skill
  id: skill:specstudio:recap
artifact:
  type: feature
  id: <feature-slug>
  path: spec/features/<feature-slug>/README.md
  revision: <sha>
payload:
  feature_slug: <feature-slug>
  revision: <sha>
  report_path: spec/features/<feature-slug>/_recap/<sha>.md
  verify_report_path: spec/features/<feature-slug>/_verify/<verify-sha>.md
  no_drift_count: <int ≥ 0>
  spec_tighter_count: <int ≥ 0>
  code_tighter_count: <int ≥ 0>
  contradiction_count: <int ≥ 0>
  unmapped_count: <int ≥ 0>
  errored_count: <int ≥ 0>
publication_result:
  resolved_actions: [<stage | commit | push>, ...]
  executed_actions: [<stage | commit | push>, ...]
  skipped_actions: [{action: <stage | commit | push>, reason: <string>}]
  commit_sha: <git SHA> | null
  push_target: <remote>/<branch> | null
```

**Invariants:**

- All six count fields are non-negative integers.
- `no_drift_count + spec_tighter_count + code_tighter_count + contradiction_count + unmapped_count + errored_count` equals the Feature's total AC count.
- The payload MUST NOT include per-AC drift details. Consumers read those from the report file at `report_path`.
- The event MUST be emitted exactly once per successful run.
- Additional payload fields MAY be added in the future without breaking this contract; the six count fields plus the four identity fields (`feature_slug`, `revision`, `report_path`, `verify_report_path`) are the minimum.

## Promotion Boundary

The next skill is `specstudio:review`, and only `specstudio:review`.

### Transition

After the report is written, publication policy is applied, and `recap.completed` is emitted:

- If `specstudio:review` is shipped: offer to transition the user to `specstudio:review`, passing the recap `report_path` as input.
- If `specstudio:review` is unshipped: hand back to the user with the report path and a recommendation to inspect drift items manually:

  > "Recap complete. Report written at `spec/features/<feature-slug>/_recap/<sha>.md` and publication policy applied. The `specstudio:review` skill is not yet shipped — review the YAML summary block in the report for per-AC drift verdicts."

The skill MUST NOT invoke `ideate`, `specify`, `plan`, `implement`, `verify`, `ship`, `writing-plans`, `frontend-design`, or `mcp-builder` on transition. The hard gate above pins this; the transition step honors it.

## Tone

The skill MUST NOT yes-machine soft drift verdicts. When a drift-narrator subagent returns `no-drift` on a commit set that obviously contradicts the AC, the skill MUST trust the subagent's verdict for THIS run (the orchestrator does not re-judge), but MUST surface the report honestly — the YAML block and the body section carry the verdict as returned. Re-judgment is `review`'s job, not `recap`'s.

When the skill detects a pre-flight refusal (Draft Feature, uncommitted Feature, missing verify report), it MUST say so with the specific cited cause (current Status value, missing-from-HEAD path, missing `_verify/` directory) and propose the alternative (`specstudio:specify`, `git commit`, `specstudio:verify`). Honest specificity, not performative agreement.

When the subagent returns malformed responses twice in a row, the skill records `error` and moves on — it MUST NOT loop infinitely.

## Verification

- [ ] Pre-flight checks passed: input slug resolves; Feature `**Status:**` ∈ {Approved, Implementing, Stable}; Feature exists at git HEAD; `_verify/` exists with at least one `<sha>.md` report reachable at HEAD
- [ ] Feature parsed cleanly via `specscore` CLI; ordered AC list captured
- [ ] Verify report resolved (HEAD-SHA exact match preferred; most-recent-by-revision fallback otherwise); its YAML summary parsed into a per-AC `{verify_verdict, verify_justification}` map; resolved path + SHA captured for downstream use
- [ ] For each AC: `git log --grep='^Verifies:.*<feature-slug>#ac:<ac-slug>'` ran; matched SHAs + commit messages collected; NO diffs pre-fetched
- [ ] Subagent dispatch was serial: at no point were two drift-narrator subagents concurrently in flight
- [ ] Each per-AC subagent prompt contained, in order: full G/W/T text, the AC's verify verdict + justification carried over verbatim, commit SHAs + messages, the verbatim drift verdict contract naming only `{no-drift, spec-tighter-than-code, code-tighter-than-spec, contradiction}`, and the fetch-on-demand Bash instruction; no pre-fetched diffs; `unmapped` and `error` NEVER named to the subagent
- [ ] Malformed drift verdicts were retried exactly once; second-time malformed verdicts were recorded as `error` (orchestrator-produced)
- [ ] Report written at `spec/features/<feature-slug>/_recap/<sha>.md` with the fenced YAML block first (top-level `feature:`, `revision:`, `verify_revision:`; per-AC `ac` + `verdict` + `narrative` entries in Feature AC order), then `## AC:` body sections carrying drift verdict + narrative + verify verdict + verify justification + commits + evidence
- [ ] `_recap/README.md` index created (if absent) or row-appended (if present) with columns `Report | Run revision | Verify revision | Drift summary`; table preserves prior rows in their existing order (newest-last)
- [ ] Both report and `_recap/README.md` added to the checkpoint manifest; publication policy applied with disclosure
- [ ] `specscore spec lint` exits zero after the run (no `readme-exists` failure on the `_recap/` directory)
- [ ] `recap.completed` event emitted exactly once; payload contains the four identity fields (`feature_slug`, `revision`, `report_path`, `verify_report_path`), the six flat count fields summing to the Feature's total AC count, and `publication_result`; no per-AC details in the payload
- [ ] Exit code is non-zero iff `contradiction_count + errored_count > 0`; otherwise zero (the four other verdicts are informational and never contribute to non-zero exit)
- [ ] No-mapped-commits edge case (every AC `unmapped`) still produced a complete report, applied publication policy, emitted the event, and exited zero
- [ ] On transition: only `specstudio:review` invoked (or hand-back when unshipped); no other skill touched

## Red Flags

- Pre-fetching commit diffs into the subagent prompt (the subagent fetches diffs on demand; the orchestrator does not)
- Naming `unmapped` or `error` in the subagent's allowed-verdict set (both are orchestrator-produced; the prompt names `{no-drift, spec-tighter-than-code, code-tighter-than-spec, contradiction}` only)
- Mandating the subagent read `_tests/<ac-slug>.md` files (the subagent MAY reference them in evidence; the orchestrator MUST NOT mandate it — Rehearse-scenario authoring is deferred)
- Dispatching two drift-narrator subagents concurrently (serial only; parallelism is explicitly deferred per the source Idea's `## Not Doing`)
- Looping the malformed-verdict retry more than once (one retry; second malformed → `error`; never a third call)
- Allowing a narrative longer than 500 characters to pass the verdict-contract check
- Hard-coding stage-only report handoff instead of resolving publication policy for `recap.completed`
- Committing unrelated staged paths or pushing without branch-policy approval
- Writing the report to `docs/` or to a path other than `spec/features/<feature-slug>/_recap/<sha>.md`
- Treating `spec-tighter-than-code`, `code-tighter-than-spec`, or `unmapped` as a non-zero-exit signal (all three are informational at the recap layer; `review` and `ship` escalate them later)
- Skipping the `recap.completed` event because "no contradictions" — the event fires on every successful run regardless of drift outcomes
- Emitting `recap.completed` with per-AC drift details in the payload (per-AC data lives in the report file; the event payload carries flat counts only)
- Nesting the six counts under a `drift_counts:` key (the contract is flat: `no_drift_count`, `spec_tighter_count`, `code_tighter_count`, `contradiction_count`, `unmapped_count`, `errored_count`)
- Omitting `verify_revision:` from the recap report's YAML or `verify_report_path` from the event payload (the verify gate is traceable end-to-end by contract)
- Accepting a `--report <path>` argument (MVP auto-resolves the latest verify report at HEAD; explicit overrides are deferred per `## Not Doing`)
- Skipping the verify-report-presence pre-flight (recap-without-verify is a category error — there is nothing to recap against)
- Invoking any skill other than `specstudio:review` on transition (no `verify`, no `ship`, no `writing-plans`, no `frontend-design`, no `mcp-builder`)
- Refusing to write a report when no commits map to any AC (the no-mapped-commits edge case still produces a complete `unmapped`-only report — provided the pre-flight verify-report-presence guard passed)
- Writing the per-run report but forgetting `_recap/README.md` (the `readme-exists` lint rule will fail on the newly created `_recap/` directory)
- Reordering or deleting prior rows in `_recap/README.md`'s `## Contents` table (the skill only appends; prior runs' rows are preserved verbatim, newest-last)
- Resolving the verify report by mtime or any signal other than HEAD-SHA-exact-match (preferred) or YAML `revision:` recency (fallback)

## References

- [Feature: Recap Skill](../../spec/features/skills/recap/README.md) — the SpecScore Feature this skill implements.
- [Feature: Verify Skill](../../spec/features/skills/verify/README.md) — the upstream Feature whose `_verify/<sha>.md` report this skill consumes.
- [Feature: Implement Skill](../../spec/features/skills/implement/README.md) — the Feature whose `Verifies:` commit-message trailer convention this skill (and verify) consumes.
- [Plan: Recap Skill MVP](../../spec/plans/recap.md) — the seven-task plan this skill realizes.
- [Verify Skill](../verify/SKILL.md) — the architectural twin; recap mirrors verify's structure exactly with the documented deltas (verify-report pre-flight, verify-report resolution, 4-bucket drift verdict, 500-char narrative cap, 6-count tally, recap-only event name, transition to review).
- [philosophy.md](../shared/philosophy.md) — shared tenets.
- [publication-policy.md](../shared/publication-policy.md) — checkpoint resolution, manifest safety, first-run preference prompt, and publication disclosure.
- [events.md](../shared/events.md) — event-envelope contract and emission transport for `recap.completed`.
- [sidekick-capture.md](../shared/sidekick-capture.md) — sidekick-idea handling during the skill's flow.
- [PRINCIPLES.md](../../PRINCIPLES.md) — repo-level principles (user-attention economy, batched questions, parallel work while user is idle).
