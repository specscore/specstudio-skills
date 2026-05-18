# Feature: Sidekick Capture

> [View in SpecStudio](https://specstudio.synchestra.io/project/features?id=specstudio-skills@synchestra-io@github.com&path=spec%2Ffeatures%2Fsidekick-capture) — graph, discussions, approvals

**Status:** Draft
**Source Ideas:** sidekick-ideas

## Summary

Phase 0 of the [`sidekick-ideas`](../../ideas/sidekick-ideas.md) Idea: a new `specstudio:sidekick` skill, a shared capture directive at `skills/shared/sidekick-capture.md`, the seed artifact format at `spec/ideas/seeds/<slug>.md`, and the `sidekick-idea.captured` Synchestra event. The skill is deliberately dumb — it validates a one-liner, writes a seed file, emits an event, and exits. No deliberation, no dedupe across sessions, no auto-promotion happens at this layer; those behaviors belong to the Phase 1 consilium and Phase 2 promotion Features that subscribe to the event. Seeds pile up usefully in `spec/ideas/seeds/` even before the consilium is built — the system is a notebook before it is a court.

The Feature is the smallest independently-shippable slice of the Idea. It produces concrete value on its own (a sideline-idea notebook captured without derailing host work) and produces the artifacts every downstream Phase will consume.

## Problem

While running focused work in `specstudio:ideate`, `specstudio:specify`, or `agent-skills:build`, host agents regularly notice tangential improvement ideas — refactors, missing tests, adjacent features, UX wins — that are out of scope for the current task. Today these get dropped: either the agent derails to chase them, the user is interrupted to triage them, or they are forgotten. None of those outcomes is good.

`sidekick-capture` solves the *write-and-continue* part of the problem and only that part. It gives host agents a one-line discipline ("capture and resume"), defines the durable artifact for the captured seed so it survives across sessions and is reviewable later, and emits an event that downstream consumers (the not-yet-built Phase 1 consilium) can subscribe to. Phase 0 is a notebook; it is not a triage system.

### Departures from the source Idea

The [`sidekick-ideas`](../../ideas/sidekick-ideas.md) Idea names three host skills for directive integration: `specstudio:ideate`, `specstudio:specify`, and `agent-skills:build`. This Feature mandates the directive in our own first-party skills (`specstudio:ideate`, `specstudio:specify`) and provides a documented *adoption contract* for third-party host skills (`agent-skills:build`, `superpowers:brainstorming`, and any other host outside our control) rather than mandating changes to files we do not own. Rationale: this Feature cannot edit files in plugins we do not control; the adoption contract gives third-party authors a stable target to opt in against, and the Feature stays revisable without touching foreign repos. If `agent-skills:build` ever adopts the contract, no further change to this Feature is required.

## Behavior

The Feature ships five components: the `specstudio:sidekick` skill, the shared capture directive, the seed artifact format, the seed lint rule, and the `sidekick-idea.captured` event contract. REQs are grouped by component.

### The `specstudio:sidekick` skill

#### REQ: invocation-triggers

The skill MUST respond to the triggers `specstudio:sidekick`, `/sidekick`, "capture a sidekick idea", "side-kick this", and "park this idea". It MAY respond to additional natural-language phrasings of the same intent. The invocation MUST carry a one-liner as its argument; trigger phrases without a one-liner MUST be treated as invalid input per REQ `input-validation`.

#### REQ: skill-single-mode

The skill MUST support exactly one operational mode: *capture-and-exit*. It MUST NOT trigger panel review, deliberation, dedupe across sessions, content-hash comparison, or auto-promotion — those behaviors belong to later Features that subscribe to the `sidekick-idea.captured` event. The skill MUST NOT introduce additional modes or flags (e.g., `--review`, `--promote`, `--dry-run`) without an additive Feature revision.

#### REQ: input-validation

The skill MUST accept a non-empty one-liner of at most 280 characters after trimming leading and trailing whitespace. It MUST reject empty, whitespace-only, or over-length invocations with a clear error message and a remediation hint ("provide a one-line description, max 280 chars"). It MUST NOT silently truncate over-length input. Unknown flags (e.g., `--review`) MUST be rejected with "unknown flag" rather than silently folded into the one-liner.

#### REQ: writes-seed-artifact

On valid input, the skill MUST write a new seed file at `spec/ideas/seeds/<slug>.md` containing (a) the YAML frontmatter defined in REQ `seed-frontmatter-schema` and (b) a body containing at least the one-liner verbatim. It MUST create `spec/ideas/seeds/` lazily if absent. On slug collision (per REQ `seed-slug-derivation`), the skill MUST disambiguate by appending `-2`, `-3`, … to the slug; it MUST NOT overwrite an existing seed file under any circumstance, even on collision. The skill MUST return the relative seed path to the caller.

#### REQ: emits-captured-event

On successful write, the skill MUST emit a `sidekick-idea.captured` event with the payload defined in REQ `event-payload-schema`. On write failure (filesystem error, validation failure, collision-disambiguation exhaustion), the skill MUST NOT emit the event. Event emission MUST follow the existing Synchestra event-bus convention; the exact transport (in-process call, file-watcher, direct HTTP) is an implementation choice and not constrained here.

### The shared capture directive

#### REQ: directive-location

The shared capture directive MUST live at `skills/shared/sidekick-capture.md`. The file MUST be a self-contained reference document that host skills link to from their own SKILL.md without copying the body. It MUST contain four sections, at minimum: (a) the enumerated heuristic-capture cues per REQ `heuristic-capture-cues`, (b) the write-and-continue discipline per REQ `write-and-continue-discipline`, (c) the invocation pattern for `specstudio:sidekick` from within a host skill, and (d) the same-session dedupe rule placed on the host (per REQ `write-and-continue-discipline`).

#### REQ: heuristic-capture-cues

The directive MUST enumerate at least the following cues as signals of a sideline-idea moment: `"would be nice if…"`, `"another approach is…"`, `"while we're here, we could…"`, `"we should also…"`, `"as a side-effect, …"`, `"tangentially, …"`, `"out of scope but…"`, `"this reminds me — …"`. The list is non-exhaustive guidance; the host skill makes the final call. The directive MUST also explicitly describe what is NOT a sideline idea (the host's own current task, in-scope refinements of the active Feature, clarifying questions to the user, and tool-call decisions for the current step).

#### REQ: write-and-continue-discipline

When a host skill captures a sideline idea (heuristic or explicit), it MUST: (1) invoke `specstudio:sidekick` with the one-liner, (2) acknowledge the capture in a single short line that references the seed path (e.g., `"Captured: <slug> at spec/ideas/seeds/<slug>.md"`), (3) return to the primary task immediately without further discussion of the sideline idea. The host MUST NOT pause to deliberate the merits of the sideline idea, ask the user about it, or branch into a discussion. Within the same conversation, the host MUST NOT re-invoke `/sidekick` for a one-liner it has already captured in that conversation; cross-session and cross-agent dedupe is explicitly *not* a Phase 0 concern and belongs to the Phase 1 consilium via content-hash on the event payload.

#### REQ: host-skill-references

The SKILL.md files for `specstudio:ideate` and `specstudio:specify` MUST contain a markdown link to `skills/shared/sidekick-capture.md` from inside the skill's checklist section. The reference MUST be a link to the directive, not a copy of its body. When this Feature ships, the existing `specstudio:ideate` and `specstudio:specify` SKILL.md files MUST be revised in the same change to add this link. When `specstudio:plan` eventually ships, its SKILL.md MUST include the same reference at authoring time.

### The seed artifact format

#### REQ: seed-path-convention

Every seed file MUST live at `spec/ideas/seeds/<slug>.md`, relative to project root. The `spec/ideas/seeds/` directory MUST be created lazily on first capture. Seeds MUST NOT be nested under further subdirectories of `seeds/`. Seeds MUST NOT live under `spec/ideas/` directly (those are full SpecScore Ideas, not seeds) or under any other path.

#### REQ: seed-frontmatter-schema

Each seed file MUST begin with a YAML frontmatter block containing exactly the following keys, no more, no less:

```yaml
---
type: sidekick-seed                   # literal string; never any other value
slug: <kebab-case-string>             # matches filename without .md
captured_at: <ISO-8601 with timezone> # UTC preferred (e.g., 2026-05-18T14:32:00Z)
captured_by: <string>                 # invoking skill id (e.g., "specstudio:specify") or "user"
captured_during: <string or null>     # active spec path (e.g., "spec/features/skills/init") or null
trigger: heuristic                    # one of: heuristic | explicit
status: queued                        # literal at capture time; populated downstream by Phase 1
synchestra_task: null                 # literal null at capture time; populated downstream by Phase 1
---
```

The lint rule (per REQ `seed-lint-rule`) MUST reject unknown frontmatter keys and missing required keys.

#### REQ: seed-slug-derivation

The slug MUST be derived deterministically from the one-liner by: (a) lowercasing, (b) replacing any character outside `[a-z0-9]` with a single hyphen, (c) collapsing consecutive hyphens into one, (d) trimming leading and trailing hyphens, (e) truncating to at most 60 characters at the nearest preceding hyphen (word boundary). On slug collision with an existing file in `spec/ideas/seeds/`, the skill MUST append `-2` and retry; on further collision `-3`; and so on. The final slug MUST match the regex `^[a-z0-9]+(-[a-z0-9]+)*$` and MUST NOT exceed 64 characters (60 + room for `-NN` disambiguator).

#### REQ: seed-lint-rule

`specscore spec lint` MUST recognize files matching `spec/ideas/seeds/*.md` as the `sidekick-seed` document type and validate them against REQ `seed-frontmatter-schema`. The rule MUST: (a) reject unknown frontmatter keys, (b) reject missing required keys, (c) reject `type` values other than `sidekick-seed`, (d) reject `trigger` values outside the enumerated set, (e) require the body to be non-empty (at least the one-liner). The rule's CLI implementation may land cross-repo in `specscore`; this Feature specifies the rule contract and behavior, not its source location.

### The event contract

#### REQ: event-payload-schema

The `sidekick-idea.captured` event payload MUST contain exactly these fields:

```yaml
event: sidekick-idea.captured        # literal
seed_path: <relative path>           # e.g., spec/ideas/seeds/persist-debug-logs.md
slug: <string>                       # matches seed file slug
captured_at: <ISO-8601 timestamp>    # mirrors seed frontmatter
captured_by: <string>                # mirrors seed frontmatter
captured_during: <string or null>    # mirrors seed frontmatter
trigger: <heuristic|explicit>        # mirrors seed frontmatter
content_hash: <SHA-256 lowercase hex># SHA-256 of the trimmed lowercase one-liner
```

Subscribers (e.g., the Phase 1 consilium) MAY use `content_hash` to dedupe panel reviews across sessions. The capture skill MUST NOT consult `content_hash` for any purpose — it is producer-only at this layer.

### Third-party adoption

#### REQ: third-party-adoption-contract

Host skills outside our control (e.g., `agent-skills:build`, `superpowers:brainstorming`) MAY adopt sidekick capture by referencing `skills/shared/sidekick-capture.md` from their own documentation. Adoption is informational and requires only that the third-party host (a) invokes `specstudio:sidekick` with a valid one-liner and (b) follows the write-and-continue discipline. This Feature does not mandate any change to third-party files; no coordination with the third-party plugin is required. When a third-party host invokes the skill, `captured_by` MUST reflect the calling skill's identifier as-supplied (the skill does not infer or rewrite this field).

## Architecture and Components

The Feature ships five components with explicit boundaries.

1. **`specstudio:sidekick` skill** — lives at `skills/sidekick/SKILL.md` plus any supporting reference files. Stateless. Single responsibility: validate input → derive slug → write seed (with disambiguation) → emit event → return path. No persistence beyond the seed file itself; no in-memory state across invocations.

2. **Shared capture directive** — lives at `skills/shared/sidekick-capture.md`. A reference document; no executable behavior. Linked from first-party host SKILL.md files (REQ `host-skill-references`) and referenced by the third-party adoption contract.

3. **Seed artifact format** — files under `spec/ideas/seeds/`. Plain markdown with YAML frontmatter conforming to REQ `seed-frontmatter-schema`. Owned by the project (committed alongside other spec artifacts); no separate datastore.

4. **Seed lint rule** — a new rule registered with `specscore spec lint` that targets `spec/ideas/seeds/*.md`. Source location for the rule's CLI implementation is the `specscore` CLI repository; this Feature specifies the contract only.

5. **`sidekick-idea.captured` event** — emitted on every successful capture via the Synchestra event-bus convention (`skills/shared/synchestra-events.md`). Payload defined in REQ `event-payload-schema`. Producer: this Feature; consumers: future Phase 1 consilium Feature.

The five components are loosely coupled. The skill produces the seed and the event; the directive instructs hosts how to invoke; the format and lint rule constrain what counts as a valid seed; the event lets downstream Features react without coupling to the skill's internals.

## Interaction with Other Features

- **`specstudio:ideate`** ([feature](../skills/ideate/)) — adds a checklist link to the shared directive (REQ `host-skill-references`). The ideate primary output (the SpecScore Idea artifact) is unchanged. Heuristic capture during an ideate session writes a seed without affecting the Idea draft.

- **`specstudio:specify`** ([feature](../skills/specify/)) — same shape: adds the checklist link. Specify's primary output (the Feature spec) is unchanged. Heuristic capture during specify writes a seed; the active Feature path becomes `captured_during`.

- **`specstudio:plan`** ([idea](../../ideas/specstudio-plan-skill.md)) — when this skill ships, its SKILL.md MUST include the same shared-directive reference (REQ `host-skill-references`).

- **`specstudio:init`** ([feature](../skills/init/)) — no change required for Phase 0. The `spec/ideas/seeds/` directory is created lazily by the sidekick skill on first capture; init does not need to pre-create it. If a future revision wants `init` to pre-create the directory for discoverability, that is an additive change to the `init` Feature, not this one.

- **`third-party-integration`** ([feature](../third-party-integration/)) — establishes the snippet/integration pattern this Feature's `third-party-adoption-contract` REQ leans on. No change to that Feature required; the adoption-contract REQ is informational only.

- **Phase 1 consilium (future Feature)** — subscribes to `sidekick-idea.captured`, dedupes by `content_hash`, reads seeds from `spec/ideas/seeds/`, writes verdicts back to the seed. This Feature is the producer; Phase 1 is the consumer. The event-payload contract (REQ `event-payload-schema`) is the integration surface.

- **Phase 2 auto-promotion (future Feature)** — does not depend on this Feature directly; depends on the Phase 1 consilium's output.

## Acceptance Criteria

### AC: invocation-with-valid-one-liner-captures

**Given** a Claude Code session in a project where `specstudio:sidekick` is installed and `spec/ideas/seeds/` may or may not exist
**When** the user invokes `/sidekick We should persist debug logs across restarts`
**Then** a file is written at `spec/ideas/seeds/we-should-persist-debug-logs-across-restarts.md` with frontmatter containing exactly the eight keys from REQ `seed-frontmatter-schema`, `type: sidekick-seed`, `trigger: explicit`, `status: queued`, `synchestra_task: null`, and a body containing the verbatim one-liner; a `sidekick-idea.captured` event is emitted; the skill returns the relative seed path.

### AC: empty-or-whitespace-input-rejected

**Given** a Claude Code session
**When** the user invokes `/sidekick` with no argument, or with only whitespace
**Then** the skill exits with a clear error indicating an empty one-liner and the 1–280-character constraint; no seed file is created; no event is emitted.

### AC: over-length-input-rejected

**Given** a Claude Code session
**When** the user invokes `/sidekick` with a one-liner of 281 or more characters (after trimming)
**Then** the skill exits with an error indicating the 280-character limit; the over-length text is not silently truncated; no seed file is created; no event is emitted.

### AC: unknown-flag-rejected

**Given** a Claude Code session
**When** the user invokes `/sidekick --review the one-liner here`
**Then** the skill exits with `"unknown flag --review"` rather than treating `--review` as part of the one-liner; no seed file is created; no event is emitted.

### AC: slug-collision-disambiguates-without-overwriting

**Given** an existing file `spec/ideas/seeds/add-caching-to-search.md`
**When** the skill is invoked with a one-liner whose slug derives to `add-caching-to-search`
**Then** a new file is written at `spec/ideas/seeds/add-caching-to-search-2.md`; the existing file is byte-identical before and after; a second such collision produces `-3`; the event payload's `slug` field reflects the disambiguated slug.

### AC: event-emitted-only-on-successful-write

**Given** a filesystem state where `spec/ideas/seeds/` cannot be created (e.g., read-only parent) or written to (e.g., disk full)
**When** the skill is invoked with a valid one-liner
**Then** the skill reports the write failure with a clear error; no `sidekick-idea.captured` event is emitted; the skill exits non-zero.

### AC: event-payload-conforms-to-schema

**Given** a successful capture
**When** the emitted `sidekick-idea.captured` event payload is inspected
**Then** it contains exactly the eight fields specified in REQ `event-payload-schema`, no more and no less; `content_hash` is the SHA-256 hex digest (lowercase) of the trimmed lowercase one-liner; the four mirrored fields (`captured_at`, `captured_by`, `captured_during`, `trigger`) match the seed frontmatter exactly.

### AC: host-skill-references-directive

**Given** the latest SKILL.md files of `specstudio:ideate` and `specstudio:specify` after this Feature ships
**When** each file is read
**Then** each contains a markdown link to `skills/shared/sidekick-capture.md` located in the skill's checklist section; neither file copies the directive body inline.

### AC: heuristic-capture-does-not-derail-host

**Given** an active `specstudio:specify` session that is mid-way through specifying Feature X
**When** the host agent recognizes a sideline-idea cue and invokes `specstudio:sidekick` with a valid one-liner
**Then** the host (a) writes the seed via the skill, (b) acknowledges in a single short line referencing the seed path, (c) returns to the next checklist step for Feature X in the same agent turn; the host does not branch into discussion of the sideline idea, does not ask the user about it, and does not pause the specify checklist.

### AC: same-session-no-double-capture

**Given** a host skill that has already invoked `/sidekick` with one-liner L in the current conversation and received seed path P
**When** the host encounters a cue that would re-fire `/sidekick` with the same L
**Then** the host does not re-invoke `/sidekick`; it may reference the existing seed P in passing without re-writing.

### AC: lint-rejects-malformed-seed

**Given** a seed file with any of: (a) an unknown frontmatter key, (b) a missing required key, (c) `type` other than `sidekick-seed`, (d) `trigger` outside the enumerated set, (e) an empty body
**When** `specscore spec lint` is run on the project
**Then** lint reports a violation pointing at the offending file and key (or the missing-key absence); exit code is non-zero (per the SpecScore CLI exit-code contract).

### AC: slug-is-url-safe-lowercase

**Given** a one-liner containing mixed case, punctuation, and non-ASCII characters
**When** the slug is derived
**Then** the resulting slug matches the regex `^[a-z0-9]+(-[a-z0-9]+)*$`, is at most 60 characters before any disambiguator is appended, and 64 characters after; no uppercase, whitespace, underscore, or non-ASCII character appears in the slug.

### AC: third-party-skill-can-invoke

**Given** a third-party host skill (e.g., a fictional `agent-skills:build`) that follows the adoption contract and invokes `specstudio:sidekick` with a valid one-liner and a `captured_by` of `"agent-skills:build"`
**When** the skill executes
**Then** the seed is written exactly as it would be for a first-party caller; `captured_by` in the frontmatter and event payload is `"agent-skills:build"` verbatim; no special handling distinguishes first-party from third-party callers in the on-disk artifact.

## Not Doing / Out of Scope

The following are deliberately deferred to later Features or rejected outright:

- **Content-hash dedupe at capture time.** Phase 0 does not scan the seeds directory before writing. Cross-session duplicates produce separate seed files; the Phase 1 consilium dedupes panel runs by `content_hash` on the event. Filesystem clutter is not considered harmful; it is provenance.
- **The consilium worker, researcher, scribe, CLI arbiter, and verdict gate.** Phase 1 Features.
- **Auto-promotion to Feature spec or implementation plan.** Phase 2 Feature.
- **Hook ergonomics (auto-drain on `Stop`, `loop`-based scheduling).** Phase 3 Feature.
- **Modifications to third-party skill files** (`agent-skills:build` SKILL.md, `superpowers:brainstorming` SKILL.md, etc.). Adoption is opt-in via the documented contract; this Feature does not edit foreign repos.
- **Pre-creation of `spec/ideas/seeds/` by `specstudio:init`.** Lazy creation by the sidekick skill is sufficient; revisit only if discoverability complaints surface.
- **Free-form prose in the seed body beyond the one-liner.** The body MUST contain the one-liner; whether longer context is permitted is an Outstanding Question (see below). Phase 0 ships with one-liner-only semantics.
- **Roster configuration for the consilium.** Not applicable at this layer; Phase 1.

## Rehearse Integration

Most ACs are testable via filesystem and event-payload observation; per the rehearse heuristic, those scaffold to Rehearse stubs at `spec/features/sidekick-capture/tests/<req-or-ac-slug>.md` with `status: pending`. Stubs scaffold for:

- `invocation-with-valid-one-liner-captures` — write + event observable
- `empty-or-whitespace-input-rejected` — exit code + absence-of-write observable
- `over-length-input-rejected` — exit code + absence-of-write observable
- `unknown-flag-rejected` — exit code + error string
- `slug-collision-disambiguates-without-overwriting` — fixture seed dir, observe second-write path
- `event-emitted-only-on-successful-write` — induce write failure (read-only fs), observe absence of event
- `event-payload-conforms-to-schema` — schema check against emitted payload
- `host-skill-references-directive` — file-content check on host SKILL.md files
- `same-session-no-double-capture` — transcript-pattern check; observable via host-skill agent behavior
- `lint-rejects-malformed-seed` — fixture seeds + `specscore spec lint` invocation
- `slug-is-url-safe-lowercase` — pure-function check against the slug deriver
- `third-party-skill-can-invoke` — fixture invocation with synthetic `captured_by`

Skipped (UX/discipline-shaped, not directly testable):

- `heuristic-capture-does-not-derail-host` — relies on agent behavior across a multi-turn session; manual transcript review covers it. A future Rehearse pattern for transcript-shape assertions could pick this up.

Rehearse stubs are scaffolded with `**Status:** pending` per the rehearse-heuristic; authoring the actual scenario steps follows the implementation plan.

## Outstanding Questions

- **One-liner length cap (280 chars).** Borrowed from the social-post conventions of similar capture tools; not anchored to a concrete constraint. If real captures routinely brush the cap, raise it; if real captures average ≤ 80 chars, leave it. Validate after a week of use.
- **Seed body beyond the one-liner.** Phase 0 requires the body to contain at least the one-liner; it does not constrain longer prose. Open: should the body be strictly the one-liner (lint enforces), or should hosts be allowed to add a short "why it surfaced" paragraph at capture time? Cost: looser body widens lint surface and may invite hosts to derail into context-writing. Default: allow longer prose but do not require it; revisit if hosts start writing essays.
- **`captured_by` format.** Currently a free-form string (e.g., `"specstudio:specify"`, `"user"`). Open: should this be a constrained enum drawn from a registry of known skill IDs, or remain free-form to accommodate third-party adopters? Default: free-form; revisit if downstream consumers (Phase 1) need a stable enum for routing.
- **`captured_during` semantics.** Currently a string referencing the active spec path (e.g., `"spec/features/skills/init"`). Open: should this instead be a Synchestra task ID, a Feature slug, or a free-form context label? Default: spec path; revisit when Phase 1 needs to correlate seeds to active tasks.
- **Seeds-directory pre-creation by `init`.** Deferred per Not Doing. If a future adopter is surprised by the directory appearing only on first capture, revisit and add to `specstudio:init`'s scaffolding.
- **Event transport mechanism.** REQ `emits-captured-event` requires emission via the Synchestra event-bus convention but does not constrain how. Open: does Synchestra currently provide an in-process emission helper, or do skills emit by writing to a known path that a watcher consumes? Resolve when implementing the skill — likely by reading the existing convention from `skills/shared/synchestra-events.md` (not yet authored; tracked by the source Idea).

---
*This document follows the https://specscore.md/feature-specification*
