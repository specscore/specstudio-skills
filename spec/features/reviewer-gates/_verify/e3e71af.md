# Verify Report: reviewer-gates @ e3e71af

```yaml
feature: reviewer-gates
revision: e3e71af
verdicts:
  - ac: reviewer-gates#ac:gates-block-preserved
    verdict: pass
    justification: "Loader Step 1 + Note 6 mandate preserving the gates: block verbatim on rewrite via unknown-fields-preserved; Step 2 resolves gates.<skill>.reviewers and Step 4 returns the list in declared order. specscore.yaml provides a real two-entry ordered example as ground truth."
  - ac: reviewer-gates#ac:untyped-entry-refused
    verdict: pass
    justification: "Loader Step 3b refuses entries missing type:, citing no-untyped-entry with link to the reviewer-gates Feature; top-of-file failure protocol mandates halt-before-dispatch, no artifact writes, and non-zero exit."
  - ac: reviewer-gates#ac:unknown-type-refused
    verdict: pass
    justification: "Step 3b refuses any type: outside {ai, human} including type: security, cites mvp-type-set, links the Feature, forbids treating unknown types as ai; MUST-clauses require halt-without-dispatch and non-zero exit."
  - ac: reviewer-gates#ac:ai-entry-shape-violations-refused
    verdict: pass
    justification: "Step 3c handles all three sub-cases (3c.i missing prompt:, 3c.ii path outside working tree, 3c.iii prompt file lacks Blocker/Advisory taxonomy). Each refusal cites ai-entry-shape, links the Feature, halts without dispatch, exits non-zero."
  - ac: reviewer-gates#ac:human-entry-min-approvers-cap
    verdict: pass
    justification: "Step 3d.ii rejects min_approvers > 1 on type: human entries, citing human-entry-shape's MVP cap and linking the reviewer-gates Feature; protocol requires halting (no dispatch, non-zero exit). AC verification map maps to Step 3d.ii."
  - ac: reviewer-gates#ac:human-entry-rejects-prompt
    verdict: pass
    justification: "Loader Step 3d.i forbids prompt: on type: human entries, mandates an error citing human-entry-shape's prohibition with Feature link, halts non-zero per the global refusal contract."
  - ac: reviewer-gates#ac:serial-dispatch-observed
    verdict: pass
    justification: "runner.md Step 2 mandates serial in-order dispatch with per-entry start/end timestamping and no-concurrent-in-flight rule; Step 6 defines the mocked Agent-tool spy harness recording (name, type, start_ts, end_ts, verdict) and asserts both no-overlap and start-order = registry-order."
  - ac: reviewer-gates#ac:and-composition-blocks-on-any-issues-found
    verdict: pass
    justification: "runner.md Step 2.7 mandates halt-after-first-Issues-Found (third entry not dispatched), Step 4 + Output require surfacing Blocker verbatim and forbid emitting feature.approved; dedicated walk-through covers this exact scenario."
  - ac: reviewer-gates#ac:rerun-policy-applies-on-structural-fix
    verdict: pass
    justification: "runner.md Step 1 dispatches previously-Issues-Found always and previously-Approved when structural-fix flag is true; Step 5 lists ## Behavior as structural for Features; Step 2 dispatches in declared order before verdict eval."
  - ac: reviewer-gates#ac:specify-loads-gate-not-builtin
    verdict: pass
    justification: "skills/specify/SKILL.md loads gates.specify.reviewers via loader and dispatches via runner with explicit no-hardcoded-baseline contract; specscore.yaml has exactly [ai → reviewer-prompt.md path, human] in order; reviewer-prompt.md has Blocker/Advisory taxonomy section."
  - ac: reviewer-gates#ac:missing-gates-block-refuses-with-error
    verdict: pass
    justification: "Loader Step 2 enumerates states (a) no gates:, (b) no gates.<skill>, (c) empty reviewers: [], refuses each with error citing reviewer-gates Feature link, recommends minimal type: human config, halts with no dispatch and no artifact write."
  - ac: reviewer-gates#ac:third-party-integration-revised
    verdict: pass
    justification: "All 6 reviewer-* REQ slugs and the reviewer-registration-and-composition AC absent from third-party-integration/README.md; Interaction-with-Other-Features table row links to ../reviewer-gates/README.md; specscore spec lint passes."
  - ac: reviewer-gates#ac:specify-feature-revised
    verdict: pass
    justification: "All 5 legacy reviewer-* REQ slugs absent from spec/features/skills/specify/README.md; single ### Reviewer gate topic links to ../../reviewer-gates/README.md; new AC consumes-gates-specify asserts gates.specify.reviewers consumption; lint clean."
  - ac: reviewer-gates#ac:review-feature-archived
    verdict: pass
    justification: "spec/features/skills/review/README.md Status body-metadata reads Archived; **Archive Reason:** line cites supersession by reviewer-gates; specscore spec lint returns 0 violations."
  - ac: reviewer-gates#ac:root-readme-link-present
    verdict: pass
    justification: "Root README.md contains link [Reviewer Gates](spec/features/reviewer-gates/README.md); pipeline arrow-chain reads ideate ⇒ specify ⇒ plan ⇒ implement ⇒ verify ⇒ recap ⇒ ship with no discrete review step."
  - ac: reviewer-gates#ac:skill-doc-cross-links-present
    verdict: pass
    justification: "All three files link to reviewer-gates/README.md (SKILL.md L25/60/284; specify feature README L11/17/48/145/149/153/238/251/283/311/312; features-index L16); features-index row has non-trivial description."
counts:
  total_acs: 16
  passed_count: 16
  failed_count: 0
  unmapped_count: 0
  errored_count: 0
```

## AC: gates-block-preserved

**Verdict:** `pass`

**Justification:** Loader Step 1 + Note 6 mandate preserving the `gates:` block verbatim on rewrite via `unknown-fields-preserved`; Step 2 resolves `gates.<skill>.reviewers` and Step 4 returns the list in declared order. `specscore.yaml` provides a real two-entry ordered example as ground truth.

**Commits:** `a05f52b`

**Evidence:**
- `skills/shared/reviewer-gates/loader.md` (Step 1, Step 2, Step 4, Note 6, AC map row `gates-block-preserved`)
- `specscore.yaml` (lines 9-17: `gates.specify.reviewers` with ordered entries `spec-document-reviewer` then `user-approval`)

## AC: untyped-entry-refused

**Verdict:** `pass`

**Justification:** Loader Step 3b refuses entries missing `type:`, citing `no-untyped-entry` with link to the reviewer-gates Feature; top-of-file failure protocol mandates halt-before-dispatch, no artifact writes, and non-zero exit, satisfying all four obligations of the AC.

**Commits:** `a05f52b`

**Evidence:**
- `skills/shared/reviewer-gates/loader.md` (Step 3b; top-of-file failure protocol items 1-3; AC verification map row `untyped-entry-refused`)

## AC: unknown-type-refused

**Verdict:** `pass`

**Justification:** Step 3b explicitly refuses any `type:` outside `{ai, human}` (including `type: security`), cites `req:mvp-type-set`, links the Feature, forbids treating unknown types as `ai`, and the loader's MUST clauses require halt-without-dispatch and non-zero exit.

**Commits:** `a05f52b`

**Evidence:**
- `skills/shared/reviewer-gates/loader.md` (Step 3b; opening MUST list; AC verification map row for `unknown-type-refused`)

## AC: ai-entry-shape-violations-refused

**Verdict:** `pass`

**Justification:** Step 3c in `loader.md` explicitly handles all three sub-cases (3c.i missing `prompt:`, 3c.ii path outside working tree incl. absolute/URL/escaping `..`-segments, 3c.iii prompt file lacks Blocker/Advisory taxonomy). Each refusal cites `req:ai-entry-shape`, links to the Feature, halts without dispatch, exits non-zero.

**Commits:** `a05f52b`

**Evidence:**
- `skills/shared/reviewer-gates/loader.md` (Step 3c.i, 3c.ii, 3c.iii; AC verification map row)

## AC: human-entry-min-approvers-cap

**Verdict:** `pass`

**Justification:** Step 3d.ii rejects `min_approvers > 1` on `type: human` entries, citing `human-entry-shape`'s MVP cap and linking to the reviewer-gates Feature; the overarching protocol requires halting (no dispatch, non-zero exit). AC verification map explicitly maps this AC to Step 3d.ii.

**Commits:** `a05f52b`

**Evidence:**
- `skills/shared/reviewer-gates/loader.md` (Step 3d.ii; AC verification map row; top-level "When validation fails" block)

## AC: human-entry-rejects-prompt

**Verdict:** `pass`

**Justification:** Loader Step 3d.i explicitly forbids `prompt:` on `type: human` entries, mandates an error message citing `human-entry-shape`'s prohibition with Feature link, and halts non-zero per the global refusal contract (lines 12–18). AC verification map row at line 170 confirms the mapping.

**Commits:** `a05f52b`

**Evidence:**
- `skills/shared/reviewer-gates/loader.md` (Step 3d.i, lines 127–129; refusal contract lines 12–18; AC verification map line 170)

## AC: serial-dispatch-observed

**Verdict:** `pass`

**Justification:** `runner.md` Step 2 mandates serial in-order dispatch with per-entry start/end timestamping and explicit no-concurrent-in-flight rule; Step 6 defines a mocked Agent-tool spy harness recording `(name, type, start_ts, end_ts, verdict)` and asserts both no-overlap and start-order-equals-registry-order; dedicated walk-through quotes the AC's three-entry (2 ai + 1 human) scenario verbatim.

**Commits:** `5cef20f`

**Evidence:**
- `skills/shared/reviewer-gates/runner.md` (Step 2.1, 2.3, Step 6, "Walk-through against AC `serial-dispatch-observed`")

## AC: and-composition-blocks-on-any-issues-found

**Verdict:** `pass`

**Justification:** `runner.md` Step 2.7 mandates halt-after-first-`Issues Found` (third entry not dispatched), Step 4 + Output require surfacing `Blocker` verbatim and forbid emitting `feature.approved`; AC verification map and dedicated walk-through cover this exact scenario.

**Commits:** `5cef20f`

**Evidence:**
- `skills/shared/reviewer-gates/runner.md` (Step 2.7, Step 4, Output section, Notes #3, "Walk-through against AC `and-composition-blocks-on-any-issues-found`")

## AC: rerun-policy-applies-on-structural-fix

**Verdict:** `pass`

**Justification:** `runner.md` Step 1 dispatches previously-`Issues Found` always and previously-`Approved` when structural-fix flag is true; Step 5 lists `## Behavior` as structural for Features; Step 2 dispatches in declared order before Step 3/4 verdict eval; dedicated AC walk-through confirms.

**Commits:** `5cef20f`

**Evidence:**
- `skills/shared/reviewer-gates/runner.md` (Step 1, Step 5, Step 2, AC walk-through `rerun-policy-applies-on-structural-fix`)

## AC: specify-loads-gate-not-builtin

**Verdict:** `pass`

**Justification:** `skills/specify/SKILL.md` loads `gates.specify.reviewers` via the loader and dispatches via the runner with explicit "no hardcoded baseline" contract; `specscore.yaml` has exactly `[ai → reviewer-prompt.md path, human]` in order; `reviewer-prompt.md` has a `## Blocker / Advisory taxonomy` section satisfying `ai-entry-shape`.

**Commits:** `3635235`

**Evidence:**
- `skills/specify/SKILL.md:190-212`
- `specscore.yaml:9-17`
- `skills/specify/references/reviewer-prompt.md:66-81`

## AC: missing-gates-block-refuses-with-error

**Verdict:** `pass`

**Justification:** Step 2 of `loader.md` explicitly enumerates states (a) no `gates:`, (b) no `gates.<skill>`, (c) empty `reviewers: []`, refuses each with error citing the reviewer-gates Feature link, recommends minimal `type: human` config, mandates halt with no dispatch and no artifact write (per top-level rules 1-3).

**Commits:** `a05f52b`

**Evidence:**
- `skills/shared/reviewer-gates/loader.md` (Step 2, lines 46-70; lines 12-18 refusal-protocol preamble)

## AC: third-party-integration-revised

**Verdict:** `pass`

**Justification:** All 6 removed REQ slugs and the `reviewer-registration-and-composition` AC are absent from `spec/features/third-party-integration/README.md`; the Interaction-with-Other-Features table contains a row linking to `../reviewer-gates/README.md` at line 128; `specscore spec lint` returns "0 violations found".

**Commits:** `a05f52b`

**Evidence:**
- `spec/features/third-party-integration/README.md` (line 128)
- `specscore spec lint`: 0 violations found

## AC: specify-feature-revised

**Verdict:** `pass`

**Justification:** All 5 removed REQ slugs absent; single `### Reviewer gate` topic links to `../../reviewer-gates/README.md`; new `### AC: consumes-gates-specify` asserts `gates.specify.reviewers` consumption; `specscore spec lint` reports 0 violations.

**Commits:** `a05f52b`

**Evidence:**
- `spec/features/skills/specify/README.md` (line 143 topic, line 277 AC, lines 48/149/153/173/238/251/281/311 `gates.specify` refs)
- `specscore spec lint`: 0 violations found

## AC: review-feature-archived

**Verdict:** `pass`

**Justification:** Status body-metadata line reads `Archived`; `**Archive Reason:**` line cites supersession by `reviewer-gates`; `specscore spec lint` returns "0 violations found".

**Commits:** `a05f52b`

**Evidence:**
- `spec/features/skills/review/README.md`
- `specscore spec lint`: 0 violations found

## AC: root-readme-link-present

**Verdict:** `pass`

**Justification:** Root `README.md` line 5 contains link `[Reviewer Gates](spec/features/reviewer-gates/README.md)`; pipeline arrow-chain reads `ideate ⇒ specify ⇒ plan ⇒ implement ⇒ verify ⇒ recap ⇒ ship` with no discrete `review` step (only the proper-noun "Reviewer Gates" reference, which is not a violation).

**Commits:** `5cef20f`

**Evidence:**
- `README.md:5`

## AC: skill-doc-cross-links-present

**Verdict:** `pass`

**Justification:** All three files link to `reviewer-gates/README.md` (`SKILL.md` L25/60/284; `specify` feature README L11/17/48/145/149/153/238/251/283/311/312; features-index L16) and features-index L16 has a row with a non-trivial multi-sentence description, not boilerplate.

**Commits:** `5cef20f`

**Evidence:**
- `skills/specify/SKILL.md` (lines 25, 60, 284)
- `spec/features/skills/specify/README.md` (lines 11, 17, 48, 145, 149, 153, 238, 251, 283, 311, 312)
- `spec/features/README.md` (line 16)
