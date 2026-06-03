# Reviewer-Gates Gate Runner

**Status:** Contract — shared instructional document for skills that execute a reviewer gate after loading its configuration via [`loader.md`](./loader.md).
**Owned by:** [reviewer-gates](../../../spec/features/reviewer-gates/README.md) Feature.

## Purpose

This file tells a consuming skill (currently `specstudio:specify`; future consumers MUST follow the same protocol) how to **execute** a reviewer gate against the validated reviewer list produced by [`loader.md`](./loader.md). The runner is the second half of the reviewer-gates contract: where the loader does load-time validation, the runner does dispatch, grade aggregation, the two-phase (automated then human) verdict computation, and rerun-policy enforcement.

The runner's output is the gate's **grade** (a whole letter `A`–`F`) and the derived verdict for the consuming skill:

- `Approved` — the computed grade satisfies `grade ≥ threshold`. The consumer MAY release whatever downstream gate it owns (e.g., for `specstudio:specify`, the User Review Gate is now released and the skill MAY emit `feature.approved`).
- `Issues Found` — `grade < threshold`. The consumer MUST surface the failing grade and the `Blocker` findings to the user, MUST NOT release any downstream gate, and MUST NOT emit any approval event. Re-running the gate after the user's fix is governed by Step 5 (Rerun policy) below.

There is no third verdict. The runner does not "skip" reviewers, does not silently downgrade `Blocker` severity to `Advisory`, does not inject any reviewer the loader did not validate, and does not write to `spec/`.

This runner implements the following REQs from the [reviewer-gates Feature](../../../spec/features/reviewer-gates/README.md):

- [`deterministic-entry-shape`](../../../spec/features/reviewer-gates/README.md#req-deterministic-entry-shape) — a `type: deterministic` entry's verdict is derived from its `run:` command's exit code (zero → `Approved`, no findings; non-zero → `Issues Found`, diagnostics captured as `Blocker`(s)).
- [`auto-approve-entry-shape`](../../../spec/features/reviewer-gates/README.md#req-auto-approve-entry-shape) — a `type: auto-approve` entry dispatches nothing and always returns `Approved` with no findings.
- [`gate-point-events-and-multi-fire`](../../../spec/features/reviewer-gates/README.md#req-gate-point-events-and-multi-fire) — a gate keyed on a multi-occurrence gate-point event (e.g., `implementation.pre_commit`) is evaluated independently at each occurrence, with no single-shot-per-run caching.
- [`dispatch-serial`](../../../spec/features/reviewer-gates/README.md#req-dispatch-serial) — one reviewer in flight at a time, in list order.
- [`verdict-contract`](../../../spec/features/reviewer-gates/README.md#req-verdict-contract) — `Blocker`/`Advisory` severity on findings; no reviewer writes to `spec/`.
- [`and-composition`](../../../spec/features/reviewer-gates/README.md#req-and-composition) — no early-halt; two-phase dispatch (all `type: ai` reviewers, then `type: human` only if the automated grade ≥ threshold).
- [`grade-band-mapping`](../../../spec/features/reviewer-gates/README.md#req-grade-band-mapping), [`grade-aggregation`](../../../spec/features/reviewer-gates/README.md#req-grade-aggregation), [`threshold-derived-verdict`](../../../spec/features/reviewer-gates/README.md#req-threshold-derived-verdict) — compute the grade from the `Blocker` union + within-band letter (Step 2.8) and release iff `grade ≥ threshold`.
- [`rerun-policy`](../../../spec/features/reviewer-gates/README.md#req-rerun-policy) — re-dispatch previously-`Issues Found` reviewers on rerun; ALSO re-dispatch previously-`Approved` reviewers when the fix touched a structural section.
- [`gate-entry-when-condition`](../../../spec/features/reviewer-gates/README.md#req-gate-entry-when-condition) — an entry carrying a `when: "branch =~ <anchored-regex>"` condition participates only if the current branch matches; an entry with no `when:` always participates. A masked-out entry is neither dispatched nor counted toward the verdict.

## Inputs

| Input | Source | Required |
|---|---|---|
| Validated reviewer list | The ordered list returned by [`loader.md`](./loader.md) Step 4. The consumer MUST NOT modify this list before passing it here. | Yes |
| Calling skill's Agent-tool dispatch capability | For `type: ai` entries. The consumer's host (Claude with the skill's runtime context) is the dispatcher. | Yes |
| Calling skill's approval-phrase recognizer | For `type: human` entries. The same recognizer used by `specstudio:ideate` and `specstudio:specify` for user-approval gates (`approve` / `approved` / `accept` / `accepted` / `lgtm` / direct semantic equivalents → `Approved`; explicit change requests → `Issues Found` with the user's text captured as a single `Blocker` finding). | Yes |
| Resolved Approve threshold | The whole-letter threshold (`A`–`F`, default `B`) returned by [`loader.md`](./loader.md) Step 2.5. The gate releases iff `grade ≥ threshold`. | Yes |
| Current branch name | The name of the branch the run is on (e.g., `main`, `feature/x`). Used only to evaluate entries that carry a `when:` branch condition (Step 1.5). When no entry in the list carries a `when:`, this input is unused. | Yes (when any entry carries `when:`) |
| Previous-pass verdict map (rerun only) | A map of `<reviewer-name> → <last-verdict>` from the previous pass, used only when this is a rerun. On the first pass the map is empty. | Yes (may be empty) |
| Structural-fix flag (rerun only) | Boolean. `true` when the user's fix between passes touched a structural section of the artifact under review (see Step 5 for what counts as structural per artifact type). On the first pass, this flag is unused. | Yes (may be `false`) |

## Output

The runner returns the **gate grade** (a whole letter `A`–`F`, computed per Step 2.8) and the derived gate verdict, exactly one of:

- `Approved` — the grade satisfies `grade ≥ threshold`. Accompanied by the grade and the per-reviewer verdict map for this pass. The consumer is now released to proceed.
- `Issues Found` — `grade < threshold`. Accompanied by (a) the grade, (b) the surfaced reviewer(s) and their structured findings list (each finding carrying `Blocker` or `Advisory` severity and a prose description), and (c) the per-reviewer verdict map for this pass (a `type: human` reviewer not dispatched because the automated grade was below threshold is absent from the map). The consumer MUST surface at least the `Blocker` findings to the user and MUST NOT release any downstream gate or emit any approval event.

In both cases the runner returns the grade plus the per-reviewer verdict map so the consumer can pass the map back as the "previous-pass verdict map" input on the next pass (the rerun), and surface the grade (e.g., `specstudio:score` renders it; a producer-exit gate may log it).

## Protocol

Follow the steps in order. Do not skip ahead.

### Step 0 — Confirm inputs

Confirm the validated reviewer list is non-empty (the loader's Step 2 guarantees this; if the runner is invoked with an empty list, that is a consumer bug — refuse with `Error: gate runner invoked with empty reviewer list; consumer must invoke loader first and pass its output`).

Confirm the previous-pass verdict map keys are a subset of the current reviewer list's `name` values. Stray keys indicate the consumer rebuilt the registry between passes — refuse with `Error: previous-pass verdict map contains reviewers not in the current registry; reviewer-list mutations between passes are not supported`. (A consumer that genuinely changed `specscore.yaml` between passes MUST treat the next invocation as a first pass — empty previous-pass map.)

### Step 1 — Compute the dispatch set for this pass

Walk the validated reviewer list in declared order. For each entry, decide whether it MUST be dispatched in this pass:

- **First pass** (previous-pass verdict map is empty): every entry MUST be dispatched. The dispatch set equals the full reviewer list, in declared order.
- **Rerun pass** (previous-pass verdict map is non-empty): for each entry, the dispatch decision is:
  - If the entry's previous verdict was `Issues Found`: dispatch it. (Per [`rerun-policy`](../../../spec/features/reviewer-gates/README.md#req-rerun-policy).)
  - If the entry's previous verdict was `Approved` **and** the structural-fix flag is `true`: dispatch it. (Per [`rerun-policy`](../../../spec/features/reviewer-gates/README.md#req-rerun-policy).)
  - If the entry's previous verdict was `Approved` **and** the structural-fix flag is `false`: the entry MAY be skipped — carry its previous `Approved` verdict forward into this pass's verdict map. (Per [`rerun-policy`](../../../spec/features/reviewer-gates/README.md#req-rerun-policy)'s "non-structural fixes" allowance.)
  - If the entry is absent from the previous-pass verdict map (e.g., a `type: human` entry deferred because the previous pass's automated grade was below threshold): dispatch it in its phase. The previous pass never collected its verdict; the rerun MUST collect one (a `type: human` entry still runs only if this pass's automated grade ≥ threshold, per Step 2.8).

The order within the dispatch set is exactly the declared order of the original reviewer list — entries skipped above do not shift the order of the remaining entries.

### Step 1.5 — Apply each entry's `when:` branch condition

Before evaluating any entry, mask the dispatch set by each entry's optional `when:` branch condition, per [`gate-entry-when-condition`](../../../spec/features/reviewer-gates/README.md#req-gate-entry-when-condition). For each entry in the dispatch set computed by Step 1:

- If the entry carries **no `when:`** field (the normalized record from [`loader.md`](./loader.md) Step 3e has no `when`): the entry **always participates** — leave it in the dispatch set unchanged.
- If the entry carries a **`when:`** condition (a string of the validated form `branch =~ <anchored-regex>`): resolve the **current branch name** (the runner input above) and evaluate the right-hand-side regex against it, **anchored** (the regex matches the full branch name; e.g., `^(main|master|release/)` matches `main`, `master`, and any branch under `release/`, but not `feature/x`). There is no glob interpretation — the grammar is anchored regex, identical to the dialect the loader validated (so the two layers never disagree).
  - **Match** → the entry **participates**: keep it in the dispatch set in its declared position.
  - **No match** → the entry **does NOT participate**: remove it from the dispatch set for this pass. It is **neither dispatched nor counted toward the verdict** — it contributes no `Blocker` (and no `Approved`/`Issues Found` entry to the verdict map), exactly as if it were absent from the list on this branch. A `when:`-masked `type: human` entry that does not match therefore means the human is not asked, and the gate can release on its remaining (matching) entries alone.

Masking does not reorder the surviving entries — their declared order is preserved into Step 2. The resulting `when:`-masked dispatch set is what Steps 2/2-det/2-auto-approve/2.9 evaluate. This is the home for per-branch autonomy masks (e.g., a `type: human` entry on `implementation.pre_commit` with `when: "branch =~ ^(main|master|release/)"` is asked before commit on `main` but not on a feature branch — the per-branch behavior comes entirely from the gate-entry `when:` condition).

### Step 2 — Automated phase: evaluate every automated reviewer (serial, no early-halt)

The **automated phase** evaluates every non-`human` reviewer — `type: ai`, `type: deterministic`, and `type: auto-approve` — in declared order; `type: human` entries are deferred to the human phase (Step 2.9). Each automated reviewer contributes its verdict and findings to the `Blocker` union (Step 2.8). There is no early-halt: every automated entry in the dispatch set is evaluated, so the union — and hence the grade — is exact in a single pass.

For each **`type: ai`** entry in the dispatch set, in declared order:

1. **Record dispatch start.** Capture the timestamp at the moment dispatch begins (for AC `serial-dispatch-observed`'s instrumentation contract — see Step 6 below).
2. **Dispatch the AI reviewer.** Invoke the consumer skill's Agent tool with the entry's resolved `prompt_path` file contents as the system prompt. Pass the artifact under review (e.g., the Feature `README.md` text for `specstudio:specify`) as the user-facing message. Wait for the subagent to return. The subagent MUST respond with exactly one of `Approved` or `Issues Found`; on `Issues Found`, the response carries a structured findings list per the prompt's documented blocker/advisory taxonomy (and, if the prompt declares the multi-role lenses, a single within-band letter).
3. **Record dispatch end.** Capture the timestamp at the moment the verdict is collected. The recorded `[start, end]` interval for this entry MUST be disjoint from every other entry's interval in this pass — no two dispatches concurrently in flight. (See Step 6 for the instrumentation contract and the test-harness assertion shape.)
4. **Validate the verdict shape.** The returned verdict MUST be exactly `Approved` or `Issues Found`. If a `type: ai` subagent returns anything else (e.g., omits the verdict, returns prose with no decision), the runner MUST refuse the gate with `Error: reviewer '<name>' returned a non-conforming verdict (expected 'Approved' or 'Issues Found'); reviewer prompts MUST conform to the documented taxonomy per reviewer-gates#req:verdict-contract`. No silent retry, no inference. This is a reviewer-prompt bug surfaced to the user.
5. **Reject writes to `spec/`.** If a reviewer attempted (or its findings instruct the consumer to perform) a write or modification to any artifact under `spec/`, the runner MUST refuse the gate with `Error: reviewer '<name>' is a misclassified Producer; reviewers MUST NOT write to spec/ per reviewer-gates#req:verdict-contract`. (Reviewers are read-only; producers own artifact writes.)
6. **Record the verdict, findings, and within-band letter.** Record `<entry.name> → <verdict>` in this pass's verdict map, and retain the entry's structured findings (each with `Blocker`/`Advisory` severity). For a `type: ai` entry whose prompt declares the multi-role lenses, also retain the single within-band letter it reported (per [`multi-role-reviewer`](../../../spec/features/reviewer-gates/README.md#req-multi-role-reviewer)); a findings-only reviewer reports no within-band letter (treated as the default `B` in Step 2.8).
7. **No early-halt.** Record the verdict and findings and continue to the next automated entry. The runner does NOT halt on `Issues Found` (per [`and-composition`](../../../spec/features/reviewer-gates/README.md#req-and-composition)): every automated entry in the dispatch set is evaluated, so the `Blocker` union — and hence the grade — is exact and every `Blocker` surfaces in a single pass. When the automated entries are exhausted, proceed to Step 2.8.

### Step 2-det — `type: deterministic` entries (verdict from exit code)

For each **`type: deterministic`** entry in the dispatch set, in its declared position in the automated phase (serial per `dispatch-serial`, no early-halt — same loop as Step 2), derive the verdict **deterministically from the command's exit code** per [`deterministic-entry-shape`](../../../spec/features/reviewer-gates/README.md#req-deterministic-entry-shape):

1. **Record dispatch start.** Capture the timestamp at the moment execution begins (same instrumentation as Step 2, for the serial-dispatch contract).
2. **Run the command.** Execute the entry's `run:` command in the repo working tree. Capture its exit code and its combined diagnostic output (stdout + stderr). The command is read-only by contract — a `type: deterministic` check that mutates artifacts under `spec/` is a misclassified Producer and the runner MUST refuse the gate with `Error: reviewer '<name>' is a misclassified Producer; reviewers MUST NOT write to spec/ per reviewer-gates#req:verdict-contract`.
3. **Record dispatch end.** Capture the timestamp at the moment the exit code is collected. The `[start, end]` interval MUST be disjoint from every other entry's interval in this pass (no concurrent in-flight dispatches).
4. **Map exit code → verdict.**
   - **Exit code zero → `Approved`** with **no findings** (contributes nothing to the `Blocker` union).
   - **Non-zero exit code → `Issues Found`**, with the command's captured diagnostic output recorded as **at least one `Blocker` finding** (severity `Blocker`, the diagnostic output as the finding's prose description). A configurable non-exit-code success predicate is out of MVP scope — the exit code is the sole signal.
5. **Record the verdict and findings** in this pass's verdict map (`<entry.name> → <verdict>`). A `type: deterministic` entry never reports a within-band letter (it is findings-only; treated as the default `B` in Step 2.8 when it contributes zero `Blocker`s).
6. **No early-halt.** Continue to the next automated entry.

At the default threshold `B`, a non-zero exit (≥ 1 `Blocker`) yields grade ≤ `C` < `B`, so the gate does NOT release; a zero exit contributes no `Blocker` and the gate releases iff the rest of the union is also zero-`Blocker`.

### Step 2-auto-approve — `type: auto-approve` entries (always approve, dispatch nothing)

For each **`type: auto-approve`** entry in the dispatch set, in its declared position in the automated phase, the runner MUST:

1. **Dispatch nothing.** Do NOT invoke the Agent tool, do NOT run any command, do NOT prompt the human — a `auto-approve` entry has no reviewer to dispatch (per [`auto-approve-entry-shape`](../../../spec/features/reviewer-gates/README.md#req-auto-approve-entry-shape)).
2. **Record `Approved` with no findings.** Record `<entry.name> → Approved` in the verdict map with an empty findings list. A `auto-approve` entry contributes **no `Blocker`** (and no `Advisory`) to the grade, and reports no within-band letter (treated as the default `B` in Step 2.8).

A `auto-approve` is the explicit auto-approve placeholder: it lets a gate be configured as "no review at this checkpoint" without removing the gate key (e.g., a `auto-approve` where a `human` would otherwise sit). Because it dispatches nothing and contributes no `Blocker`, it can never lower the grade.

### Step 2.8 — Compute the automated grade and route

After the automated phase (Step 2, Step 2-det, Step 2-auto-approve) has evaluated every automated entry (`type: ai`, `type: deterministic`, `type: auto-approve`), compute the **automated grade** from their findings, per [`grade-band-mapping`](../../../spec/features/reviewer-gates/README.md#req-grade-band-mapping) and [`grade-aggregation`](../../../spec/features/reviewer-gates/README.md#req-grade-aggregation):

1. **Blocker union.** Form the union of all `Blocker` findings across every automated reviewer dispatched this pass — every `type: ai` reviewer (and, for a multi-role reviewer, every lens) and every `type: deterministic` reviewer (its non-zero-exit diagnostics). `type: auto-approve` entries contribute nothing. Let `n = |union|`. (Because no reviewer was halted, this union is exact.)
2. **Band by Blocker count.** `n == 0` → pass band (`A`/`B`); `n == 1` → `C`; `2 ≤ n ≤ 3` → `D`; `n ≥ 4` → `F`.
3. **Within-band letter (pass band only).** Across the reviewers that reported a within-band letter, take the **lowest** (since `A > B`, any reported `B` wins over `A`). A reviewer that reported no within-band letter (findings-only) contributes the default `B`. Result: the grade is `A` iff at least one reviewer reported `A` and none reported `B`; otherwise `B`.
4. **Route on the automated grade** (letters ordered `A > B > C > D > F`):
   - If the automated grade `< threshold`: the gate verdict is `Issues Found`. **Skip the human phase** — do NOT dispatch any `type: human` reviewer (a human MUST NOT be asked to approve an artifact the automated reviewers already failed) — and go to Step 4.
   - If the automated grade `≥ threshold`: the automated reviewers are satisfied; proceed to Step 2.9 for the human phase (final sign-off).

### Step 2.9 — Human phase: final sign-off (only when the automated grade passes)

Reached only when Step 2.8 routed here (automated grade `≥ threshold`). Dispatch each `type: human` entry in the dispatch set, in declared order, serially (per `dispatch-serial`). For each:

1. Record dispatch start/end timestamps (same instrumentation as Step 2).
2. Present the artifact under review to the user and invoke the consumer skill's approval-phrase recognizer (the same one used by `specstudio:ideate` / `specstudio:specify`):
   - Explicit approval phrase (`approve` / `approved` / `accept` / `accepted` / `lgtm` or direct semantic equivalents) → `Approved` (contributes no `Blocker`).
   - Explicit change request → `Issues Found`, captured as a single `Blocker` finding under the entry's `name:`.
   - Ambiguous response → ask the user to clarify; not yet a verdict.
3. Record the verdict (and any finding) in the verdict map.

After the human entries, **re-compute the grade** including any human-contributed `Blocker`s (Step 2.8 steps 1–3 over the full union), and derive the final verdict: `Approved` iff `grade ≥ threshold`, else `Issues Found`. A human change request adds a `Blocker`, so it can only lower the grade — a human confirms or blocks, never raises, the automated grade. Proceed to Step 3 (`Approved`) or Step 4 (`Issues Found`).

### Step 3 — Gate verdict is `Approved`

When the final grade (after the human phase, Step 2.9) satisfies `grade ≥ threshold`, return `Approved` with the computed grade and the per-reviewer verdict map. At the default threshold `B` this coincides exactly with "zero `Blocker`s across all `type: ai` reviewers, and the human approved."

### Step 4 — Gate verdict is `Issues Found`

When Step 2.8 derives `Issues Found` (`grade < threshold`), the gate did not release.

Surface to the user:

- The computed grade and the resolved threshold (e.g., "Grade `C`, threshold `B` — not released").
- Every reviewer whose findings drove the failing grade and their structured findings. Because there is no early-halt, surface every dispatched reviewer's `Blocker` findings so the author can fix them all in one pass. **At a minimum every `Blocker` finding contributing to the grade MUST be surfaced verbatim.** Advisory findings SHOULD also be surfaced (the consumer MAY format them under a clearly-labeled "Advisory" section), but they do not affect the verdict.
- A clear statement that the gate did NOT release and no downstream event (e.g., `feature.approved`) was emitted.
- When the human phase was skipped (automated grade below threshold), a clear statement that the `type: human` reviewer(s) were NOT dispatched because the automated reviewers must pass first — naming them by `name:` so the user understands they were not asked to approve a failing artifact.

Severity discipline: the runner MUST NOT downgrade a `Blocker` finding to `Advisory`, MUST NOT collapse multiple findings into a single message that obscures severity, and MUST NOT omit a `Blocker` finding from what is surfaced to the user.

Return `Issues Found` with the verdict map (containing only the reviewers reached in this pass) and the surfaced reviewer + findings payload.

### Step 5 — Rerun policy

This step is invoked **between passes**, not within a single pass. After the user has addressed the surfaced findings (typically by editing the artifact under review), the consumer invokes the runner again. The consumer is responsible for:

1. Reloading the artifact under review.
2. Determining whether the fix touched a structural section (see below). This is the "structural-fix flag" input on the next invocation.
3. Passing the previous pass's verdict map as the "previous-pass verdict map" input.
4. Re-invoking this runner from Step 0.

**Structural sections.** What counts as "structural" depends on the artifact type the consumer is gating:

- **SpecScore Feature artifacts** (the only MVP consumer surface — `specstudio:specify` gates Feature `README.md` files): the structural sections are `## Behavior`, `## Architecture`, and `## Acceptance Criteria`. A fix that adds, removes, or modifies content under any of these section headings sets the structural-fix flag to `true`. A fix that only touches `## Summary`, `## Problem`, body metadata, prose outside structural sections, or fixes typos/links/comments inside structural sections without changing the semantics MAY set the flag to `false`.

> **Per-artifact-type extension story.** The structural-section enumeration above is currently scoped to SpecScore **Feature** artifacts because the only MVP consumer (`specstudio:specify`) produces Features. Future consumers (`specstudio:plan` gating Plan artifacts, `specstudio:implement` gating implementation diffs, etc.) will need their own structural-section enumerations — for each one, the structurally-load-bearing sections defined by that artifact's specification become the trigger set for the structural-fix flag. The runner contract above (Step 1's branches keyed on the flag) is artifact-agnostic; only the consumer's flag-computation logic varies by artifact type. When a future consumer wires this runner, its skill documentation MUST name the structural sections it considers load-bearing for its artifact type, and this comment is the extension hook telling the consumer where the rule belongs.

**What the consumer MUST do on rerun:**

- Re-dispatch every reviewer whose previous-pass verdict was `Issues Found`. Always — regardless of the structural-fix flag.
- Re-dispatch every reviewer whose previous-pass verdict was `Approved`, **if and only if** the structural-fix flag is `true`.
- For previously-`Approved` reviewers when the flag is `false`: skipping is permitted per Step 1's non-structural-fix allowance; the previous `Approved` verdict carries forward into the new pass's verdict map without re-dispatch.
- For reviewers absent from the previous-pass verdict map (e.g., a `type: human` entry deferred because the previous automated grade failed): dispatch in its phase — the previous pass never collected its verdict (a `type: human` entry still runs only if this pass's automated grade ≥ threshold).

**What the consumer MUST NOT do:**

- Skip a previously-`Issues Found` reviewer on rerun. (`Issues Found` reviewers MUST always re-validate after the fix.)
- Carry a previously-`Issues Found` verdict forward without re-dispatch. (No "the user said they fixed it, so we'll assume `Approved`" — the reviewer's re-verdict is the source of truth.)
- Re-dispatch fewer entries than the policy requires. (The user's fix may have introduced new blockers in adjacent reviewers; structural fixes specifically widen the affected reviewer set.)

The dispatch order on rerun is still the declared order of the validated reviewer list, walked in Step 2 — entries the rerun skips per Step 1 simply do not contribute to the dispatch set; their declared position is otherwise unchanged.

### Step 6 — Instrumentation and test-harness contract for `serial-dispatch-observed`

To verify [AC: serial-dispatch-observed](../../../spec/features/reviewer-gates/README.md#ac-serial-dispatch-observed), the runner's Step 2 records dispatch start/end timestamps per entry. The corresponding test harness MUST be a **mocked Agent-tool spy that records dispatch start/end timestamps per reviewer entry**, configured as follows:

- The spy replaces the live Agent-tool dispatcher for `type: ai` entries and the live approval-phrase recognizer for `type: human` entries during the test pass.
- For each dispatched entry, the spy records `(name, type, start_timestamp, end_timestamp, verdict)` exactly once. Timestamps are monotonic (e.g., `process.hrtime.bigint()` or equivalent) — not wall-clock, to avoid clock-skew flakes.
- The spy returns a configurable verdict per entry (the test sets the return-verdict table up front).

The harness then asserts:

1. **No-overlap.** For every pair of recorded `[start, end]` intervals `(s_a, e_a)` and `(s_b, e_b)` where `a ≠ b`: `e_a ≤ s_b` OR `e_b ≤ s_a`. Equivalently: the intervals are disjoint. Any overlap fails the test — it indicates concurrent in-flight dispatches.
2. **Start order equals registry order.** Sort the recorded entries by `start_timestamp` ascending. The resulting `name` sequence MUST equal the input `reviewers` list's `name` sequence (restricted to the dispatch set for the pass). Any reorder fails the test.

This assertion shape rules out the looser "list-order-only" reading of the AC: it directly observes both temporal seriality (assertion 1) and registry-order start sequencing (assertion 2), matching the AC's literal "instrumentation that records dispatch start/end timestamps" language.

The Rehearse scenario stub at [`spec/features/reviewer-gates/_tests/serial-dispatch-observed.md`](../../../spec/features/reviewer-gates/_tests/serial-dispatch-observed.md) is the canonical place to author the Given/When/Then steps for this harness; the harness shape above is the implementation contract the scenario steps lower to.

### Step 7 — Per-occurrence evaluation for multi-fire gate-point events

A gate may be keyed on a **pre-action gate-point event** that occurs at execution checkpoints rather than at a once-per-artifact lifecycle transition. The MVP gate-point events are `implementation.pre_commit` (before each commit a producer makes during an `implement` run) and `implementation.pre_push` (before a publish/promote); both are registered in [`events.md`](../events.md) alongside the lifecycle events. (Wiring an actual `implement` run to *fire* these gate-points is a downstream Feature — the implement-autonomy layer — not this contract.)

A gate keyed on an event that occurs **multiple times** in one run (e.g., `implementation.pre_commit` firing once per commit/batch) MUST be evaluated **independently at each occurrence**, per [`gate-point-events-and-multi-fire`](../../../spec/features/reviewer-gates/README.md#req-gate-point-events-and-multi-fire):

- Each occurrence of the event is a **fresh gate run**: the consumer invokes this runner from Step 0 with an **empty previous-pass verdict map** (each occurrence is a first pass, not a rerun of a prior occurrence) and dispatches the gate's full reviewer list (Step 1's first-pass branch). Each occurrence produces its **own independent verdict** (grade + per-reviewer verdict map).
- There is **no single-shot-per-run caching**: the runner MUST NOT memoize an occurrence's verdict and reuse it for a later occurrence of the same event in the same run, and MUST NOT skip dispatch on a later occurrence because an earlier occurrence already passed. A reviewer's verdict at occurrence *k* says nothing about occurrence *k+1* (the artifact/diff under review differs per occurrence).
- The rerun policy (Step 5) governs **re-evaluation of a single occurrence** after a fix — it does NOT carry state across distinct occurrences. The previous-pass verdict map threaded between Step-0 invocations applies only within one occurrence's fix-rerun loop, never across occurrences.

A gate keyed on a once-per-artifact lifecycle event (e.g., `feature.approved`) fires exactly once per run; the multi-fire contract above is a no-op for it (a single occurrence).

**Test-harness contract for `pre-commit-gate-fires-per-occurrence`.** The gate-runner harness simulates a run in which `implementation.pre_commit` occurs three times. The harness drives this runner three times — once per occurrence, each from Step 0 with an empty previous-pass map — and asserts: (1) the runner is invoked exactly three times (once per occurrence, not once per run); (2) each invocation dispatches the gate's reviewers (the dispatch spy from Step 6 records three independent dispatch sets); (3) each invocation yields its own verdict object, and the three verdicts are computed independently (no verdict is reused/cached from a prior occurrence). The Rehearse scenario stub at [`spec/features/reviewer-gates/_tests/pre-commit-gate-fires-per-occurrence.md`](../../../spec/features/reviewer-gates/_tests/pre-commit-gate-fires-per-occurrence.md) is the canonical place to author the Given/When/Then steps.

## Notes for skill authors

1. **Where to invoke this runner.** Call this runner immediately after [`loader.md`](./loader.md) Step 4 returns the validated reviewer list. Do not insert any other gate-related work between the loader and the runner; together they form the gate's single execution path.
2. **Single source of truth for verdicts.** The runner's per-reviewer verdict map is the authoritative record for the current pass. The consumer MUST NOT shadow it with its own derived state; on the next pass it passes the same map straight back as the "previous-pass verdict map" input.
3. **Halt-after-first-failure is mandatory, not configurable.** [`and-composition`](../../../spec/features/reviewer-gates/README.md#req-and-composition) is explicit on this. A consumer that wants to surface findings from multiple failing reviewers MUST collect them across multiple passes (each pass surfaces the first one, the user fixes, the rerun surfaces the next one) — it MUST NOT dispatch beyond the first `Issues Found` within a single pass.
4. **Severity discipline.** A `Blocker` finding MUST be surfaced to the user verbatim, MUST NOT be silently downgraded to `Advisory`, and MUST NOT be combined with other findings in a way that loses its severity label. Advisory findings MAY be ignored by the consumer at its discretion.
5. **No `spec/` writes from reviewers.** This is enforced at the runner level (Step 2.5) because the verdict contract explicitly classifies any `spec/` write as a producer action, not a reviewer action. A reviewer that attempts this is misconfigured; the runner refuses the gate and surfaces a clear error.
6. **Approval-phrase recognizer reuse.** The `type: human` dispatch path reuses the consumer skill's existing approval-phrase recognizer — the runner does NOT define a new recognizer. This is intentional: the recognizer is already battle-tested in `ideate`/`specify`'s user-approval gates and centralizing it here would create a parallel implementation. Future-tighten the recognizer in one place, both paths benefit.
7. **Per-artifact-type extension for `rerun-policy`.** See the boxed note inside Step 5. The structural-section list above is for SpecScore Feature artifacts (MVP scope). When a future consumer (`plan`, `implement`, etc.) wires this runner, that consumer's skill documentation defines its own structurally-load-bearing sections; the runner contract is unchanged.

## AC verification map

This runner is the implementation of the following acceptance criteria from the [reviewer-gates Feature](../../../spec/features/reviewer-gates/README.md):

| AC | Where verified in this runner |
|---|---|
| [`serial-dispatch-observed`](../../../spec/features/reviewer-gates/README.md#ac-serial-dispatch-observed) | Step 2 (serial dispatch loop with per-entry start/end timestamping) + Step 6 (mocked-Agent-tool spy harness: no-overlap and start-order-equals-registry-order assertions). |
| [`and-composition-blocks-on-any-issues-found`](../../../spec/features/reviewer-gates/README.md#ac-and-composition-blocks-on-any-issues-found) | Step 2 (all `type: ai` dispatched, no early-halt) + Step 2.8 (automated grade `C` < `B` → `Issues Found`, human phase skipped) + Step 4 (surface the `Blocker`; do NOT release; do NOT emit `feature.approved`). |
| [`deterministic-verdict-from-exit`](../../../spec/features/reviewer-gates/README.md#ac-deterministic-verdict-from-exit) | Step 2-det (exit code zero → `Approved`, no findings; non-zero → `Issues Found` with diagnostics as ≥1 `Blocker`) + Step 2.8.1 (the `Blocker` enters the union → grade ≤ `C` < default `B` → gate does NOT release). |
| [`auto-approve-always-approves`](../../../spec/features/reviewer-gates/README.md#ac-auto-approve-always-approves) | Step 2-auto-approve (dispatch nothing; record `Approved` with no findings; contribute no `Blocker`) + Step 2.8.1 (`auto-approve` contributes nothing to the union). |
| [`pre-commit-gate-fires-per-occurrence`](../../../spec/features/reviewer-gates/README.md#ac-pre-commit-gate-fires-per-occurrence) | Step 7 (each occurrence of `implementation.pre_commit` is a fresh gate run from Step 0 with an empty previous-pass map, dispatches the reviewers, and yields its own independent verdict; no single-shot-per-run caching) + the three-occurrence harness contract. |
| [`when-condition-masks-by-branch`](../../../spec/features/reviewer-gates/README.md#ac-when-condition-masks-by-branch) | Step 1.5 (resolve the current branch; a `when:`-masked entry participates iff the anchored regex matches — on no match it is neither dispatched nor counted toward the verdict; an entry with no `when:` always participates). The malformed-`when:` refusal is the loader's job ([`loader.md`](./loader.md) Step 3-when). |
| [`rerun-policy-applies-on-structural-fix`](../../../spec/features/reviewer-gates/README.md#ac-rerun-policy-applies-on-structural-fix) | Step 1 (rerun-pass dispatch-set computation: re-dispatch previously-`Issues Found` always; re-dispatch previously-`Approved` when structural-fix flag is `true`) + Step 5 (structural-section enumeration for Feature artifacts and per-artifact-type extension note). |
| [`grade-band-by-blocker-count`](../../../spec/features/reviewer-gates/README.md#ac-grade-band-by-blocker-count) | Step 2.8.1–2.8.2 (Blocker union over dispatched reviewers → band: 0→A/B, 1→C, 2–3→D, 4+→F). |
| [`within-band-letter-derivation`](../../../spec/features/reviewer-gates/README.md#ac-within-band-letter-derivation) | Step 2.8.3 (lowest within-band letter across reviewers; findings-only reviewer contributes default `B`) + Step 2.8.4 (threshold `A` releases only on grade `A`; default `B` releases on `A`/`B`). |
| [`worst-wins-union-across-reviewers`](../../../spec/features/reviewer-gates/README.md#ac-worst-wins-union-across-reviewers) | Step 2.8.1 (union of `Blocker`s across reviewers) — one Blocker from any reviewer → grade `C`, fails at default `B`. |
| [`threshold-default-reproduces-today`](../../../spec/features/reviewer-gates/README.md#ac-threshold-default-reproduces-today) | Step 2.8.4 with default threshold `B`: zero `Blocker`s → grade ≥ `B` → `Approved`; any `Blocker` → grade ≤ `C` → `Issues Found` (identical to pre-grade AND-composition). |
| [`lenient-threshold-tolerates-blocker`](../../../spec/features/reviewer-gates/README.md#ac-lenient-threshold-tolerates-blocker) | Step 2 (no early-halt) + Step 2.8 (`threshold: C`, one `Blocker` → grade `C` → `C ≥ C` → automated grade passes → `Approved`). |

### Walk-through against AC `serial-dispatch-observed`

> **Given** a `gates.feature.approved.reviewers` list with three entries (two `ai` plus one `human`) and instrumentation that records dispatch start/end timestamps per entry,
> **When** `specstudio:specify` runs through the gate,
> **Then** at no point during the run are two reviewer dispatches concurrently in flight, and the recorded dispatch start order matches the list order exactly.

Following this runner: Step 1 includes all three entries in the dispatch set (first pass, no previous verdicts). Step 2 walks them in declared order; each entry records `[start, end]` and the next dispatch is invoked only after the previous verdict is collected (Step 2.3 wording: "after the verdict is collected"). Step 6's harness then asserts (a) no two intervals overlap and (b) sorted-by-start order equals declared order — both assertions hold by construction of Step 2. Outcome matches.

### Walk-through against AC `and-composition-blocks-on-any-issues-found`

> **Given** a `gates.feature.approved.reviewers` list with two `ai` entries followed by one `human` entry, where the first `ai` entry returns `Approved` and the second `ai` entry returns `Issues Found` with one `Blocker` finding,
> **When** `specstudio:specify` runs through the gate,
> **Then** both `ai` entries MUST be dispatched (no early-halt), the automated grade MUST be `C`, the gate MUST NOT release, the skill MUST surface the `Blocker` finding, the human entry MUST NOT be dispatched (automated grade below the default threshold `B`), and the skill MUST NOT emit `feature.approved`.

Following this runner: Step 1 includes all three in the dispatch set. Step 2 (automated phase) dispatches the first `ai` entry (`Approved`), then the second `ai` entry (`Issues Found`, one `Blocker`) — no halt, both run. Step 2.8 computes the automated grade from the union (one `Blocker` → `C`); since `C < B` (default threshold) it routes to Step 4 and skips the human phase, so the third (human) entry is NOT dispatched. Step 4 surfaces the second `ai` entry's `Blocker` verbatim and returns `Issues Found`. The consumer (`specstudio:specify`) does not emit `feature.approved`. Outcome matches.

### Walk-through against AC `when-condition-masks-by-branch`

> **Given** a `gates.implementation.pre_commit.reviewers` list with a `type: human` entry carrying `when: "branch =~ ^(main|master|release/)"`,
> **When** the gate is evaluated on a feature branch (e.g., `feature/x`) and separately on `main`,
> **Then** on the feature branch the human entry does NOT participate (commits stay autonomous) and on `main` it does (the human is asked before commit); the per-branch behavior comes entirely from the gate-entry `when:` condition.

Following this runner: the loader (Step 3-when) validated the `when:` shape and carried it onto the human entry's normalized record. **On `feature/x`:** Step 1 puts the human entry in the dispatch set (first pass); Step 1.5 resolves the current branch `feature/x`, evaluates `^(main|master|release/)` anchored against it, finds **no match**, and removes the human entry — it is neither dispatched nor counted. The remaining (e.g., `auto-approve`/`ai`) entries with no `when:` carry the gate, which releases without asking the human (commits stay autonomous). **On `main`:** Step 1.5 evaluates the same regex against `main`, finds a **match**, and keeps the human entry; Step 2.9's human phase asks the human before the commit proceeds. The only difference between the two runs is the current-branch input fed to Step 1.5 — the per-branch behavior comes entirely from the gate-entry `when:` condition. Outcome matches.

### Walk-through against AC `rerun-policy-applies-on-structural-fix`

> **Given** a gate run in which reviewer A (`type: ai`) returned `Approved`, reviewer B (`type: ai`) returned `Issues Found`, and the user has applied a fix that modifies the Feature's `## Behavior` section,
> **When** `specstudio:specify` re-runs the gate after the fix,
> **Then** both reviewer A (because the fix touched a structural section) and reviewer B (because it previously returned `Issues Found`) MUST be re-dispatched in list order before the gate verdict is re-evaluated.

Following this runner: the consumer invokes the runner with the previous-pass verdict map `{A → Approved, B → Issues Found}` and the structural-fix flag `true` (the fix touched `## Behavior`, which Step 5's enumeration lists as structural for Feature artifacts). Step 1's rerun branch evaluates each entry:

- Reviewer A: previous verdict `Approved`, structural flag `true` → dispatch.
- Reviewer B: previous verdict `Issues Found` → dispatch (always, regardless of flag).

The dispatch set is `[A, B]` in declared order. Step 2 walks both in that order, collecting fresh verdicts before the gate verdict is re-evaluated in Step 3 or Step 4. Outcome matches.
