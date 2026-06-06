---
format: https://specscore.md/feature-specification
status: Approved
---

# Feature: Reviewer Gates

> [SpecScore.**Studio**](https://specscore.studio): | [Explore](https://specscore.studio/app/github.com/specscore/specstudio-skills/spec/features/reviewer-gates?op=explore) | [Edit](https://specscore.studio/app/github.com/specscore/specstudio-skills/spec/features/reviewer-gates?op=edit) | [Ask question](https://specscore.studio/app/github.com/specscore/specstudio-skills/spec/features/reviewer-gates?op=ask) | [Request change](https://specscore.studio/app/github.com/specscore/specstudio-skills/spec/features/reviewer-gates?op=request-change) |

**Status:** Approved
**Date:** 2026-05-22
**Owner:** alex
**Source Ideas:** reviewer-gates
**Supersedes:** —
**Grade:** B

## Summary

Defines the canonical reviewer-gates contract: **event-keyed** reviewer lists scoped under a `gates:` block in `specscore.yaml`, with a `type:` discriminator and type-specific fields per reviewer entry. Gates are keyed by the **event/checkpoint they guard** (`gates.<event>`) — both artifact-lifecycle events (e.g., `feature.approved`) and pre-action gate-point events that may fire repeatedly within a run (e.g., `implementation.pre_commit`, `implementation.pre_push`). Pins the schema, dispatch semantics, and verdict contract for the type set (`ai`, `human`, `deterministic`, `auto-approve`), and wires `specstudio:specify` as the first consumer (on `gates.feature.approved`) — replacing its built-in reviewer dispatch and User Review Gate with the new typed-gate model. Carves the Reviewer parts of [`third-party-integration`](../third-party-integration/README.md) out into this Feature.

The gate's verdict currency is an A–F **grade**: per-reviewer `Blocker`/`Advisory` findings are aggregated into a grade, and a gate releases iff `grade ≥ threshold` (configurable, default `B`). Both producer-exit gates and the manual [`score-command`](../score-command/README.md) consume this same grade, so verdict parity is structural. The grade model is designed so the **default threshold reproduces today's binary behavior exactly** (see [the grade design doc](../../research/reviewer-gates-grade-design.md)).

## Problem

`specstudio-skills` today pins reviewer dispatch in three places: [`third-party-integration`](../third-party-integration/README.md) (the `reviewers:` registry in `specscore.yaml`), [`specstudio:specify`](../skills/specify/README.md) (built-in baseline reviewer + extension hook + a separate "User Review Gate"), and [`specstudio:plan`](../skills/plan/README.md) (mirrored shape). Three Features describe the same mechanism inconsistently, none can be configured per pipeline stage, and the abstraction is invisible from the root `README.md`. Two sidekick seeds ([`future-review-skill-could-discover-available-claude-code`](../../ideas/seeds/future-review-skill-could-discover-available-claude-code.md), [`extend-consilium-to-review-regular-specscore-ideas-not-just`](../../ideas/seeds/extend-consilium-to-review-regular-specscore-ideas-not-just.md)) anticipate multi-reviewer dispatch but have no schema to register entries against. The draft [`review`](../skills/review/README.md) Feature explicitly defers naming ("human vs AI") and placement ("one step or per-artifact") to ideation.

The MVP shape pinned in [the source Idea](../../ideas/reviewer-gates.md) — a typed, per-stage `gates:` registry in `specscore.yaml`, AND-composition across reviewer entries, humans-as-typed-reviewer-entry — closes those three loose ends with one schema and one MVP consumer.

## Behavior

### Schema

#### REQ: gates-block-location

`specscore.yaml` MUST support a top-level `gates:` key. Each child key under `gates:` MUST be a **gate-point event identifier** — either a canonical artifact-lifecycle event from [`events.md`](../../../skills/shared/events.md) that the gate guards (e.g., `feature.approved`, `idea.approved`, `plan.approved`) or a canonical pre-action gate-point event (e.g., `implementation.pre_commit`, `implementation.pre_push`; see `gate-point-events-and-multi-fire`). A bare skill/command name (e.g., `specify`) is NOT a valid gate key and MUST be rejected per `migration-to-event-keys`. The block is preserved across SpecScore tooling reads/writes via the [SpecScore Repo Config Feature](https://github.com/specscore/specscore/blob/main/spec/features/repo-config/README.md)'s `unknown-fields-preserved` requirement; no new file convention or dotfile is introduced.

#### REQ: per-gate-shape

Each gate value MUST be an object containing exactly one required field: `reviewers:` — an ordered list. The order of entries in `reviewers:` MUST be preserved as the dispatch order (see `dispatch-serial`). An empty list (`reviewers: []`) is syntactically valid but is refused at consumer load time per `missing-gates-block-refuses`.

### Reviewer entry shape

#### REQ: reviewer-entry-required-fields

Every reviewer entry MUST declare `type:` (string, one of the values in `mvp-type-set`). `name:` (string, lowercase + hyphens) is OPTIONAL; when omitted it defaults to the entry's `type:` value. The *effective* name (declared or defaulted) MUST be unique within the gate's `reviewers:` list — so two entries that share a `type:` on the same gate MUST each declare an explicit `name:` to disambiguate. An entry missing `type:`, or a gate whose effective names collide, MUST be rejected at consumer load time with an error citing this REQ.

#### REQ: mvp-type-set

The type set is exactly `{ai, human, deterministic, auto-approve}`. Any other `type:` value — including the former `noop` (renamed to `auto-approve`) — MUST be rejected at consumer load time with an error pointing at this Feature; consumers MUST NOT silently treat unknown types as `ai`. `ai` and `human` are defined in `ai-entry-shape` / `human-entry-shape`; `deterministic` and `auto-approve` in `deterministic-entry-shape` / `auto-approve-entry-shape`. Further specialized types (e.g., `ux`, `peer-review-bot`, `third-party-ci`) remain deferred to additive revisions — a tool-backed check (lint, security scanner) is expressed as `type: deterministic` with a `run:` command, not as a bespoke type.

#### REQ: ai-entry-shape

`type: ai` entries MUST declare `prompt:` (string, a path resolving to a file inside the repo working tree, expressed relative to repo root; absolute filesystem paths outside the repo and network URLs are forbidden). They MAY declare `model:` (string identifier — opaque to this contract; consumers interpret) and `description:` (string ≤ 200 chars). The prompt file MUST contain an explicit blocker/advisory taxonomy section documenting which finding categories the reviewer treats as `Blocker` vs. `Advisory`; a prompt with no documented taxonomy MUST cause the consumer to reject the registry entry.

#### REQ: human-entry-shape

`type: human` entries MUST NOT declare a `prompt:` field — humans have no programmatic prompt. They MAY declare `min_approvers:` (integer ≥ 1); MVP pins `min_approvers: 1` — values > 1 MUST be rejected at consumer load time with an error pointing at this Feature (see `## Not Doing`). The human's verdict is collected from the same explicit approval-phrase recognizer used by `specstudio:ideate` and `specstudio:specify` for user-approval gates (the recognizer that accepts `approve` / `approved` / `accept` / `accepted` / `lgtm` plus direct semantic equivalents in the user's language as `Approved`, treats vague positive signals as ambiguous, and treats explicit change requests as `Issues Found`).

#### REQ: deterministic-entry-shape

`type: deterministic` entries run a repo-local tool (linter, security scanner, conflict check, etc.) rather than an LLM or a human. They MUST declare `run:` (string — a command resolvable in the repo, e.g. a script path or Make target; network commands are forbidden) and MAY declare `description:`. They MUST NOT declare `prompt:` or `model:`. The verdict MUST be derived deterministically: exit code zero maps to `Approved`; a non-zero exit maps to `Issues Found`, with the tool's diagnostic output captured as `Blocker` finding(s). (A configurable non-exit-code success predicate is out of MVP scope.) Like every reviewer, a `deterministic` entry MUST NOT write to `spec/` artifacts (it is a read-only check per `verdict-contract`); a tool that mutates artifacts is a Producer, not a reviewer.

#### REQ: auto-approve-entry-shape

`type: auto-approve` entries dispatch nothing and always return `Approved` with no findings. They are the explicit auto-approve placeholder that lets an event's gate be configured as "no review at this checkpoint" *without removing the gate key* — so autonomy at a checkpoint is expressed as gate configuration (an `auto-approve` where a `human` would otherwise sit), not as a separate mechanism. An `auto-approve` entry MUST NOT declare `prompt:`, `model:`, or `run:`; it MAY declare `description:` (e.g., the rationale for auto-approving this checkpoint). (This type was formerly named `noop`; the rename makes the auto-approve intent explicit in config.)

#### REQ: gate-point-events-and-multi-fire

Gate keys MAY be **pre-action gate-point events** that occur at execution checkpoints rather than at artifact-lifecycle transitions. The MVP gate-point events are `implementation.pre_commit` (evaluated before each commit a producer makes during an `implement` run) and `implementation.pre_push` (evaluated before a publish/promote). Follow-on consumer Features register additional gate-point events the same way — `specstudio:ship` registers `ship.pre_dispatch` (a single-fire pre-deploy checkpoint) in the catalog and keys its reviewer gate on it. Gate-point events MUST be registered in the canonical [`events.md`](../../../skills/shared/events.md) catalog alongside lifecycle events. A gate keyed on an event that occurs multiple times in one run (e.g., `implementation.pre_commit` firing per commit/batch) MUST be evaluated **independently at each occurrence** — each firing dispatches the gate's reviewers and yields its own verdict; there is no single-shot-per-run assumption. A gate keyed on a once-per-artifact lifecycle event (e.g., `feature.approved`) fires once.

#### REQ: migration-to-event-keys

This revision replaces command/skill-keyed gates (`gates.<skill>`, e.g. `gates.specify`) with event-keyed gates (`gates.<event>`) as a **clean break — no back-compat window**. A consumer that encounters a gate key which is a bare skill/command name rather than a registered event identifier MUST reject it at load time with an error pointing at this Feature and naming the event key to migrate to (e.g., `gates.specify` → `gates.feature.approved`). The repo's own `specscore.yaml` MUST be migrated as part of this Feature's implementation so that `gates.specify` becomes `gates.feature.approved`.

#### REQ: no-untyped-entry

A reviewer entry without an explicit `type:` field MUST be rejected at consumer load time with an error citing this REQ. There is no implicit default for `type:` — entries from any pre-existing flat `reviewers:` registry (e.g., the predecessor shape carried by `third-party-integration` prior to this Feature) that lack a `type:` MUST be rejected with a clear migration message pointing at this Feature.

#### REQ: gate-entry-when-condition

A reviewer entry MAY declare an optional `when:` field that conditions the entry's participation in the gate on the current branch. When present, the entry participates in the gate **only if** the current branch matches; when absent, the entry **always** participates (the unconditioned default — unchanged behavior). The match grammar is **anchored regex** against the current branch name, expressed in the exact form `when: "branch =~ <anchored-regex>"` (e.g., `when: "branch =~ ^(main|master|release/)"`). The `branch =~` prefix is mandatory and the right-hand side is an anchored regular expression matched against the current branch name; a glob alternative is NOT supported. A `when:` value that is not a string of the form `branch =~ <regex>` MUST be rejected at consumer load time with an error citing this REQ. This single grammar is shared by both the gates layer and any autonomy layer that masks gate entries per branch (e.g., [`approval-autonomy`](../approval-autonomy/README.md)'s `branch-mask-via-gate-when`), so the two layers never guess the dialect differently. An entry whose `when:` does not match the current branch does NOT participate: it is neither dispatched nor counted toward the gate's verdict (it contributes no `Blocker`, exactly as if it were absent from the list for that branch). Per-branch autonomy is therefore expressed entirely as a `when:` mask on a gate entry, NOT as autonomy-local config.

### Dispatch and verdict

#### REQ: dispatch-serial

Within a single gate, reviewers MUST be dispatched serially in `reviewers:` list order. Parallel dispatch is out of MVP scope (see `## Not Doing`). The next reviewer's dispatch is invoked only after the previous reviewer's verdict has been collected. A consumer MUST NOT have more than one reviewer dispatch concurrently in flight for the same gate run.

#### REQ: verdict-contract

Every reviewer (regardless of type) MUST resolve to exactly one of two verdicts: `Approved` or `Issues Found`. On `Issues Found`, the reviewer MUST attach a structured findings list, where each finding declares its severity as `Blocker` or `Advisory`. For `type: ai`, the verdict is the subagent's response per the prompt's documented taxonomy. For `type: human`, the verdict is `Approved` on an explicit approval phrase and `Issues Found` (with the user's change request captured as a single `Blocker` finding) on a change request. For `type: ai` reviewers **whose prompt declares the multi-role lenses** (`multi-role-reviewer`), the response additionally carries the per-lens sub-assessment and a single within-band letter. Reviewers whose prompt emits only findings (e.g., the current baseline prompt) are valid and carry no within-band letter; the gate still computes the band deterministically from the `Blocker` count per `### Grade and threshold`. Reviewers MUST NOT write or modify any artifact in `spec/`; a reviewer that attempts a write is a misclassified Producer and the consumer MUST reject the gate.

#### REQ: and-composition

A gate's verdict is the **grade** computed from all reviewers' findings, released iff `grade ≥ threshold` (per `threshold-derived-verdict`). The gate **does NOT early-halt**: every `type: ai` reviewer is always dispatched, so the full `Blocker` union — and therefore the grade — is exact in a single pass. Surfacing every `Blocker` at once (rather than one reviewer per pass) lets the author fix all findings before a single re-run, avoiding fix-one-rerun-fix-next iteration.

Dispatch is **two-phase**:

1. **Automated phase.** Dispatch every `type: ai` reviewer in `reviewers:` list order (serially per `dispatch-serial`) with no early-halt. Collect findings from all of them and compute the **automated grade** from the full `Blocker` union (`grade-aggregation`, `grade-band-mapping`).
2. **Human phase.** Dispatch the `type: human` reviewer(s), in list order, **only if** the automated grade already satisfies `grade ≥ threshold`. A human change request contributes a single `Blocker`, which re-grades the gate (it may drop the grade below the threshold). If the automated grade is below the threshold, the consumer MUST NOT dispatch any `type: human` reviewer — it surfaces every automated-phase `Blocker` so the author can fix them in one pass. (A human MUST NOT be asked to approve an artifact the automated reviewers already failed.)

The gate releases iff the final grade (after any human-phase findings) satisfies `grade ≥ threshold`. Re-dispatch on the next pass is governed by `rerun-policy`. Advisory findings MAY be ignored by the consumer. Consumers MUST NOT silently downgrade a `Blocker` finding to `Advisory` severity and MUST NOT skip a registered `type: ai` reviewer (a `type: human` reviewer is deferred or skipped only by the automated-grade gate in phase 2). At the default threshold `B`, the gate releases iff the aggregated `Blocker` count is zero — identical to the prior binary behavior.

#### REQ: rerun-policy

On `Issues Found`, the consumer MUST address every `Blocker` finding before re-running the gate. On re-run, the consumer MUST re-dispatch every reviewer that previously returned `Issues Found`. Reviewers that previously returned `Approved` MUST be re-dispatched when the fix changes the artifact's structural sections — for SpecScore Feature artifacts these are `## Behavior`, `## Architecture`, and `## Acceptance Criteria`; for other artifact types the structurally-load-bearing sections defined by that artifact's specification. On non-structural fixes (typos, link repair, comment-only changes), previously-Approved reviewers MAY be skipped at the consumer's discretion.

### Grade and threshold

The gate's verdict currency is an A–F **grade**. Per-reviewer findings still use the `Blocker` / `Advisory` severities defined in `verdict-contract`; the gate computes the grade from them and derives the release verdict from a configurable threshold. At the default threshold this reduces exactly to the binary AND-composition above, so existing gates are unaffected.

#### REQ: grade-band-mapping

The gate MUST map an artifact's aggregated findings to one of five whole-letter grades — `A`, `B`, `C`, `D`, `F` (no `+`/`-` variants in MVP). The band is fixed **deterministically by the count of `Blocker` findings** in the aggregated set (the union defined by `grade-aggregation`):

| Aggregated `Blocker` count | Band | Within-band letter |
|---|---|---|
| 0 | pass | `A` or `B` |
| 1 | fail | `C` |
| 2–3 | fail | `D` |
| 4 or more | fail | `F` |

Within the zero-`Blocker` pass band, the runner sets the letter from the reviewers' explicit within-band judgment: the grade is `A` only when at least one reviewer supplies an explicit within-band `A` and no reviewer supplies `B` (worst-wins per `grade-aggregation`); **absent any within-band letter the pass-band grade defaults to `B`**. This default is load-bearing for migration: a findings-only reviewer (no within-band judgment, e.g. the current baseline prompt) therefore yields `B` on zero `Blocker`s, which passes the default threshold `B` exactly as today. `Advisory` findings MAY inform a reviewer's within-band judgment but MUST NOT move the grade out of the pass band (Advisories never fail a gate). The pass/fail band itself MUST depend only on `Blocker` count — never on reviewer judgment — so the band is reproducible across runs holding findings constant; reviewer judgment can only distinguish `A` from `B` within the already-passing band.

#### REQ: threshold-derived-verdict

A gate's release verdict MUST be derived as `Approved` iff `grade ≥ threshold`, where letters are ordered `A > B > C > D > F` and `threshold` is resolved per `threshold-config`. `Issues Found` is the complementary case (`grade < threshold`). The default threshold `B` means the gate releases iff the grade is `A` or `B` — i.e., iff the aggregated `Blocker` count is zero — which is identical to the binary `and-composition` behavior. A threshold of `A` additionally requires the reviewer's within-band judgment to reach `A`; a threshold of `C` (or lower) tolerates the corresponding number of `Blocker` findings.

#### REQ: threshold-config

The Approve threshold MUST be resolvable from `specscore.yaml` in this order: (1) a per-stage `gates.<event>.threshold` value; (2) a top-level `grade.threshold` value; (3) the built-in default `B` when neither is present. A `threshold` value MUST be one of the whole letters `A`, `B`, `C`, `D`, `F`; any other value (including `E`, `+`/`-` variants, or non-letters) MUST be rejected at consumer load time with an error pointing at this Feature. The `gates.<event>.threshold` and top-level `grade.threshold` keys are additive to the existing schema and MUST be preserved across tooling reads/writes per `gates-block-location`'s preservation guarantee.

#### REQ: grade-aggregation

When a gate has multiple reviewers, AND when a single multi-role reviewer reports findings across multiple lenses, the gate MUST aggregate by **worst-wins**: the `Blocker` set is the **union** of all `Blocker` findings across every `type: ai` reviewer and every lens, and the gate grade is computed from that union per `grade-band-mapping`. Because the gate does NOT early-halt (`and-composition`), every `type: ai` reviewer always runs in the automated phase, so this union — and therefore the grade — is always **exact** (no panel truncation). A reviewer's lenses contribute only to the `Blocker` union — they do NOT each emit a separate letter; a multi-role reviewer emits exactly **one** within-band letter representing its overall pass-band judgment (per `multi-role-reviewer`). In the pass band, the lowest within-band letter across reviewers wins, and a reviewer that supplies no within-band letter contributes the default `B` (per `grade-band-mapping`). At the default threshold `B`, any single `Blocker` from any reviewer still blocks the gate (grade ≤ `C` < `B`).

#### REQ: multi-role-reviewer

The **recommended default reviewer shape** for a stage is **one `type: ai` reviewer that evaluates the artifact through multiple lenses** — at minimum Business-Analyst (BA), Developer, and QA. A reviewer that takes this multi-role shape MUST, in addition to the `verdict-contract` findings list, emit a per-lens sub-assessment naming what each lens checked AND exactly **one** within-band letter representing its overall pass-band judgment (one letter for the reviewer, not one per lens — lenses contribute to the `Blocker` union per `grade-aggregation`). When a multi-role reviewer is used, its **BA lens MUST treat "the requirements do not demonstrably address the artifact's stated `## Problem`" as a `Blocker` category**, closing the problem→requirements traceability gap. The lens set is fixed in MVP — per-stage lens configuration is out of scope (see `## Not Doing`).

This shape is the recommended default, not a constraint on every `type: ai` entry: a findings-only prompt (such as the current baseline at `skills/specify/references/reviewer-prompt.md`) remains valid per `verdict-contract` and contributes the default `B` in the pass band. Upgrading the baseline prompt to the multi-role shape is part of the grade increment (see the [Plan](../../plans/reviewer-gates.md)'s grade-increment task). A multi-reviewer **panel** (separate `type: ai` entries per role) remains available by adding entries to `gates.<event>.reviewers`, and the `consilium` skill remains the heavyweight multi-role escape hatch. This REQ governs reviewer-prompt content and output shape; it does not change the entry schema in `ai-entry-shape`.

### Recording the grade

#### REQ: grade-recording

A **producer** consumer that releases a gate MUST record the released grade so the outcome is durable in two places:

1. **Event payload.** The gate-release event the producer emits MUST carry the grade — e.g., `specstudio:specify`'s `feature.approved` payload includes `grade: <letter>` (e.g., a "Feature approved with grade `B`" record).
2. **Artifact metadata.** The producer MUST write the released grade onto the approved artifact as a `**Grade:** <letter>` body-metadata line, added immediately after `**Supersedes:**` (or updated in place if already present), on every gate release. The line reflects the artifact's grade at its most recent approval.

The gate runner itself writes nothing — reviewers are read-only per `verdict-contract`; recording is the producer's action on release. **Signal-only** consumers (e.g., the manual `/score`, which produces no canonical artifact) do NOT record via this REQ — they surface/persist the grade through their own flags (`--save` / `--badge`), owned by [`score-command`](../score-command/README.md).

### `specstudio:specify` wiring

#### REQ: specify-loads-gate

`specstudio:specify` MUST resolve its reviewer list exclusively from `gates.feature.approved.reviewers` in `specscore.yaml`. The skill MUST NOT carry a hardcoded baseline reviewer. The existing baseline-reviewer prompt at `skills/specify/references/reviewer-prompt.md` MUST be referenced from a `gates.feature.approved.reviewers` entry of `type: ai` with that file as the entry's `prompt:` value — making the baseline an opt-in registry entry like any other, not a hidden default.

#### REQ: specify-no-separate-user-gate

The distinct "User Review Gate" step in `specstudio:specify` MUST be removed. The user's approval is collected through a `type: human` entry in `gates.feature.approved.reviewers` exactly like every other reviewer — dispatched serially per `dispatch-serial`, contributing its verdict to AND-composition per `and-composition`. The five REQs in the existing `specify` Feature listed under `specify-feature-revision` are the structural manifestation of this collapse.

#### REQ: missing-gates-block-refuses

When `specscore.yaml` has no top-level `gates:` key, no `gates.feature.approved` sub-key, or `gates.feature.approved.reviewers` is an empty list (`[]`), `specstudio:specify` MUST refuse to run with a clear error pointing at this Feature and recommending the canonical minimal gate configuration (at minimum one `type: human` entry). The skill MUST NOT silently fall back to any prior built-in baseline reviewer and MUST NOT silently fall back to a User Review Gate.

### Carve-out and cleanup

#### REQ: third-party-integration-revision

The [`third-party-integration`](../third-party-integration/README.md) Feature MUST be revised in place as part of this Feature's implementation: (a) the REQs `reviewer-registration-mechanism`, `reviewer-registry-entry-shape`, `reviewer-prompt-location`, `reviewer-contract`, `reviewer-composition`, and `reviewer-no-canonical-writes` are removed; (b) the AC `reviewer-registration-and-composition` is removed; (c) all Reviewer-shape narrative content (including the `### Reviewer shape` section and references to `reviewers:` in `specscore.yaml`) is removed; (d) the `## Interaction with Other Features` table gains a row pointing at this Feature for the Reviewer contract; (e) the path-table row for `spec/reviewers/<name>/` is removed (this Feature's `ai-entry-shape` REQ owns the prompt-path convention); (f) the Feature's remaining scope (Producer and Capability shapes, plus the snippet versioning AC) MUST remain valid and `specscore spec lint` MUST pass after the carve-out.

#### REQ: specify-feature-revision

The [`specify`](../skills/specify/README.md) Feature MUST be revised in place to: (a) remove the REQs `reviewer-subagent-required`, `reviewer-baseline-blockers`, `reviewer-extension-hook`, `reviewer-composition`, and `user-approval-required`; (b) remove or replace the dependent ACs — at minimum `reviewer-then-user` MUST be replaced by a single AC asserting `gates.feature.approved` consumption; (c) the `### Reviewer subagent gate` and `### User Review Gate` topic sections MUST be collapsed into a single `### Reviewer gate` topic that delegates to this Feature via a link; (d) all other REQs and ACs in `specify` remain unchanged.

#### REQ: review-feature-archival

The draft [`review`](../skills/review/README.md) Feature MUST be transitioned to `**Status:** Archived` (via `specscore feature change-status review --to=archived` or equivalent) as part of this Feature's implementation, with the archive reason "Superseded by reviewer-gates — reviews are stage-internal under each producer's gate; no standalone review skill is required." No replacement skill is created.

### Visibility

#### REQ: root-readme-link

The repo-root `README.md` MUST be edited to include a visible link to this Feature. The pipeline overview sentence (today `**ideate ⇒ specify ⇒ plan ⇒ implement ⇒ verify ⇒ recap ⇒ review ⇒ ship**`) MUST be updated to remove the standalone `review` step, since reviews become stage-internal under this Feature. The exact link copy and placement are implementation discretion; the REQ is satisfied iff the root `README.md` contains a link whose href resolves to `spec/features/reviewer-gates/README.md` (or a canonical equivalent), AND the pipeline sentence no longer contains `review` as a discrete step.

#### REQ: skill-doc-cross-links

The skill `skills/specify/SKILL.md`, the Feature `spec/features/skills/specify/README.md`, and the features index `spec/features/README.md` MUST each contain at least one link pointing at this Feature. The features-index entry MUST include a one-line description of the Feature alongside the link.

## Architecture

- **Schema owner.** This Feature owns the canonical `gates:` block schema in `specscore.yaml`. The SpecScore Repo Config Feature's `unknown-fields-preserved` requirement is the load-bearing dependency; no upstream change is required.
- **Consumer (MVP).** `specstudio:specify` — reads `gates.feature.approved.reviewers`, validates each entry's shape per `reviewer-entry-required-fields` / `mvp-type-set` / `ai-entry-shape` / `human-entry-shape` / `no-untyped-entry`, dispatches entries serially per `dispatch-serial`, aggregates verdicts under `and-composition`, and re-runs per `rerun-policy`.
- **Future consumers (out of MVP scope).** `specstudio:plan`, `specstudio:implement`, `specstudio:verify`, `specstudio:recap`. Each will be a separate follow-on Feature; this contract is designed consumer-agnostic so future wiring needs only schema reads, not contract changes.
- **Reviewer dispatch surfaces.** `type: ai` dispatches via the consumer skill's Agent tool with the prompt file as the system prompt. `type: human` dispatches via the consumer skill's existing user-prompt + approval-phrase recognizer.
- **Verdict aggregation.** Stateless per-gate. Two-phase, no early-halt (`and-composition`): all `type: ai` reviewers run → exact `Blocker` union → A–F grade (`grade-aggregation`, `grade-band-mapping`); the `type: human` phase runs only if the automated grade ≥ threshold. The gate releases iff the final `grade ≥ threshold` (`threshold-derived-verdict`). At the default threshold `B` this is identical to "release iff zero `Blocker`s." No persisted state between gate runs; rerun discipline is captured in `rerun-policy`.
- **Outputs.** The gate runner performs no artifact writes and emits no events (reviewers are read-only). The grade is part of the gate's released verdict; per `grade-recording`, a **producer** consumer records it on release — in its own event payload (e.g., `feature.approved` carrying `grade`) and as a `**Grade:**` body-metadata line on the approved artifact. The runner remains a pure contract; recording is the producer's action.

## Interaction with Other Features

| Feature | Relationship |
|---|---|
| [Third-Party Integration](../third-party-integration/README.md) | This Feature carves the Reviewer shape out of `third-party-integration`. Per `third-party-integration-revision`, the Reviewer REQs there are removed; the Producer and Capability shapes remain. |
| [Specify Skill](../skills/specify/README.md) | This Feature's MVP consumer. Per `specify-loads-gate`, `specify-no-separate-user-gate`, and `specify-feature-revision`, the existing Reviewer-subagent and User-Review-Gate REQs in `specify` are replaced by `gates.feature.approved` consumption. |
| [Plan Skill](../skills/plan/README.md) | Not wired in MVP. The existing reviewer-subagent REQs in `plan` remain until a follow-on Feature wires `plan` to consume `gates.plan`. Out of this Feature's scope. |
| [Review Skill (archived by this Feature)](../skills/review/README.md) | Per `review-feature-archival`, the standalone `review` pipeline step is archived in favor of stage-internal reviewer gates. |
| [Score Command](../score-command/README.md) | Consumes this layer's grade + threshold as the manual `/score` surface. The grade is single-sourced here so manual `/score` and the producer-exit gates return identical verdicts (verdict parity). `/score`'s `--save` / `--badge` flags layer on top and are owned there. |
| SpecScore Repo Config | Hosts the new `gates:` extension key and the top-level `grade.threshold` key. Relies on `unknown-fields-preserved`; no change required upstream. |

## Acceptance Criteria

ACs are grouped here with explicit REQ back-references, mirroring sibling Features' style.

### AC: gates-block-preserved (verifies REQ:gates-block-location, REQ:per-gate-shape)

**Given** a `specscore.yaml` containing a top-level `gates:` block with a child key `feature.approved:` (an event identifier) whose value is an object containing a `reviewers:` list of two entries in a specific order,
**When** SpecScore tooling reads and re-writes the file (e.g., `specscore` CLI commands that touch the config),
**Then** the `gates:` block MUST be preserved verbatim on rewrite (per `unknown-fields-preserved`), and consumers MUST be able to resolve `gates.feature.approved.reviewers` to the original ordered list of entries in the original order.

### AC: untyped-entry-refused (verifies REQ:no-untyped-entry, REQ:reviewer-entry-required-fields)

**Given** a `gates.feature.approved.reviewers` list containing an entry with `name:` but no `type:` field,
**When** `specstudio:specify` attempts to load the gate,
**Then** the skill MUST refuse to run with an error citing `no-untyped-entry` and pointing at this Feature, MUST NOT dispatch any reviewer in the gate, and MUST exit non-zero.

### AC: unknown-type-refused (verifies REQ:mvp-type-set)

**Given** a `gates.feature.approved.reviewers` entry with `type: noop` (the former type name, now outside the set `{ai, human, deterministic, auto-approve}`),
**When** `specstudio:specify` attempts to load the gate,
**Then** the skill MUST refuse to run with an error citing `mvp-type-set` and pointing at this Feature, MUST NOT dispatch any reviewer, and MUST exit non-zero.

### AC: name-defaults-to-type (verifies REQ:reviewer-entry-required-fields)

**Given** a `gates.feature.approved.reviewers` entry that declares `type: auto-approve` and omits `name:`,
**When** a consumer loads the gate,
**Then** the entry's effective name MUST be `auto-approve` (its `type:` value), and the gate MUST load without error.

### AC: duplicate-effective-name-refused (verifies REQ:reviewer-entry-required-fields)

**Given** a gate `reviewers:` list with two `type: ai` entries that both omit `name:` (so both default to the effective name `ai`),
**When** a consumer loads the gate,
**Then** the consumer MUST refuse to run with an error citing `reviewer-entry-required-fields` (duplicate effective name) and pointing at this Feature, MUST NOT dispatch any reviewer, and MUST exit non-zero.

### AC: legacy-command-key-rejected (verifies REQ:migration-to-event-keys, REQ:gates-block-location)

**Given** a `specscore.yaml` whose `gates:` block uses the legacy command-keyed form `gates.specify` (a bare skill name) rather than an event key,
**When** a consumer loads the gate,
**Then** it MUST reject the key with an error pointing at this Feature and naming the event key to migrate to (`gates.feature.approved`), MUST NOT dispatch any reviewer, and MUST exit non-zero.

### AC: deterministic-verdict-from-exit (verifies REQ:deterministic-entry-shape)

**Given** a gate `reviewers:` list with a `type: deterministic` entry whose `run:` command exits non-zero,
**When** the gate is evaluated,
**Then** that reviewer's verdict MUST be `Issues Found` with the command's diagnostic output captured as at least one `Blocker` finding, and the gate MUST NOT release at the default threshold `B`; and given the same command exits zero, that reviewer MUST contribute `Approved` with no findings.

### AC: auto-approve-always-approves (verifies REQ:auto-approve-entry-shape)

**Given** a gate `reviewers:` list containing a `type: auto-approve` entry,
**When** the gate is evaluated,
**Then** the `auto-approve` entry MUST return `Approved` with no findings, MUST dispatch nothing (no subagent, no command, no human prompt), and MUST NOT contribute any `Blocker` to the grade.

### AC: when-condition-masks-by-branch (verifies REQ:gate-entry-when-condition)

**Given** a `gates.implementation.pre_commit.reviewers` list with a `type: human` entry carrying `when: "branch =~ ^(main|master|release/)"`,
**When** the gate is evaluated on a feature branch (e.g., `feature/x`) and separately on `main`,
**Then** on the feature branch the human entry does NOT participate — it is neither dispatched nor counted toward the verdict, so the gate releases without asking the human (commits stay autonomous) — and on `main` the human entry DOES participate (the human is asked before commit); a sibling entry with no `when:` participates on both branches. The per-branch behavior comes entirely from the gate-entry `when:` condition, evaluated by the runner against the current branch with the anchored-regex grammar; and a malformed `when:` (not of the form `branch =~ <regex>`) is refused at load time citing `gate-entry-when-condition`.

### AC: pre-commit-gate-fires-per-occurrence (verifies REQ:gate-point-events-and-multi-fire)

**Given** a gate keyed on a multi-occurrence gate-point event (`implementation.pre_commit`) and a run in which that event occurs three times (simulated via the gate-runner harness),
**When** the runner processes the run,
**Then** the gate MUST be evaluated three times — once per occurrence — each evaluation dispatching the gate's reviewers and producing its own independent verdict, with no single-shot-per-run caching. (Wiring an actual `implement` run to fire this event is a follow-on Feature; this AC fixes the runner's per-occurrence contract.)

### AC: ai-entry-shape-violations-refused (verifies REQ:ai-entry-shape)

**Given** a `gates.feature.approved.reviewers` entry of `type: ai` in any of the following invalid shapes — (a) missing the `prompt:` field, (b) `prompt:` resolves to a path outside the repo working tree, (c) `prompt:` resolves to a file inside the repo whose contents contain no documented blocker/advisory taxonomy section,
**When** `specstudio:specify` attempts to load the gate,
**Then** the skill MUST refuse to run with an error citing `ai-entry-shape` and pointing at this Feature, MUST NOT dispatch the entry, and MUST exit non-zero.

### AC: human-entry-min-approvers-cap (verifies REQ:human-entry-shape)

**Given** a `gates.feature.approved.reviewers` entry of `type: human` with `min_approvers: 2`,
**When** `specstudio:specify` attempts to load the gate,
**Then** the skill MUST refuse to run with an error citing `human-entry-shape`'s MVP `min_approvers: 1` cap and pointing at this Feature, MUST NOT dispatch the human entry, and MUST exit non-zero.

### AC: human-entry-rejects-prompt (verifies REQ:human-entry-shape)

**Given** a `gates.feature.approved.reviewers` entry of `type: human` that declares a `prompt:` field,
**When** `specstudio:specify` attempts to load the gate,
**Then** the skill MUST refuse to run with an error citing `human-entry-shape`'s prohibition on `prompt:` for human entries, MUST NOT dispatch the entry, and MUST exit non-zero.

### AC: serial-dispatch-observed (verifies REQ:dispatch-serial)

**Given** a `gates.feature.approved.reviewers` list with three entries (two `ai` plus one `human`) where both `ai` reviewers return `Approved` (so the human phase runs per `and-composition`), and instrumentation that records dispatch start/end timestamps per entry,
**When** `specstudio:specify` runs through the gate,
**Then** at no point during the run are two reviewer dispatches concurrently in flight, and the recorded dispatch start order matches the list order exactly.

### AC: and-composition-blocks-on-any-issues-found (verifies REQ:and-composition, REQ:verdict-contract)

**Given** a `gates.feature.approved.reviewers` list with two `ai` entries followed by one `human` entry, where the first `ai` entry returns `Approved` and the second `ai` entry returns `Issues Found` with one `Blocker` finding,
**When** `specstudio:specify` runs through the gate,
**Then** both `ai` entries MUST be dispatched (the gate does NOT early-halt), the automated grade MUST be `C` (one `Blocker`), the gate MUST NOT release, the skill MUST surface the `Blocker` finding to the user, the `human` entry MUST NOT be dispatched (the automated grade `C` is below the default threshold `B`, so the human phase is skipped), and the skill MUST NOT emit `feature.approved`.

### AC: rerun-policy-applies-on-structural-fix (verifies REQ:rerun-policy)

**Given** a gate run in which reviewer A (`type: ai`) returned `Approved`, reviewer B (`type: ai`) returned `Issues Found`, and the user has applied a fix that modifies the Feature's `## Behavior` section,
**When** `specstudio:specify` re-runs the gate after the fix,
**Then** both reviewer A (because the fix touched a structural section) and reviewer B (because it previously returned `Issues Found`) MUST be re-dispatched in list order before the gate verdict is re-evaluated.

### AC: specify-loads-gate-not-builtin (verifies REQ:specify-loads-gate)

**Given** a `gates.feature.approved.reviewers` list whose first entry is `type: ai` with `prompt: skills/specify/references/reviewer-prompt.md` (the prior baseline path) and whose second entry is `type: human`,
**When** `specstudio:specify` runs and successfully passes the gate,
**Then** the skill MUST have dispatched exactly the two listed entries in list order, MUST NOT have additionally dispatched any reviewer hardcoded inside the skill's own logic, and the resulting verdict-aggregation MUST be composed from the registry entries alone.

### AC: missing-gates-block-refuses-with-error (verifies REQ:missing-gates-block-refuses)

**Given** a `specscore.yaml` in any of the following states — (a) no top-level `gates:` key, (b) `gates:` present but no `gates.feature.approved` sub-key, (c) `gates.feature.approved.reviewers: []`,
**When** `specstudio:specify` is invoked,
**Then** the skill MUST refuse to run, MUST print an error pointing at this Feature and recommending a minimal configuration (at minimum one `type: human` entry), MUST NOT dispatch any reviewer, MUST NOT write or modify any artifact, and MUST exit non-zero.

### AC: third-party-integration-revised (verifies REQ:third-party-integration-revision)

**Given** this Feature is approved and its implementation has run,
**When** a downstream consumer reads `spec/features/third-party-integration/README.md`,
**Then** the file MUST NOT contain any of the six removed REQ slugs (`reviewer-registration-mechanism`, `reviewer-registry-entry-shape`, `reviewer-prompt-location`, `reviewer-contract`, `reviewer-composition`, `reviewer-no-canonical-writes`), MUST NOT contain the `reviewer-registration-and-composition` AC, MUST contain at least one link to this Feature in the `## Interaction with Other Features` table, and MUST pass `specscore spec lint`.

### AC: specify-feature-revised (verifies REQ:specify-feature-revision)

**Given** this Feature is approved and its implementation has run,
**When** a downstream consumer reads `spec/features/skills/specify/README.md`,
**Then** the file MUST NOT contain any of the five removed REQ slugs (`reviewer-subagent-required`, `reviewer-baseline-blockers`, `reviewer-extension-hook`, `reviewer-composition`, `user-approval-required`), MUST contain a single `### Reviewer gate` topic that delegates to this Feature via a link, MUST contain a replacement AC that asserts `gates.feature.approved` consumption, and MUST pass `specscore spec lint`.

### AC: review-feature-archived (verifies REQ:review-feature-archival)

**Given** this Feature is approved and its implementation has run,
**When** a downstream consumer reads `spec/features/skills/review/README.md`,
**Then** the file's `**Status:**` body-metadata line MUST be `Archived`, the file MUST contain an `**Archive Reason:**` line citing supersession by this Feature, and `specscore spec lint` MUST pass.

### AC: root-readme-link-present (verifies REQ:root-readme-link)

**Given** this Feature is approved and its implementation has run,
**When** a downstream consumer reads the repo-root `README.md`,
**Then** the file MUST contain at least one link whose href resolves to `spec/features/reviewer-gates/README.md` (or a canonical equivalent), AND the pipeline-overview sentence MUST NOT contain `review` as a discrete pipeline step.

### AC: skill-doc-cross-links-present (verifies REQ:skill-doc-cross-links)

**Given** this Feature is approved and its implementation has run,
**When** a downstream consumer reads `skills/specify/SKILL.md`, `spec/features/skills/specify/README.md`, and `spec/features/README.md`,
**Then** each of the three files MUST contain at least one link pointing at this Feature's `README.md`, and the features-index file (`spec/features/README.md`) MUST include a row with a one-line description of this Feature.

### AC: grade-band-by-blocker-count (verifies REQ:grade-band-mapping)

**Given** four gate runs whose aggregated findings contain exactly 0, 1, 3, and 4 `Blocker` findings respectively,
**When** the gate computes the grade for each,
**Then** the 0-Blocker run MUST land in the pass band (`A` or `B`), the 1-Blocker run MUST be `C`, the 3-Blocker run MUST be `D`, and the 4-Blocker run MUST be `F`; and the pass/fail band MUST be identical across repeated runs holding the findings constant (no dependence on reviewer judgment).

### AC: threshold-default-reproduces-today (verifies REQ:threshold-config, REQ:threshold-derived-verdict, REQ:grade-aggregation)

**Given** a repo with no `gates.<event>.threshold` and no top-level `grade.threshold`, using the existing reviewer prompt,
**When** a gate runs on an artifact whose aggregated findings contain zero `Blocker`s, and separately on an artifact with one or more `Blocker`s,
**Then** the default threshold `B` MUST be applied, the zero-`Blocker` artifact MUST release (`Approved`), and the `Blocker`-bearing artifact MUST NOT release (`Issues Found`) — identical to the pre-grade `and-composition` behavior.

### AC: threshold-resolution-order (verifies REQ:threshold-config)

**Given** a `specscore.yaml` with top-level `grade.threshold: C` and `gates.feature.approved.threshold: B`,
**When** a consumer resolves the threshold for the `specify` stage and for a second stage that declares no per-stage `threshold`,
**Then** the `specify` stage MUST resolve to `B` (per-stage overrides top-level), the second stage MUST resolve to `C` (top-level default), and a repo with neither key MUST resolve to `B` (built-in default).

### AC: invalid-threshold-refused (verifies REQ:threshold-config)

**Given** a `gates.feature.approved.threshold: E` (a value outside the allowed set `{A, B, C, D, F}`),
**When** `specstudio:specify` attempts to load the gate,
**Then** the skill MUST refuse to run with an error citing `threshold-config` and pointing at this Feature, MUST NOT dispatch any reviewer, and MUST exit non-zero.

### AC: lenient-threshold-tolerates-blocker (verifies REQ:threshold-derived-verdict)

**Given** `gates.feature.approved.threshold: C` and an artifact whose aggregated findings contain exactly one `Blocker` (grade `C`),
**When** the gate computes the verdict,
**Then** the gate MUST release (`Approved`) because `C ≥ C`; and the same artifact under the default threshold `B` MUST NOT release.

### AC: worst-wins-union-across-reviewers (verifies REQ:grade-aggregation)

**Given** a two-`ai`-reviewer panel where reviewer A reports zero `Blocker`s and reviewer B reports one `Blocker`,
**When** the gate aggregates the verdicts,
**Then** the aggregated `Blocker` union MUST be 1, the grade MUST be `C`, and the gate MUST NOT release at the default threshold `B`.

### AC: within-band-letter-derivation (verifies REQ:grade-band-mapping, REQ:grade-aggregation)

**Given** three zero-`Blocker` pass-band gate runs — (a) a single reviewer supplies within-band letter `A`; (b) one reviewer supplies `A` and another supplies `B`; (c) a findings-only reviewer supplies no within-band letter,
**When** the gate computes the grade,
**Then** the grade MUST be `A` in (a), `B` in (b) (lowest-wins across reviewers), and `B` in (c) (default when no letter is supplied); and a configured threshold of `A` MUST release only in case (a), while the default threshold `B` MUST release in all three.

### AC: grade-recorded-on-release (verifies REQ:grade-recording)

**Given** a Feature whose `specstudio:specify` gate releases with grade `B` (zero `Blocker`s, no within-band `A`),
**When** the gate releases,
**Then** `specstudio:specify` MUST emit a `feature.approved` event whose payload includes `grade: B`, AND the Feature's `README.md` MUST contain a `**Grade:** B` body-metadata line immediately after `**Supersedes:**` (added if absent, updated in place if already present), AND `specscore spec lint` MUST pass with that line present.

### AC: ba-lens-problem-traceability-blocker (verifies REQ:multi-role-reviewer)

**Given** a Feature whose requirements are internally consistent but do not demonstrably address its stated `## Problem`, reviewed by the default multi-role reviewer,
**When** the BA lens evaluates the artifact,
**Then** the reviewer MUST emit a `Blocker` finding under the BA lens for problem→requirements traceability, the aggregated `Blocker` count MUST be at least 1, and the gate MUST NOT release at the default threshold `B`.

## Rehearse Integration

Every AC above is testable via filesystem fixtures (mock `specscore.yaml` configurations, mock reviewer prompts, mock subagent verdicts) and either consumer-skill instrumentation (dispatch order, refusal exit codes, verdict-aggregation outcome) or grep-style assertions on the revised/archived/linked files. The non-testable cases (the `ai` reviewer subagent's judgment quality, the human's actual real-world approval-phrase usage) are validated at the assumption-validation layer of the source Idea, not as Rehearse scenarios.

Rehearse stubs for each AC are scaffolded at `_tests/<ac-slug>.md` with `**Status:** pending`; authoring scenario steps follows the implementation plan.

## Not Doing

Inherited from the source Idea and pinned here:

- **Reviewer types beyond `ai`, `human`, `deterministic`, `auto-approve`** — bespoke types like `ux`, `peer-review-bot`, `third-party-ci` are deferred. Tool-backed checks (lint, security scanners) are NOT new types — they are `type: deterministic` with a `run:` command. The type set rejects unknown values; future additive revisions add types one at a time as real consumers ship.
- **`min_approvers > 1` for `type: human`** — pinned to 1 in MVP. Multi-approver workflows are deferred until a real need surfaces.
- **Wiring `plan`, `implement`, `verify`, `recap` into `gates`** — each is a separate follow-on Feature with its own status-transition and event-emission specifics.
- **Auto-skip rule** — a `type: human` reviewer auto-passed when preceding `ai` reviewers all return `Approved` is explicitly out of scope. Deferred until at least one gate runs in real dogfood and we have signal on false-positive risk.
- **Parallel reviewer dispatch** — MVP is serial, mirroring `verify`'s MVP discipline. Parallelism lands in a follow-on Idea once typical reviewer counts and token-burst behavior are observed.
- **Skill-discovery for reviewer entries** — the `future-review-skill-could-discover-available-claude-code` seed is separable; the entry shape can grow a `skill:` field additively later.
- **Reviewer prompt-pack publishing / cross-repo prompt distribution** — prompts stay repo-local for MVP.
- **Non-catalog gate-point events** — gate keys MUST be registered events (lifecycle or the MVP gate-points `implementation.pre_commit`/`pre_push`). Inventing arbitrary gate-point identifiers, and wiring gate-points for stages other than `implement` (e.g., a `plan.pre_commit`), are deferred to the follow-on Features that wire those consumers.
- **`gates:` configuration validation as a `specscore` CLI lint rule** — schema lint of `gates:` entries is deferred. Consumers (MVP: `specstudio:specify`) own load-time validation. A future `specscore config validate` (or a lint rule) MAY subsume.
- **Promotion of the schema into the `specscore` repo** — the `gates:` schema lives in this Feature in `specstudio-skills` for MVP. Promoting to the canonical SpecScore Repo Config Feature happens once a second consumer ships.
- **`+`/`-` letter grades** — whole letters `A`–`F` only in MVP; half-step precision (`B+`, `B-`) and the comparison rules it needs are deferred.
- **Per-stage lens-set configuration** — the BA/dev/QA lens set in `multi-role-reviewer` is fixed in MVP. Making the lens set configurable per stage is deferred; a panel of separate reviewer entries already provides per-role flexibility.
- **Severity-weighted fail band** — the C/D/F mapping is by `Blocker` count (`grade-band-mapping`); weighting `Blocker`s by a sub-severity rank is deferred to avoid introducing a new severity taxonomy.
- **Numeric or per-axis published sub-scores** — the per-lens sub-assessment informs the within-band letter only; emitting a numeric score or a persisted per-axis breakdown is deferred (the A–F letter is the currency).
- **Wiring the grade into `plan`/`implement`/`verify`/`recap` gates** — the grade contract is consumer-agnostic, but only `specstudio:specify` (and the manual `/score`) consume it in MVP; other stages wire in their own follow-on Features.

## Open Questions

- Canonical wording of the migration error message emitted when a consumer encounters a legacy untyped `reviewers:` entry (per `no-untyped-entry`) — deferred to implementation; the REQ pins the requirement, not the copy.
- Whether a `description:` field on `type: human` entries adds enough value at MVP to mandate. Currently the schema permits but does not require it; implementation may add usage examples without revising this Feature.
- **Grade as verdict currency** — now specified in `### Grade and threshold` (`grade-band-mapping`, `threshold-derived-verdict`, `threshold-config`, `grade-aggregation`, `multi-role-reviewer`), per [the grade design doc](../../research/reviewer-gates-grade-design.md). Resolved sub-decisions: fail-band by `Blocker` count (C=1, D=2–3, F=4+); fixed BA/dev/QA lens set; whole-letter grades only. Remaining: only `specstudio:specify` and the manual `/score` consume the grade in MVP — wiring other stages is deferred (see `## Not Doing`).

---
*This document follows the https://specscore.md/feature-specification*
