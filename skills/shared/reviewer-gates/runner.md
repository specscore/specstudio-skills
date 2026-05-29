# Reviewer-Gates Gate Runner

**Status:** Contract — shared instructional document for skills that execute a reviewer gate after loading its configuration via [`loader.md`](./loader.md).
**Owned by:** [reviewer-gates](../../../spec/features/reviewer-gates/README.md) Feature.

## Purpose

This file tells a consuming skill (currently `specstudio:specify`; future consumers MUST follow the same protocol) how to **execute** a reviewer gate against the validated reviewer list produced by [`loader.md`](./loader.md). The runner is the second half of the reviewer-gates contract: where the loader does load-time validation, the runner does dispatch, verdict aggregation, halt-after-first-failure, and rerun-policy enforcement.

The runner's output is the gate's **grade** (a whole letter `A`–`F`) and the derived verdict for the consuming skill:

- `Approved` — the computed grade satisfies `grade ≥ threshold`. The consumer MAY release whatever downstream gate it owns (e.g., for `specstudio:specify`, the User Review Gate is now released and the skill MAY emit `feature.approved`).
- `Issues Found` — `grade < threshold`. The consumer MUST surface the failing grade and the `Blocker` findings to the user, MUST NOT release any downstream gate, and MUST NOT emit any approval event. Re-running the gate after the user's fix is governed by Step 5 (Rerun policy) below.

There is no third verdict. The runner does not "skip" reviewers, does not silently downgrade `Blocker` severity to `Advisory`, does not inject any reviewer the loader did not validate, and does not write to `spec/`.

This runner implements the following REQs from the [reviewer-gates Feature](../../../spec/features/reviewer-gates/README.md):

- [`dispatch-serial`](../../../spec/features/reviewer-gates/README.md#req-dispatch-serial) — one reviewer in flight at a time, in list order.
- [`verdict-contract`](../../../spec/features/reviewer-gates/README.md#req-verdict-contract) — `Blocker`/`Advisory` severity on findings; no reviewer writes to `spec/`.
- [`and-composition`](../../../spec/features/reviewer-gates/README.md#req-and-composition) — threshold-aware early-halt (halt on first `Blocker` only at zero-`Blocker` thresholds `A`/`B`; dispatch all at lenient thresholds).
- [`grade-band-mapping`](../../../spec/features/reviewer-gates/README.md#req-grade-band-mapping), [`grade-aggregation`](../../../spec/features/reviewer-gates/README.md#req-grade-aggregation), [`threshold-derived-verdict`](../../../spec/features/reviewer-gates/README.md#req-threshold-derived-verdict) — compute the grade from the `Blocker` union + within-band letter (Step 2.8) and release iff `grade ≥ threshold`.
- [`rerun-policy`](../../../spec/features/reviewer-gates/README.md#req-rerun-policy) — re-dispatch previously-`Issues Found` reviewers on rerun; ALSO re-dispatch previously-`Approved` reviewers when the fix touched a structural section.

## Inputs

| Input | Source | Required |
|---|---|---|
| Validated reviewer list | The ordered list returned by [`loader.md`](./loader.md) Step 4. The consumer MUST NOT modify this list before passing it here. | Yes |
| Calling skill's Agent-tool dispatch capability | For `type: ai` entries. The consumer's host (Claude with the skill's runtime context) is the dispatcher. | Yes |
| Calling skill's approval-phrase recognizer | For `type: human` entries. The same recognizer used by `specstudio:ideate` and `specstudio:specify` for user-approval gates (`approve` / `approved` / `accept` / `accepted` / `lgtm` / direct semantic equivalents → `Approved`; explicit change requests → `Issues Found` with the user's text captured as a single `Blocker` finding). | Yes |
| Resolved Approve threshold | The whole-letter threshold (`A`–`F`, default `B`) returned by [`loader.md`](./loader.md) Step 2.5. The gate releases iff `grade ≥ threshold`. | Yes |
| Previous-pass verdict map (rerun only) | A map of `<reviewer-name> → <last-verdict>` from the previous pass, used only when this is a rerun. On the first pass the map is empty. | Yes (may be empty) |
| Structural-fix flag (rerun only) | Boolean. `true` when the user's fix between passes touched a structural section of the artifact under review (see Step 5 for what counts as structural per artifact type). On the first pass, this flag is unused. | Yes (may be `false`) |

## Output

The runner returns the **gate grade** (a whole letter `A`–`F`, computed per Step 2.8) and the derived gate verdict, exactly one of:

- `Approved` — the grade satisfies `grade ≥ threshold`. Accompanied by the grade and the per-reviewer verdict map for this pass. The consumer is now released to proceed.
- `Issues Found` — `grade < threshold`. Accompanied by (a) the grade, (b) the surfaced reviewer(s) and their structured findings list (each finding carrying `Blocker` or `Advisory` severity and a prose description), and (c) the per-reviewer verdict map for this pass (reviewers the runner halted before reaching are absent from the map). The consumer MUST surface at least the `Blocker` findings to the user and MUST NOT release any downstream gate or emit any approval event.

In both cases the runner returns the grade plus the per-reviewer verdict map so the consumer can pass the map back as the "previous-pass verdict map" input on the next pass (the rerun), and surface the grade (e.g., `specstudio:score` renders it; a producer-exit gate may log it).

## Protocol

Follow the steps in order. Do not skip ahead. The first halt-condition is terminal for the current pass.

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
  - If the entry is absent from the previous-pass verdict map (i.e., the previous pass halted before reaching it): dispatch it. The previous pass never collected its verdict; the rerun MUST collect one.

The order within the dispatch set is exactly the declared order of the original reviewer list — entries skipped above do not shift the order of the remaining entries.

### Step 2 — Dispatch serially, in declared order

For each entry in the dispatch set, in order:

1. **Record dispatch start.** Capture the timestamp at the moment dispatch begins (for AC `serial-dispatch-observed`'s instrumentation contract — see Step 6 below).
2. **Dispatch by type:**
   - `type: ai` — invoke the consumer skill's Agent tool with the entry's resolved `prompt_path` file contents as the system prompt. Pass the artifact under review (e.g., the Feature `README.md` text for `specstudio:specify`) as the user-facing message. Wait for the subagent to return. The subagent MUST respond with exactly one of `Approved` or `Issues Found`; on `Issues Found`, the response carries a structured findings list per the prompt's documented blocker/advisory taxonomy.
   - `type: human` — invoke the consumer skill's existing approval-phrase recognizer (the same one used by `specstudio:ideate` and `specstudio:specify` for user-approval gates). Present the artifact under review to the user; wait for the user's response. Map the response:
     - Explicit approval phrase (`approve` / `approved` / `accept` / `accepted` / `lgtm` or direct semantic equivalents in the user's language) → `Approved`.
     - Explicit change request → `Issues Found` with the user's change-request text captured as a single `Blocker` finding. The reviewer-name in surfaced findings is the entry's `name:`.
     - Ambiguous response → ask the user to clarify (this is not a verdict yet — the recognizer's standard ambiguity-handling applies). Continue waiting.
3. **Record dispatch end.** Capture the timestamp at the moment the verdict is collected. The recorded `[start, end]` interval for this entry MUST be disjoint from every other entry's interval in this pass — no two dispatches concurrently in flight. (See Step 6 for the instrumentation contract and the test-harness assertion shape.)
4. **Validate the verdict shape.** The returned verdict MUST be exactly `Approved` or `Issues Found`. If a `type: ai` subagent returns anything else (e.g., omits the verdict, returns prose with no decision), the runner MUST refuse the gate with `Error: reviewer '<name>' returned a non-conforming verdict (expected 'Approved' or 'Issues Found'); reviewer prompts MUST conform to the documented taxonomy per reviewer-gates#req:verdict-contract`. No silent retry, no inference. This is a reviewer-prompt bug surfaced to the user.
5. **Reject writes to `spec/`.** If a reviewer attempted (or its findings instruct the consumer to perform) a write or modification to any artifact under `spec/`, the runner MUST refuse the gate with `Error: reviewer '<name>' is a misclassified Producer; reviewers MUST NOT write to spec/ per reviewer-gates#req:verdict-contract`. (Reviewers are read-only; producers own artifact writes.)
6. **Record the verdict, findings, and within-band letter.** Record `<entry.name> → <verdict>` in this pass's verdict map, and retain the entry's structured findings (each with `Blocker`/`Advisory` severity). For a `type: ai` entry whose prompt declares the multi-role lenses, also retain the single within-band letter it reported (per [`multi-role-reviewer`](../../../spec/features/reviewer-gates/README.md#req-multi-role-reviewer)); a findings-only reviewer reports no within-band letter (treated as the default `B` in Step 2.8).
7. **Check for the threshold-aware halt condition.** If the verdict is `Issues Found` **and** the resolved threshold is a zero-`Blocker` bar (`A` or `B`): **halt the pass immediately** — record the remaining entries as "not dispatched in this pass" (their verdict-map slots remain absent) and proceed to Step 2.8 (Compute the grade). The first `Blocker` already guarantees `grade < threshold` at these thresholds, so dispatching further reviewers is wasted effort — this is the mandatory halt of [`and-composition`](../../../spec/features/reviewer-gates/README.md#req-and-composition), now scoped to zero-`Blocker` thresholds. If the threshold is **lenient** (`C`, `D`, or `F`): do NOT halt — continue dispatching the remaining entries so the full `Blocker` union can be counted, because a tolerable `Blocker` count may still satisfy `grade ≥ threshold`. When the dispatch set is exhausted (or the halt fired), proceed to Step 2.8.

### Step 2.8 — Compute the grade and derive the gate verdict

After the dispatch loop completes for this pass — whether it ran the full dispatch set or halted early per the zero-`Blocker` threshold rule — compute the gate grade from the findings collected so far, per [`grade-band-mapping`](../../../spec/features/reviewer-gates/README.md#req-grade-band-mapping) and [`grade-aggregation`](../../../spec/features/reviewer-gates/README.md#req-grade-aggregation):

1. **Blocker union.** Form the union of all `Blocker` findings across every reviewer **dispatched in this pass** and (for a multi-role reviewer) every lens. Let `n = |union|`.
2. **Band by Blocker count.** `n == 0` → pass band (`A`/`B`); `n == 1` → `C`; `2 ≤ n ≤ 3` → `D`; `n ≥ 4` → `F`.
3. **Within-band letter (pass band only).** Across the reviewers that reported a within-band letter, take the **lowest** (since `A > B`, any reported `B` wins over `A`). A reviewer that reported no within-band letter (findings-only) contributes the default `B`. Result: the grade is `A` iff at least one reviewer reported `A` and none reported `B`; otherwise `B`. (A multi-role reviewer reports exactly one within-band letter — lenses feed only the `Blocker` union, not separate letters.)
4. **Derive the verdict.** With letters ordered `A > B > C > D > F`, the gate verdict is `Approved` iff `grade ≥ threshold`, else `Issues Found` (per [`threshold-derived-verdict`](../../../spec/features/reviewer-gates/README.md#req-threshold-derived-verdict)).

The early-halt at zero-`Blocker` thresholds never changes this verdict: if the halt fired, `n ≥ 1`, so `grade ≤ C < threshold` (threshold is `A` or `B`) → `Issues Found`. The halt only bounds how many `Blocker`s were counted; it cannot turn a fail into a pass.

Proceed to Step 3 when the derived verdict is `Approved`, else Step 4.

### Step 3 — Gate verdict is `Approved`

When Step 2.8 derives `Approved` (`grade ≥ threshold`), return `Approved` with the computed grade and the per-reviewer verdict map. (At the default threshold `B` this coincides exactly with "every dispatched reviewer returned `Approved`" — zero `Blocker`s — with every previously-`Approved` entry skipped per Step 1's non-structural-fix allowance carrying its verdict forward.)

### Step 4 — Gate verdict is `Issues Found`

When Step 2.8 derives `Issues Found` (`grade < threshold`), the gate did not release.

Surface to the user:

- The computed grade and the resolved threshold (e.g., "Grade `C`, threshold `B` — not released").
- The reviewer(s) whose findings drove the failing grade and their structured findings. At a zero-`Blocker` threshold the halt makes this the first reviewer that returned `Issues Found`; at a lenient threshold (no early-halt) surface every dispatched reviewer's `Blocker` findings. **At a minimum every `Blocker` finding contributing to the grade MUST be surfaced verbatim.** Advisory findings SHOULD also be surfaced (the consumer MAY format them under a clearly-labeled "Advisory" section), but they do not affect the verdict.
- A clear statement that the gate did NOT release and no downstream event (e.g., `feature.approved`) was emitted.
- A clear statement that any subsequent reviewer in the list was NOT dispatched in this pass, naming them by `name:` if the user benefits from seeing the deferred set (especially for `type: human` reviewers: the user themselves may be the next-in-line reviewer who was halted before; saying so explicitly avoids confusion).

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
- For reviewers absent from the previous-pass verdict map (the ones halted before in the previous pass): always dispatch — the previous pass never collected their verdict.

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
| [`and-composition-blocks-on-any-issues-found`](../../../spec/features/reviewer-gates/README.md#ac-and-composition-blocks-on-any-issues-found) | Step 2.7 (halt on first `Issues Found` within the pass — subsequent entries not dispatched), Step 4 (surface the first failing reviewer's `Blocker` findings; do NOT release any downstream gate; do NOT emit `feature.approved`). |
| [`rerun-policy-applies-on-structural-fix`](../../../spec/features/reviewer-gates/README.md#ac-rerun-policy-applies-on-structural-fix) | Step 1 (rerun-pass dispatch-set computation: re-dispatch previously-`Issues Found` always; re-dispatch previously-`Approved` when structural-fix flag is `true`) + Step 5 (structural-section enumeration for Feature artifacts and per-artifact-type extension note). |
| [`grade-band-by-blocker-count`](../../../spec/features/reviewer-gates/README.md#ac-grade-band-by-blocker-count) | Step 2.8.1–2.8.2 (Blocker union over dispatched reviewers → band: 0→A/B, 1→C, 2–3→D, 4+→F). |
| [`within-band-letter-derivation`](../../../spec/features/reviewer-gates/README.md#ac-within-band-letter-derivation) | Step 2.8.3 (lowest within-band letter across reviewers; findings-only reviewer contributes default `B`) + Step 2.8.4 (threshold `A` releases only on grade `A`; default `B` releases on `A`/`B`). |
| [`worst-wins-union-across-reviewers`](../../../spec/features/reviewer-gates/README.md#ac-worst-wins-union-across-reviewers) | Step 2.8.1 (union of `Blocker`s across reviewers) — one Blocker from any reviewer → grade `C`, fails at default `B`. |
| [`threshold-default-reproduces-today`](../../../spec/features/reviewer-gates/README.md#ac-threshold-default-reproduces-today) | Step 2.8.4 with default threshold `B`: zero `Blocker`s → grade ≥ `B` → `Approved`; any `Blocker` → grade ≤ `C` → `Issues Found` (identical to pre-grade AND-composition). |
| [`lenient-threshold-tolerates-blocker`](../../../spec/features/reviewer-gates/README.md#ac-lenient-threshold-tolerates-blocker) | Step 2.7 (no early-halt at lenient threshold) + Step 2.8.4 (`threshold: C`, one `Blocker` → grade `C` → `C ≥ C` → `Approved`). |

### Walk-through against AC `serial-dispatch-observed`

> **Given** a `gates.specify.reviewers` list with three entries (two `ai` plus one `human`) and instrumentation that records dispatch start/end timestamps per entry,
> **When** `specstudio:specify` runs through the gate,
> **Then** at no point during the run are two reviewer dispatches concurrently in flight, and the recorded dispatch start order matches the list order exactly.

Following this runner: Step 1 includes all three entries in the dispatch set (first pass, no previous verdicts). Step 2 walks them in declared order; each entry records `[start, end]` and the next dispatch is invoked only after the previous verdict is collected (Step 2.3 wording: "after the verdict is collected"). Step 6's harness then asserts (a) no two intervals overlap and (b) sorted-by-start order equals declared order — both assertions hold by construction of Step 2. Outcome matches.

### Walk-through against AC `and-composition-blocks-on-any-issues-found`

> **Given** a `gates.specify.reviewers` list with two `ai` entries followed by one `human` entry, where the first `ai` entry returns `Approved` and the second `ai` entry returns `Issues Found` with one `Blocker` finding,
> **When** `specstudio:specify` runs through the gate,
> **Then** the gate MUST NOT release, the skill MUST surface the `Blocker` finding to the user, the third entry (the human) MUST NOT be dispatched in the same pass after the failure, and the skill MUST NOT emit `feature.approved`.

Following this runner: Step 1 includes all three in the dispatch set. Step 2 dispatches the first `ai` entry, gets `Approved`, appends to the verdict map. Step 2 then dispatches the second `ai` entry, gets `Issues Found` with one `Blocker` finding. Step 2.7's halt condition fires — the runner jumps to Step 4 WITHOUT dispatching the third (human) entry. Step 4 surfaces the second `ai` entry's name and its `Blocker` finding verbatim, returns verdict `Issues Found` with the verdict map containing only the two reached entries. The consumer (`specstudio:specify`), per its skill spec, does not emit `feature.approved` when the runner returns `Issues Found`. Outcome matches.

### Walk-through against AC `rerun-policy-applies-on-structural-fix`

> **Given** a gate run in which reviewer A (`type: ai`) returned `Approved`, reviewer B (`type: ai`) returned `Issues Found`, and the user has applied a fix that modifies the Feature's `## Behavior` section,
> **When** `specstudio:specify` re-runs the gate after the fix,
> **Then** both reviewer A (because the fix touched a structural section) and reviewer B (because it previously returned `Issues Found`) MUST be re-dispatched in list order before the gate verdict is re-evaluated.

Following this runner: the consumer invokes the runner with the previous-pass verdict map `{A → Approved, B → Issues Found}` and the structural-fix flag `true` (the fix touched `## Behavior`, which Step 5's enumeration lists as structural for Feature artifacts). Step 1's rerun branch evaluates each entry:

- Reviewer A: previous verdict `Approved`, structural flag `true` → dispatch.
- Reviewer B: previous verdict `Issues Found` → dispatch (always, regardless of flag).

The dispatch set is `[A, B]` in declared order. Step 2 walks both in that order, collecting fresh verdicts before the gate verdict is re-evaluated in Step 3 or Step 4. Outcome matches.
