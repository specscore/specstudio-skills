---
name: verify
description: |
  Consumes an approved SpecScore Feature and produces a machine-checkable
  per-AC verdict report at spec/features/<feature-slug>/_verify/<sha>.md.
  Walks `git log` for the `Verifies:` commit-message trailer per AC,
  dispatches one built-in AI subagent per mapped AC (serial, not parallel),
  aggregates verdicts into a Markdown report opened by a grep-friendly YAML
  summary block, emits `verify.completed`, and transitions only to
  `specstudio:recap`. Applies publication policy to the report checkpoint.
  Trigger: "verify", "/verify", "verify this feature", "specstudio:verify",
  or the explicit downstream transition from `specstudio:implement`.
aliases: [verify]
---

# Verify

Turn an approved SpecScore Feature plus its `Verifies:` commit trailers into a per-AC verdict report. Close the SpecStudio pipeline's verify slot so `recap` has an honest gate to consume.

## Hard Gate

<HARD-GATE>
Do NOT invoke `specstudio:ideate`, `specstudio:specify`, `specstudio:plan`, `specstudio:implement`, `specstudio:review`, `specstudio:ship`, `writing-plans`, `frontend-design`, `mcp-builder`, or ANY other skill until ALL of the following are true:
  1. Pre-flight passed: the input slug resolves to `spec/features/<feature-slug>/README.md`, the Feature's `**Status:**` is in `{Approved, Implementing, Stable}`, and the Feature exists at git HEAD (`git cat-file -e HEAD:spec/features/<feature-slug>/README.md` exits zero).
  2. The Feature parsed cleanly via the `specscore` CLI's Feature parser; the ordered AC ID list is available.
  3. The report has been written to `spec/features/<feature-slug>/_verify/<sha>.md` with a fenced ` ```yaml ` summary block followed by one `## AC: <ac-slug>` body section per AC.
  4. The `_verify/README.md` index has been created (if absent) or updated with the current run's row (if present), per `## Report Format → Index README`.
  5. Both the report and the `_verify/README.md` index are in the publication manifest and the `verify.completed` publication checkpoint has been resolved, disclosed, and applied.
  6. The `verify.completed` event has been emitted exactly once.

The only skill invoked after `specstudio:verify` is `specstudio:recap` (or — while `recap` is unshipped — a hand-back to the user with the report path and a recommendation to review verdicts manually).
</HARD-GATE>

## When to Use

- An approved Feature at `spec/features/<feature-slug>/README.md` exists in git HEAD and the user wants a per-AC verdict on the work already committed.
- `specstudio:implement` has just transitioned via its `transition-to-verify` REQ and the consolidated batch has been committed.
- The user wants to re-verify a Feature after additional `Verifies:`-trailed commits have landed.

**Refuse and redirect when:**

- The Feature's `**Status:**` is `Draft` or `Under Review` → print the current Status and recommend `specstudio:specify` to re-approve. Write no report. Exit non-zero. (AC: `refuses-draft-feature`)
- The Feature exists only in the working tree (uncommitted) → instruct the user to commit the Feature first. Write no report. Exit non-zero. (AC: `refuses-uncommitted-feature`)
- The user asks the skill to bypass publication policy, commit unrelated paths, or push without branch-policy approval → refuse.
- The user asks the skill to invoke `ship`, `review`, or any non-`recap` skill → refuse; the only permitted downstream transition is `specstudio:recap`.

## Pre-Flight

1. **Resolve input.** The skill MUST accept exactly one positional argument — the Feature slug — and resolve it to `spec/features/<feature-slug>/README.md`.
2. **Status guard.** Read the Feature's body-metadata `**Status:**` line. Refuse if it is outside `{Approved, Implementing, Stable}`. On refusal, print the current Status, recommend `specstudio:specify` to re-approve, write no report, and exit non-zero.
3. **HEAD-existence guard.** Run `git cat-file -e HEAD:spec/features/<feature-slug>/README.md`. On non-zero exit, refuse with an instruction to commit the Feature first, write no report, and exit non-zero. This mirrors `implement`'s pre-flight discipline — the `Verifies:` trailer convention requires the Feature to exist in git history.
4. **Resolve HEAD SHA.** Capture `git rev-parse --short HEAD` once. This SHA is the `<sha>` in the report path and the `revision` in the YAML summary and the event payload.

## Checklist

Create a task for each and complete in order:

1. **Pre-flight** (steps above). Refusals exit immediately; do not proceed.
2. **Parse the Feature.** Delegate to the `specscore` CLI's Feature parser. Surface the ordered list of AC IDs in the form `<feature-slug>#ac:<ac-slug>` paired with each AC's full `Given / When / Then` text. Parse failures stop the skill with the CLI's lint-rule citation surfaced verbatim to the user.
3. **Collect commits per AC.** For each AC ID in Feature order, run:

   ```
   git log --extended-regexp --grep='^Verifies:.*<feature-slug>#ac:<ac-slug>' --format='%H%n%s%n%b%x00'
   ```

   Collect the matching commit SHAs in chronological (oldest-first) order paired with their commit messages (subject + body). Do NOT pre-fetch commit diffs — the subagent fetches diffs on demand. An AC with zero matching commits is recorded as `unmapped` and skipped for subagent dispatch.
4. **Dispatch verifier subagents serially.** For each mapped AC (commit count ≥ 1) in Feature AC order, dispatch one verifier subagent via the Agent tool with `subagent_type: general-purpose`. Construct the prompt per `## Verifier Subagent Contract` below. Wait for the subagent to return a parseable verdict before dispatching the next AC's subagent. At no point during the run MUST more than one verifier subagent be concurrently in flight.
5. **Parse the verdict.** Validate per `## Verifier Subagent Contract`. On a malformed first response, re-dispatch the same subagent exactly once with a corrective prompt that quotes the failure (verdict outside `{pass, fail, error}`, missing justification, or justification exceeding 400 characters) and reiterates the verdict contract. On a malformed second response, record the AC's verdict as `error` and MUST NOT call the subagent a third time.
6. **Tally counts.** Compute `passed_count`, `failed_count`, `unmapped_count`, `errored_count` over the full AC list. Each is a non-negative integer; their sum equals the Feature's total AC count.
7. **Write the report.** Create `spec/features/<feature-slug>/_verify/` if absent. Write the report file to `spec/features/<feature-slug>/_verify/<sha>.md` per `## Report Format` below.
8. **Write or update the `_verify/README.md` index.** Per `## Report Format → Index README` below. If absent, create it with one row in `## Contents`. If present, append one row for the current run, preserving prior rows. Without this step the project's `readme-exists` lint rule will fail on the newly created `_verify/` directory.
9. **Publication checkpoint.** Add `spec/features/<feature-slug>/_verify/<sha>.md` and `spec/features/<feature-slug>/_verify/README.md` to the manifest. Apply [publication-policy.md](../shared/publication-policy.md) for `verify.completed`, staging only manifest paths and committing/pushing only when policy and safety allow.
10. **Emit `verify.completed`.** Per `## Event Emission` below, including `publication_result`. Exactly once per successful run.
11. **Transition.** Per `## Promotion Boundary` below. Only `specstudio:recap` (or hand-back when `recap` is unshipped).
12. **Determine exit code.** Per `## Exit Semantics` below. Non-zero iff `failed_count + errored_count > 0`. Otherwise zero.
13. **Throughout** — watch for sidekick ideas (e.g., a flaky subagent shape, a deferred AC-coverage gap surfacing during verification). When an out-of-scope improvement surfaces, invoke `specstudio:sidekick` with a one-liner, acknowledge in one line, and return to the current checklist step immediately. Do not derail.

## Verifier Subagent Contract

Each verifier subagent is dispatched with an **isolated prompt** — it MUST NOT inherit the parent session's context. Construct the prompt freshly per AC.

### Prompt shape (four required parts, in this order)

1. **AC identification and full text.** The AC ID (`<feature-slug>#ac:<ac-slug>`) followed by its complete `Given / When / Then` text quoted verbatim from the source Feature.
2. **Matching commits.** The ordered list of commit SHAs (chronological, oldest-first) paired with each commit's full message (subject + body). Format:

   ```
   ## Matching commits

   - <sha-1>
     <commit message subject + body>
   - <sha-2>
     <commit message subject + body>
   ```

   The orchestrator MUST NOT include pre-fetched commit diffs in the prompt. The subagent fetches diffs and reads source files on demand via its own Bash tool.
3. **Verdict contract (verbatim).** Quote the following block into the prompt verbatim, so the subagent knows the required output shape:

   ```
   Return exactly one verdict from the set: {pass, fail, error}.

   Required output shape:

     verdict: <pass | fail | error>
     justification: <one-line snippet, MAXIMUM 400 characters>
     evidence:
       - <file path, commit SHA, or reference>
       - ...

   - `pass`   = the matching commits satisfy the AC's `Then` clause.
   - `fail`   = the matching commits do NOT satisfy the AC's `Then` clause.
   - `error`  = you cannot reach a verdict (e.g., a commit is unreachable, a referenced
                file is missing, the AC text is ambiguous in a way that no diff can resolve).

   The value `unmapped` is produced only by the orchestrator for ACs with zero matching
   commits and MUST NOT appear in your response.

   Malformed responses (verdict outside the allowed set, missing justification, or
   justification exceeding 400 characters) will be re-dispatched exactly once with a
   corrective prompt; a second malformed response is recorded as `error`.
   ```

4. **Fetch-on-demand instruction.** An explicit instruction that the subagent fetches commit diffs and reads source files on its own via Bash. Suggested commands to mention:

   ```
   git show <sha>
   git show <sha> -- <path>
   git show <sha> --stat
   cat <path>
   ```

   Include the guidance: "Read at the depth required to reach a verdict; do not enumerate every file in every commit if the verdict is obvious from one diff."

### Out of scope for the prompt

- **No pre-fetched diffs.** The orchestrator MUST NOT inline `git show` output into the prompt. Diff fetching is the subagent's responsibility.
- **No `_tests/` reference.** The semantic role of `_tests/<ac-slug>.md` Markdown scenarios is deliberately deferred to a future Idea (captured as a sidekick seed during the Feature's specify session). The prompt MUST NOT instruct the subagent to read `_tests/` files. The subagent MAY discover them via Bash if it chooses, but no instruction directs it there.
- **No `unmapped` verdict.** `unmapped` is orchestrator-produced. The allowed-verdict set named in the prompt is `{pass, fail, error}` ONLY.

### Verdict parsing

A well-formed response contains:

- A `verdict:` line whose value is exactly one of `pass`, `fail`, `error`.
- A `justification:` line whose value is non-empty and at most 400 characters.
- An `evidence:` list with zero or more entries (file paths, commit SHAs, or other references).

Anything outside this shape is malformed. Apply the retry-once-then-error semantics per checklist step 5.

## Report Format

### Path

```
spec/features/<feature-slug>/_verify/<sha>.md
```

`<sha>` is the abbreviated git SHA of HEAD at run time (`git rev-parse --short HEAD`). Create the `_verify/` directory if absent.

### YAML summary block (grep target)

The report MUST open with a fenced YAML block (delimited by ` ```yaml ` and ` ``` `) listing every AC in Feature AC order. This block is the grep target downstream skills (`recap`, eventually `ship`) consume.

```yaml
feature: <feature-slug>
revision: <sha>
verdicts:
  - ac: <feature-slug>#ac:<ac-slug-1>
    verdict: pass
    justification: "<one-line snippet, ≤400 chars>"
  - ac: <feature-slug>#ac:<ac-slug-2>
    verdict: unmapped
    justification: "no commits reference this AC"
  - ac: <feature-slug>#ac:<ac-slug-3>
    verdict: fail
    justification: "<one-line snippet>"
  - ac: <feature-slug>#ac:<ac-slug-4>
    verdict: error
    justification: "<one-line snippet>"
```

Every AC from the source Feature MUST appear exactly once, in Feature AC order. The four allowed `verdict` values are `pass`, `fail`, `error`, `unmapped`.

### Body (per-AC sections)

After the YAML block, the report MUST contain one `## AC: <ac-slug>` section per AC, in Feature AC order. Each section contains:

```markdown
## AC: <ac-slug>

**Verdict:** <pass | fail | error | unmapped>

**Justification:** <full justification text — same as the YAML one-liner if short, or
expanded for human readability if the subagent supplied a richer rationale>

**Commits:**
- <sha-1> — <commit subject>
- <sha-2> — <commit subject>

(Or: "No commits reference this AC." for `unmapped` ACs.)

**Evidence:**
- <file path, commit SHA, or other reference>
- ...
```

The YAML block is the machine-readable surface; the body is the human-readable surface.

### Index README

The skill MUST create `spec/features/<feature-slug>/_verify/README.md` if absent and MUST append a row for the current run to its `## Contents` table on every run. The README is the directory's index — without it, the project's `readme-exists` lint rule fails on the newly created `_verify/` directory.

**Shape (when creating from absent):**

```markdown
# Verify Reports — <feature-slug>

Per-run verify reports produced by `specstudio:verify`. Each report is named `<sha>.md` where `<sha>` is the abbreviated git SHA of `HEAD` at run time.

## Contents

| Report | Run revision | Verdict summary |
|---|---|---|
| [<sha>.md](<sha>.md) | <sha> | <one-line verdict summary, e.g., "5 passed, 0 failed, 0 unmapped, 0 errored" or "19 unmapped, 0 passed/failed/errored"> |

## Open Questions

None at this time.

---
*This document follows the https://specscore.md/index-specification*
```

**Append rule (when README already exists):**

Locate the `## Contents` table. Insert a new row at the bottom of the table (newest-last). Preserve every prior row in its existing order — never reorder, never delete. The new row's `Report` cell is a Markdown link `[<sha>.md](<sha>.md)`; the `Run revision` cell is the abbreviated SHA; the `Verdict summary` cell is a one-line tally derived from the four flat counts.

If the table is missing entirely (e.g., README was hand-edited), the skill MUST refuse to write the row, surface the missing-table error to the user, and recommend manual repair before re-running.

### Publication discipline

After writing the report and the index README, the skill MUST add both to the checkpoint manifest and apply [publication-policy.md](../shared/publication-policy.md) for `verify.completed`. If the allowed actions include `stage`, stage only those manifest paths. If they include `commit` or `push`, run the shared unrelated-index and branch-safety checks first. The disclosure must name the resolved policy, executed actions, skipped actions, and manifest paths.

### No-commits edge case

When a Feature has zero `Verifies:` trailers in the entire branch history (e.g., a Feature that was specified but not yet implemented), the skill MUST still:

- Write the report at the canonical path.
- Mark every AC `unmapped` in both the YAML block and the body sections.
- Apply publication policy to the report and index manifest.
- Emit `verify.completed` with `unmapped_count` equal to the Feature's total AC count, the other three counts equal to zero, and `publication_result`.
- Exit zero.

The report itself communicates that nothing has been implemented yet; the absence of a report is not the right signal.

## Exit Semantics

The skill MUST exit non-zero **if and only if** `failed_count + errored_count > 0`. Equivalently:

| Tally | Exit code |
|---|---|
| Every AC `pass` or `unmapped` (any mix) | `0` |
| At least one AC `fail` | non-zero |
| At least one AC `error` | non-zero |
| At least one `fail` AND at least one `error` | non-zero |
| Every AC `unmapped` (no-commits edge case) | `0` |

`unmapped` is **informational** at the verify layer. `ship` is the eventual gate that escalates `unmapped` to blocking; `verify` does not.

A pre-flight refusal (Draft Feature, uncommitted Feature) exits non-zero without writing a report — those refusals are distinct from the verdict-tally exit code above.

## Event Emission

After the report is written and publication policy has been applied (and only on a successful run — pre-flight refusals do NOT emit), the skill MUST emit exactly one `verify.completed` event via the convention in [events.md](../shared/events.md). Use `specscore event emit <event.yaml>` when the CLI is available; fall back to appending the event JSONL line to `.specscore/events.jsonl`.

Payload shape (flat counts — NOT a nested `verdict_counts` object):

```yaml
event: verify.completed
version: 1
uuid: <generated>
timestamp: <ISO-8601>
actor:
  kind: skill
  id: skill:specstudio:verify
artifact:
  type: feature
  id: <feature-slug>
  path: spec/features/<feature-slug>/README.md
  revision: <sha>
payload:
  feature_slug: <feature-slug>
  revision: <sha>
  report_path: spec/features/<feature-slug>/_verify/<sha>.md
  passed_count: <int ≥ 0>
  failed_count: <int ≥ 0>
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

- All four count fields are non-negative integers.
- `passed_count + failed_count + unmapped_count + errored_count` equals the Feature's total AC count.
- The payload MUST NOT include per-AC verdict details. Consumers read those from the report file at `report_path`.
- The event MUST be emitted exactly once per successful run.
- Additional payload fields MAY be added in the future without breaking this contract; the four count fields plus `feature_slug`, `revision`, and `report_path` are the minimum.

## Promotion Boundary

The next skill is `specstudio:recap`, and only `specstudio:recap`.

### Transition

After the report is written, publication policy is applied, and `verify.completed` is emitted:

- If `specstudio:recap` is shipped: offer to transition the user to `specstudio:recap`, passing the `report_path` as input.
- If `specstudio:recap` is unshipped: hand back to the user with the report path and a recommendation to review verdicts manually:

  > "Verify complete. Report written at `spec/features/<feature-slug>/_verify/<sha>.md` and publication policy applied. The `specstudio:recap` skill is not yet shipped — review the YAML summary block in the report for per-AC verdicts."

The skill MUST NOT invoke `ideate`, `specify`, `plan`, `implement`, `review`, `ship`, `writing-plans`, `frontend-design`, or `mcp-builder` on transition. The hard gate above pins this; the transition step honors it.

## Tone

The skill MUST NOT yes-machine soft verdicts. When a subagent returns `pass` on a commit set that obviously does not satisfy the AC, the skill MUST trust the subagent's verdict for THIS run (the orchestrator does not re-judge), but MUST surface the report honestly — the YAML block and the body section carry the verdict as returned. Re-judgment is `recap`'s job, not `verify`'s.

When the skill detects a pre-flight refusal (Draft Feature, uncommitted Feature), it MUST say so with the specific cited cause (current Status value, missing-from-HEAD path) and propose the alternative (`specstudio:specify`, `git commit`). Honest specificity, not performative agreement.

When the subagent returns malformed responses twice in a row, the skill records `error` and moves on — it MUST NOT loop infinitely.

## Verification

- [ ] Pre-flight checks passed: input slug resolves; Feature `**Status:**` ∈ {Approved, Implementing, Stable}; Feature exists at git HEAD
- [ ] Feature parsed cleanly via `specscore` CLI; ordered AC list captured
- [ ] For each AC: `git log --grep='^Verifies:.*<feature-slug>#ac:<ac-slug>'` ran; matched SHAs + commit messages collected; NO diffs pre-fetched
- [ ] Subagent dispatch was serial: at no point were two verifier subagents concurrently in flight
- [ ] Each per-AC subagent prompt contained: full G/W/T text, commit SHAs + messages, verbatim verdict contract, and the fetch-on-demand Bash instruction; no pre-fetched diffs
- [ ] Malformed verdicts were retried exactly once; second-time malformed verdicts were recorded as `error`
- [ ] Report written at `spec/features/<feature-slug>/_verify/<sha>.md` with the fenced YAML block first, then `## AC:` body sections
- [ ] Every AC appears in the YAML block in Feature AC order with one of `{pass, fail, error, unmapped}`
- [ ] `_verify/README.md` index created (if absent) or row-appended (if present); table preserves prior rows in their existing order
- [ ] Both report and `_verify/README.md` added to the checkpoint manifest; publication policy applied with disclosure
- [ ] `specscore spec lint` exits zero after the run (no `readme-exists` failure on the `_verify/` directory)
- [ ] `verify.completed` event emitted exactly once; payload counts sum to the Feature's total AC count; no per-AC details in the payload; `publication_result` is present
- [ ] Exit code is non-zero iff `failed_count + errored_count > 0`; otherwise zero
- [ ] No-commits edge case (every AC `unmapped`) still produced a complete report, applied publication policy, emitted the event, and exited zero
- [ ] On transition: only `specstudio:recap` invoked (or hand-back when unshipped); no other skill touched

## Red Flags

- Pre-fetching commit diffs into the subagent prompt (the subagent fetches diffs on demand; the orchestrator does not)
- Naming `unmapped` in the subagent's allowed-verdict set (it is orchestrator-produced; the prompt names `{pass, fail, error}` only)
- Instructing the subagent to read `_tests/<ac-slug>.md` files (deferred to a future Idea; not in MVP)
- Dispatching two verifier subagents concurrently (serial only; parallelism is explicitly deferred per the source Idea's `## Not Doing`)
- Looping the malformed-verdict retry more than once (one retry; second malformed → `error`; never a third call)
- Allowing a justification longer than 400 characters to pass the verdict-contract check
- Hard-coding stage-only report handoff instead of resolving publication policy for `verify.completed`
- Committing unrelated staged paths or pushing without branch-policy approval
- Writing the report to `docs/` or to a path other than `spec/features/<feature-slug>/_verify/<sha>.md`
- Treating `unmapped` as a non-zero-exit signal (it is informational at the verify layer; `ship` escalates it later)
- Skipping the `verify.completed` event because "no failures" — the event fires on every successful run regardless of verdict outcomes
- Emitting `verify.completed` with per-AC verdict details in the payload (per-AC data lives in the report file; the event payload carries flat counts only)
- Nesting the four counts under a `verdict_counts:` key (the contract is flat: `passed_count`, `failed_count`, `unmapped_count`, `errored_count`)
- Invoking any skill other than `specstudio:recap` on transition (no `review`, no `ship`, no `writing-plans`, no `frontend-design`, no `mcp-builder`)
- Refusing to write a report when no commits match (the no-commits edge case still produces a complete `unmapped`-only report)
- Writing the per-run report but forgetting `_verify/README.md` (the `readme-exists` lint rule will fail on the newly created `_verify/` directory)
- Reordering or deleting prior rows in `_verify/README.md`'s `## Contents` table (the skill only appends; prior runs' rows are preserved verbatim)

## References

- [Feature: Verify Skill](../../spec/features/skills/verify/README.md) — the SpecScore Feature this skill implements.
- [Feature: Implement Skill](../../spec/features/skills/implement/README.md) — the upstream Feature whose `Verifies:` commit-message trailer this skill consumes.
- [Plan: Verify Skill MVP](../../spec/plans/verify.md) — the seven-task plan this skill realizes.
- [philosophy.md](../shared/philosophy.md) — shared tenets.
- [publication-policy.md](../shared/publication-policy.md) — checkpoint resolution, manifest safety, first-run preference prompt, and publication disclosure.
- [events.md](../shared/events.md) — event-envelope contract and emission transport for `verify.completed`.
- [sidekick-capture.md](../shared/sidekick-capture.md) — sidekick-idea handling during the skill's flow.
- [PRINCIPLES.md](../../PRINCIPLES.md) — repo-level principles (user-attention economy, batched questions, parallel work while user is idle).
