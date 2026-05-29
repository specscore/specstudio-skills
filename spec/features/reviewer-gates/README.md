# Feature: Reviewer Gates

> [SpecScore.**Studio**](https://specscore.studio): | [Explore](https://specscore.studio/app/github.com/specscore/specstudio-skills/spec/features/reviewer-gates?op=explore) | [Edit](https://specscore.studio/app/github.com/specscore/specstudio-skills/spec/features/reviewer-gates?op=edit) | [Ask question](https://specscore.studio/app/github.com/specscore/specstudio-skills/spec/features/reviewer-gates?op=ask) | [Request change](https://specscore.studio/app/github.com/specscore/specstudio-skills/spec/features/reviewer-gates?op=request-change) |

**Status:** Approved
**Date:** 2026-05-22
**Owner:** alex
**Source Ideas:** reviewer-gates
**Supersedes:** —

## Summary

Defines the canonical reviewer-gates contract: per-stage reviewer lists scoped under a `gates:` block in `specscore.yaml`, with a `type:` discriminator and type-specific fields per reviewer entry. Pins the schema, dispatch semantics, and verdict contract for the MVP type set (`ai`, `human`), and wires `specstudio:specify` as the first consumer — replacing its built-in reviewer dispatch and User Review Gate with the new typed-gate model. Carves the Reviewer parts of [`third-party-integration`](../third-party-integration/README.md) out into this Feature.

## Problem

`specstudio-skills` today pins reviewer dispatch in three places: [`third-party-integration`](../third-party-integration/README.md) (the `reviewers:` registry in `specscore.yaml`), [`specstudio:specify`](../skills/specify/README.md) (built-in baseline reviewer + extension hook + a separate "User Review Gate"), and [`specstudio:plan`](../skills/plan/README.md) (mirrored shape). Three Features describe the same mechanism inconsistently, none can be configured per pipeline stage, and the abstraction is invisible from the root `README.md`. Two sidekick seeds ([`future-review-skill-could-discover-available-claude-code`](../../ideas/seeds/future-review-skill-could-discover-available-claude-code.md), [`extend-consilium-to-review-regular-specscore-ideas-not-just`](../../ideas/seeds/extend-consilium-to-review-regular-specscore-ideas-not-just.md)) anticipate multi-reviewer dispatch but have no schema to register entries against. The draft [`review`](../skills/review/README.md) Feature explicitly defers naming ("human vs AI") and placement ("one step or per-artifact") to ideation.

The MVP shape pinned in [the source Idea](../../ideas/reviewer-gates.md) — a typed, per-stage `gates:` registry in `specscore.yaml`, AND-composition across reviewer entries, humans-as-typed-reviewer-entry — closes those three loose ends with one schema and one MVP consumer.

## Behavior

### Schema

#### REQ: gates-block-location

`specscore.yaml` MUST support a top-level `gates:` key. Each child key under `gates:` MUST match a SpecStudio skill's bare name (e.g., `specify`, `plan`, `implement`) — plugin-namespace prefixes such as `specstudio:specify` are NOT allowed in MVP (see `## Not Doing`). The block is preserved across SpecScore tooling reads/writes via the [SpecScore Repo Config Feature](https://github.com/specscore/specscore/blob/main/spec/features/repo-config/README.md)'s `unknown-fields-preserved` requirement; no new file convention or dotfile is introduced.

#### REQ: per-gate-shape

Each gate value MUST be an object containing exactly one required field: `reviewers:` — an ordered list. The order of entries in `reviewers:` MUST be preserved as the dispatch order (see `dispatch-serial`). An empty list (`reviewers: []`) is syntactically valid but is refused at consumer load time per `missing-gates-block-refuses`.

### Reviewer entry shape

#### REQ: reviewer-entry-required-fields

Every reviewer entry MUST declare `name:` (string, unique within the gate's `reviewers:` list, lowercase + hyphens) and `type:` (string, one of the values in `mvp-type-set`). Entries missing `name:` or `type:` MUST be rejected at consumer load time with an error citing this REQ.

#### REQ: mvp-type-set

The MVP type set is exactly `{ai, human}`. Any other `type:` value MUST be rejected at consumer load time with an error pointing at this Feature. Extension to additional types (e.g., `lint`, `security`, `ux`, `peer-review-bot`, `third-party-ci`) is explicitly deferred to additive revisions of this Feature; consumers MUST NOT silently treat unknown types as `ai`.

#### REQ: ai-entry-shape

`type: ai` entries MUST declare `prompt:` (string, a path resolving to a file inside the repo working tree, expressed relative to repo root; absolute filesystem paths outside the repo and network URLs are forbidden). They MAY declare `model:` (string identifier — opaque to this contract; consumers interpret) and `description:` (string ≤ 200 chars). The prompt file MUST contain an explicit blocker/advisory taxonomy section documenting which finding categories the reviewer treats as `Blocker` vs. `Advisory`; a prompt with no documented taxonomy MUST cause the consumer to reject the registry entry.

#### REQ: human-entry-shape

`type: human` entries MUST NOT declare a `prompt:` field — humans have no programmatic prompt. They MAY declare `min_approvers:` (integer ≥ 1); MVP pins `min_approvers: 1` — values > 1 MUST be rejected at consumer load time with an error pointing at this Feature (see `## Not Doing`). The human's verdict is collected from the same explicit approval-phrase recognizer used by `specstudio:ideate` and `specstudio:specify` for user-approval gates (the recognizer that accepts `approve` / `approved` / `accept` / `accepted` / `lgtm` plus direct semantic equivalents in the user's language as `Approved`, treats vague positive signals as ambiguous, and treats explicit change requests as `Issues Found`).

#### REQ: no-untyped-entry

A reviewer entry without an explicit `type:` field MUST be rejected at consumer load time with an error citing this REQ. There is no implicit default for `type:` — entries from any pre-existing flat `reviewers:` registry (e.g., the predecessor shape carried by `third-party-integration` prior to this Feature) that lack a `type:` MUST be rejected with a clear migration message pointing at this Feature.

### Dispatch and verdict

#### REQ: dispatch-serial

Within a single gate, reviewers MUST be dispatched serially in `reviewers:` list order. Parallel dispatch is out of MVP scope (see `## Not Doing`). The next reviewer's dispatch is invoked only after the previous reviewer's verdict has been collected. A consumer MUST NOT have more than one reviewer dispatch concurrently in flight for the same gate run.

#### REQ: verdict-contract

Every reviewer (regardless of type) MUST resolve to exactly one of two verdicts: `Approved` or `Issues Found`. On `Issues Found`, the reviewer MUST attach a structured findings list, where each finding declares its severity as `Blocker` or `Advisory`. For `type: ai`, the verdict is the subagent's response per the prompt's documented taxonomy. For `type: human`, the verdict is `Approved` on an explicit approval phrase and `Issues Found` (with the user's change request captured as a single `Blocker` finding) on a change request. Reviewers MUST NOT write or modify any artifact in `spec/`; a reviewer that attempts a write is a misclassified Producer and the consumer MUST reject the gate.

#### REQ: and-composition

A gate MUST release only when every reviewer in its `reviewers:` list returns `Approved`. Any single `Issues Found` from any reviewer MUST block the gate. Within a single gate pass, on the first `Issues Found` verdict the consumer MUST halt — subsequent reviewers in the list MUST NOT be dispatched in that pass. Halting after the first failure is mandatory (not an optimization): it prevents wasting the user's time on later reviewers (notably `type: human`) when the gate is already known to fail, and re-dispatch on the next pass is governed by `rerun-policy`. Advisory findings MAY be ignored by the consumer. Consumers MUST NOT silently downgrade a `Blocker` finding to `Advisory` severity and MUST NOT skip a registered reviewer.

#### REQ: rerun-policy

On `Issues Found`, the consumer MUST address every `Blocker` finding before re-running the gate. On re-run, the consumer MUST re-dispatch every reviewer that previously returned `Issues Found`. Reviewers that previously returned `Approved` MUST be re-dispatched when the fix changes the artifact's structural sections — for SpecScore Feature artifacts these are `## Behavior`, `## Architecture`, and `## Acceptance Criteria`; for other artifact types the structurally-load-bearing sections defined by that artifact's specification. On non-structural fixes (typos, link repair, comment-only changes), previously-Approved reviewers MAY be skipped at the consumer's discretion.

### `specstudio:specify` wiring

#### REQ: specify-loads-gate

`specstudio:specify` MUST resolve its reviewer list exclusively from `gates.specify.reviewers` in `specscore.yaml`. The skill MUST NOT carry a hardcoded baseline reviewer. The existing baseline-reviewer prompt at `skills/specify/references/reviewer-prompt.md` MUST be referenced from a `gates.specify.reviewers` entry of `type: ai` with that file as the entry's `prompt:` value — making the baseline an opt-in registry entry like any other, not a hidden default.

#### REQ: specify-no-separate-user-gate

The distinct "User Review Gate" step in `specstudio:specify` MUST be removed. The user's approval is collected through a `type: human` entry in `gates.specify.reviewers` exactly like every other reviewer — dispatched serially per `dispatch-serial`, contributing its verdict to AND-composition per `and-composition`. The five REQs in the existing `specify` Feature listed under `specify-feature-revision` are the structural manifestation of this collapse.

#### REQ: missing-gates-block-refuses

When `specscore.yaml` has no top-level `gates:` key, no `gates.specify` sub-key, or `gates.specify.reviewers` is an empty list (`[]`), `specstudio:specify` MUST refuse to run with a clear error pointing at this Feature and recommending the canonical minimal gate configuration (at minimum one `type: human` entry). The skill MUST NOT silently fall back to any prior built-in baseline reviewer and MUST NOT silently fall back to a User Review Gate.

### Carve-out and cleanup

#### REQ: third-party-integration-revision

The [`third-party-integration`](../third-party-integration/README.md) Feature MUST be revised in place as part of this Feature's implementation: (a) the REQs `reviewer-registration-mechanism`, `reviewer-registry-entry-shape`, `reviewer-prompt-location`, `reviewer-contract`, `reviewer-composition`, and `reviewer-no-canonical-writes` are removed; (b) the AC `reviewer-registration-and-composition` is removed; (c) all Reviewer-shape narrative content (including the `### Reviewer shape` section and references to `reviewers:` in `specscore.yaml`) is removed; (d) the `## Interaction with Other Features` table gains a row pointing at this Feature for the Reviewer contract; (e) the path-table row for `spec/reviewers/<name>/` is removed (this Feature's `ai-entry-shape` REQ owns the prompt-path convention); (f) the Feature's remaining scope (Producer and Capability shapes, plus the snippet versioning AC) MUST remain valid and `specscore spec lint` MUST pass after the carve-out.

#### REQ: specify-feature-revision

The [`specify`](../skills/specify/README.md) Feature MUST be revised in place to: (a) remove the REQs `reviewer-subagent-required`, `reviewer-baseline-blockers`, `reviewer-extension-hook`, `reviewer-composition`, and `user-approval-required`; (b) remove or replace the dependent ACs — at minimum `reviewer-then-user` MUST be replaced by a single AC asserting `gates.specify` consumption; (c) the `### Reviewer subagent gate` and `### User Review Gate` topic sections MUST be collapsed into a single `### Reviewer gate` topic that delegates to this Feature via a link; (d) all other REQs and ACs in `specify` remain unchanged.

#### REQ: review-feature-archival

The draft [`review`](../skills/review/README.md) Feature MUST be transitioned to `**Status:** Archived` (via `specscore feature change-status review --to=archived` or equivalent) as part of this Feature's implementation, with the archive reason "Superseded by reviewer-gates — reviews are stage-internal under each producer's gate; no standalone review skill is required." No replacement skill is created.

### Visibility

#### REQ: root-readme-link

The repo-root `README.md` MUST be edited to include a visible link to this Feature. The pipeline overview sentence (today `**ideate ⇒ specify ⇒ plan ⇒ implement ⇒ verify ⇒ recap ⇒ review ⇒ ship**`) MUST be updated to remove the standalone `review` step, since reviews become stage-internal under this Feature. The exact link copy and placement are implementation discretion; the REQ is satisfied iff the root `README.md` contains a link whose href resolves to `spec/features/reviewer-gates/README.md` (or a canonical equivalent), AND the pipeline sentence no longer contains `review` as a discrete step.

#### REQ: skill-doc-cross-links

The skill `skills/specify/SKILL.md`, the Feature `spec/features/skills/specify/README.md`, and the features index `spec/features/README.md` MUST each contain at least one link pointing at this Feature. The features-index entry MUST include a one-line description of the Feature alongside the link.

## Architecture

- **Schema owner.** This Feature owns the canonical `gates:` block schema in `specscore.yaml`. The SpecScore Repo Config Feature's `unknown-fields-preserved` requirement is the load-bearing dependency; no upstream change is required.
- **Consumer (MVP).** `specstudio:specify` — reads `gates.specify.reviewers`, validates each entry's shape per `reviewer-entry-required-fields` / `mvp-type-set` / `ai-entry-shape` / `human-entry-shape` / `no-untyped-entry`, dispatches entries serially per `dispatch-serial`, aggregates verdicts under `and-composition`, and re-runs per `rerun-policy`.
- **Future consumers (out of MVP scope).** `specstudio:plan`, `specstudio:implement`, `specstudio:verify`, `specstudio:recap`. Each will be a separate follow-on Feature; this contract is designed consumer-agnostic so future wiring needs only schema reads, not contract changes.
- **Reviewer dispatch surfaces.** `type: ai` dispatches via the consumer skill's Agent tool with the prompt file as the system prompt. `type: human` dispatches via the consumer skill's existing user-prompt + approval-phrase recognizer.
- **Verdict aggregation.** Stateless per-gate. The gate's verdict is `Approved` iff every reviewer's last verdict in the current run is `Approved`. No persisted state between gate runs; rerun discipline is captured in `rerun-policy`.
- **Outputs.** This Feature defines no artifact writes and no new events. Consumer skills emit their own events (e.g., `feature.approved`); this Feature is purely a contract.

## Interaction with Other Features

| Feature | Relationship |
|---|---|
| [Third-Party Integration](../third-party-integration/README.md) | This Feature carves the Reviewer shape out of `third-party-integration`. Per `third-party-integration-revision`, the Reviewer REQs there are removed; the Producer and Capability shapes remain. |
| [Specify Skill](../skills/specify/README.md) | This Feature's MVP consumer. Per `specify-loads-gate`, `specify-no-separate-user-gate`, and `specify-feature-revision`, the existing Reviewer-subagent and User-Review-Gate REQs in `specify` are replaced by `gates.specify` consumption. |
| [Plan Skill](../skills/plan/README.md) | Not wired in MVP. The existing reviewer-subagent REQs in `plan` remain until a follow-on Feature wires `plan` to consume `gates.plan`. Out of this Feature's scope. |
| [Review Skill (archived by this Feature)](../skills/review/README.md) | Per `review-feature-archival`, the standalone `review` pipeline step is archived in favor of stage-internal reviewer gates. |
| SpecScore Repo Config | Hosts the new `gates:` extension key. Relies on `unknown-fields-preserved`; no change required upstream. |

## Acceptance Criteria

ACs are grouped here with explicit REQ back-references, mirroring sibling Features' style.

### AC: gates-block-preserved (verifies REQ:gates-block-location, REQ:per-gate-shape)

**Given** a `specscore.yaml` containing a top-level `gates:` block with a child key `specify:` whose value is an object containing a `reviewers:` list of two entries in a specific order,
**When** SpecScore tooling reads and re-writes the file (e.g., `specscore` CLI commands that touch the config),
**Then** the `gates:` block MUST be preserved verbatim on rewrite (per `unknown-fields-preserved`), and consumers MUST be able to resolve `gates.specify.reviewers` to the original ordered list of entries in the original order.

### AC: untyped-entry-refused (verifies REQ:no-untyped-entry, REQ:reviewer-entry-required-fields)

**Given** a `gates.specify.reviewers` list containing an entry with `name:` but no `type:` field,
**When** `specstudio:specify` attempts to load the gate,
**Then** the skill MUST refuse to run with an error citing `no-untyped-entry` and pointing at this Feature, MUST NOT dispatch any reviewer in the gate, and MUST exit non-zero.

### AC: unknown-type-refused (verifies REQ:mvp-type-set)

**Given** a `gates.specify.reviewers` entry with `type: security` (a type outside the MVP set `{ai, human}`),
**When** `specstudio:specify` attempts to load the gate,
**Then** the skill MUST refuse to run with an error citing `mvp-type-set` and pointing at this Feature, MUST NOT dispatch any reviewer, and MUST exit non-zero.

### AC: ai-entry-shape-violations-refused (verifies REQ:ai-entry-shape)

**Given** a `gates.specify.reviewers` entry of `type: ai` in any of the following invalid shapes — (a) missing the `prompt:` field, (b) `prompt:` resolves to a path outside the repo working tree, (c) `prompt:` resolves to a file inside the repo whose contents contain no documented blocker/advisory taxonomy section,
**When** `specstudio:specify` attempts to load the gate,
**Then** the skill MUST refuse to run with an error citing `ai-entry-shape` and pointing at this Feature, MUST NOT dispatch the entry, and MUST exit non-zero.

### AC: human-entry-min-approvers-cap (verifies REQ:human-entry-shape)

**Given** a `gates.specify.reviewers` entry of `type: human` with `min_approvers: 2`,
**When** `specstudio:specify` attempts to load the gate,
**Then** the skill MUST refuse to run with an error citing `human-entry-shape`'s MVP `min_approvers: 1` cap and pointing at this Feature, MUST NOT dispatch the human entry, and MUST exit non-zero.

### AC: human-entry-rejects-prompt (verifies REQ:human-entry-shape)

**Given** a `gates.specify.reviewers` entry of `type: human` that declares a `prompt:` field,
**When** `specstudio:specify` attempts to load the gate,
**Then** the skill MUST refuse to run with an error citing `human-entry-shape`'s prohibition on `prompt:` for human entries, MUST NOT dispatch the entry, and MUST exit non-zero.

### AC: serial-dispatch-observed (verifies REQ:dispatch-serial)

**Given** a `gates.specify.reviewers` list with three entries (two `ai` plus one `human`) and instrumentation that records dispatch start/end timestamps per entry,
**When** `specstudio:specify` runs through the gate,
**Then** at no point during the run are two reviewer dispatches concurrently in flight, and the recorded dispatch start order matches the list order exactly.

### AC: and-composition-blocks-on-any-issues-found (verifies REQ:and-composition, REQ:verdict-contract)

**Given** a `gates.specify.reviewers` list with two `ai` entries followed by one `human` entry, where the first `ai` entry returns `Approved` and the second `ai` entry returns `Issues Found` with one `Blocker` finding,
**When** `specstudio:specify` runs through the gate,
**Then** the gate MUST NOT release, the skill MUST surface the `Blocker` finding to the user, the third entry (the human) MUST NOT be dispatched in the same pass after the failure (the consumer halts after the first `Issues Found` in the current pass), and the skill MUST NOT emit `feature.approved`.

### AC: rerun-policy-applies-on-structural-fix (verifies REQ:rerun-policy)

**Given** a gate run in which reviewer A (`type: ai`) returned `Approved`, reviewer B (`type: ai`) returned `Issues Found`, and the user has applied a fix that modifies the Feature's `## Behavior` section,
**When** `specstudio:specify` re-runs the gate after the fix,
**Then** both reviewer A (because the fix touched a structural section) and reviewer B (because it previously returned `Issues Found`) MUST be re-dispatched in list order before the gate verdict is re-evaluated.

### AC: specify-loads-gate-not-builtin (verifies REQ:specify-loads-gate)

**Given** a `gates.specify.reviewers` list whose first entry is `type: ai` with `prompt: skills/specify/references/reviewer-prompt.md` (the prior baseline path) and whose second entry is `type: human`,
**When** `specstudio:specify` runs and successfully passes the gate,
**Then** the skill MUST have dispatched exactly the two listed entries in list order, MUST NOT have additionally dispatched any reviewer hardcoded inside the skill's own logic, and the resulting verdict-aggregation MUST be composed from the registry entries alone.

### AC: missing-gates-block-refuses-with-error (verifies REQ:missing-gates-block-refuses)

**Given** a `specscore.yaml` in any of the following states — (a) no top-level `gates:` key, (b) `gates:` present but no `gates.specify` sub-key, (c) `gates.specify.reviewers: []`,
**When** `specstudio:specify` is invoked,
**Then** the skill MUST refuse to run, MUST print an error pointing at this Feature and recommending a minimal configuration (at minimum one `type: human` entry), MUST NOT dispatch any reviewer, MUST NOT write or modify any artifact, and MUST exit non-zero.

### AC: third-party-integration-revised (verifies REQ:third-party-integration-revision)

**Given** this Feature is approved and its implementation has run,
**When** a downstream consumer reads `spec/features/third-party-integration/README.md`,
**Then** the file MUST NOT contain any of the six removed REQ slugs (`reviewer-registration-mechanism`, `reviewer-registry-entry-shape`, `reviewer-prompt-location`, `reviewer-contract`, `reviewer-composition`, `reviewer-no-canonical-writes`), MUST NOT contain the `reviewer-registration-and-composition` AC, MUST contain at least one link to this Feature in the `## Interaction with Other Features` table, and MUST pass `specscore spec lint`.

### AC: specify-feature-revised (verifies REQ:specify-feature-revision)

**Given** this Feature is approved and its implementation has run,
**When** a downstream consumer reads `spec/features/skills/specify/README.md`,
**Then** the file MUST NOT contain any of the five removed REQ slugs (`reviewer-subagent-required`, `reviewer-baseline-blockers`, `reviewer-extension-hook`, `reviewer-composition`, `user-approval-required`), MUST contain a single `### Reviewer gate` topic that delegates to this Feature via a link, MUST contain a replacement AC that asserts `gates.specify` consumption, and MUST pass `specscore spec lint`.

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

## Rehearse Integration

Every AC above is testable via filesystem fixtures (mock `specscore.yaml` configurations, mock reviewer prompts, mock subagent verdicts) and either consumer-skill instrumentation (dispatch order, refusal exit codes, verdict-aggregation outcome) or grep-style assertions on the revised/archived/linked files. The non-testable cases (the `ai` reviewer subagent's judgment quality, the human's actual real-world approval-phrase usage) are validated at the assumption-validation layer of the source Idea, not as Rehearse scenarios.

Rehearse stubs for each AC are scaffolded at `_tests/<ac-slug>.md` with `**Status:** pending`; authoring scenario steps follows the implementation plan.

## Not Doing

Inherited from the source Idea and pinned here:

- **Reviewer types beyond `ai` and `human`** — `lint`, `security`, `ux`, `peer-review-bot`, `third-party-ci`, etc. are deferred. The MVP type set rejects unknown values; future additive revisions of this Feature add types one at a time as real consumers ship.
- **`min_approvers > 1` for `type: human`** — pinned to 1 in MVP. Multi-approver workflows are deferred until a real need surfaces.
- **Wiring `plan`, `implement`, `verify`, `recap` into `gates`** — each is a separate follow-on Feature with its own status-transition and event-emission specifics.
- **Auto-skip rule** — a `type: human` reviewer auto-passed when preceding `ai` reviewers all return `Approved` is explicitly out of scope. Deferred until at least one gate runs in real dogfood and we have signal on false-positive risk.
- **Parallel reviewer dispatch** — MVP is serial, mirroring `verify`'s MVP discipline. Parallelism lands in a follow-on Idea once typical reviewer counts and token-burst behavior are observed.
- **Skill-discovery for reviewer entries** — the `future-review-skill-could-discover-available-claude-code` seed is separable; the entry shape can grow a `skill:` field additively later.
- **Reviewer prompt-pack publishing / cross-repo prompt distribution** — prompts stay repo-local for MVP.
- **Plugin-namespace prefixes in gate keys** — `gates.specstudio:specify` is not allowed in MVP; only bare skill names (`gates.specify`). Revisit when multiple plugins ship skills with colliding bare names.
- **`gates:` configuration validation as a `specscore` CLI lint rule** — schema lint of `gates:` entries is deferred. Consumers (MVP: `specstudio:specify`) own load-time validation. A future `specscore config validate` (or a lint rule) MAY subsume.
- **Promotion of the schema into the `specscore` repo** — the `gates:` schema lives in this Feature in `specstudio-skills` for MVP. Promoting to the canonical SpecScore Repo Config Feature happens once a second consumer ships.

## Open Questions

- Canonical wording of the migration error message emitted when a consumer encounters a legacy untyped `reviewers:` entry (per `no-untyped-entry`) — deferred to implementation; the REQ pins the requirement, not the copy.
- Whether a `description:` field on `type: human` entries adds enough value at MVP to mandate. Currently the schema permits but does not require it; implementation may add usage examples without revising this Feature.
- **Grade as verdict currency (planned dependency, own design pass).** The [`score-command`](../score-command/README.md) Feature and the unified `/score` decision ([`spec/research/review-vs-score-command-merge.md`](../../research/review-vs-score-command-merge.md)) require this layer to add a **grade** as the reviewer/gate output: a findings → A–F **aggregation function** and a **configurable Approve threshold** (default `B`) in `specscore.yaml`, where `Approved` is defined as `grade ≥ threshold`. Both the producer-exit gates and manual `/score` would then emit the same grade, making verdict parity structural. The default reviewer shape is **one multi-role-aware reviewer per stage** (BA / developer / QA lenses, per-lens sub-scores → grade), with a multi-agent panel opt-in via the existing `gates.<stage>.reviewers` list. This is a real change to the verdict model here (currently binary `Approved | Issues Found` under AND-composition) and needs its own brainstorm + spec revision — it is recorded here, not yet specified, so the current contract above is unchanged.

---
*This document follows the https://specscore.md/feature-specification*
