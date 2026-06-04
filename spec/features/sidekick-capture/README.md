# Feature: Sidekick Capture

> [SpecScore.**Studio**](https://specscore.studio): | [Explore](https://specscore.studio/app/github.com/specscore/specstudio-skills/spec/features/sidekick-capture?op=explore) | [Edit](https://specscore.studio/app/github.com/specscore/specstudio-skills/spec/features/sidekick-capture?op=edit) | [Ask question](https://specscore.studio/app/github.com/specscore/specstudio-skills/spec/features/sidekick-capture?op=ask) | [Request change](https://specscore.studio/app/github.com/specscore/specstudio-skills/spec/features/sidekick-capture?op=request-change) |

**Status:** Approved
**Source Ideas:** sidekick-ideas

## Summary

Phase 0 of the [`sidekick-ideas`](../../ideas/sidekick-ideas.md) Idea: a new `specstudio:sidekick` skill, a shared capture directive at `skills/shared/sidekick-capture.md`, the seed artifact format at `spec/ideas/seeds/<slug>.md`, two-way back-links from each seed's source artifact (Feature, Idea, or Plan) to the generated seed via a `## Sidekick Seeds Generated` section, and the `sidekick-idea.captured` event. The skill is deliberately narrow — it validates a one-liner, writes a seed file, updates the source artifact's back-link section (when `captured_during` resolves to an existing file), emits an event, and exits. No deliberation, no dedupe across sessions, no auto-promotion happens at this layer; those behaviors belong to the Phase 1 consilium and Phase 2 promotion Features that subscribe to the event. Seeds pile up usefully in `spec/ideas/seeds/` even before the consilium is built — the system is a notebook before it is a court.

The Feature is the smallest independently-shippable slice of the Idea. It produces concrete value on its own (a sideline-idea notebook captured without derailing host work) and produces the artifacts every downstream Phase will consume.

## Contents

| Child | Description |
|---|---|
| [destination-resolution](destination-resolution/README.md) | Pre-write destination resolution: the host AI agent deliberates which repo the seed belongs to in a multi-repo workspace; the pick is surfaced as an inline confirmation the human accepts or overrides. Ships alongside the `specstudio:relocate-idea` recovery skill. |

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

The skill MUST accept a non-empty one-liner of at most 500 characters after trimming leading and trailing whitespace. It MUST reject empty, whitespace-only, or over-length invocations with a clear error message and a remediation hint ("provide a one-line description, max 500 chars"). It MUST NOT silently truncate over-length input. Unknown flags (e.g., `--review`) MUST be rejected with "unknown flag" rather than silently folded into the one-liner.

#### REQ: writes-seed-artifact

On valid input, the skill MUST write a new seed file at `spec/ideas/seeds/<slug>.md` containing (a) the YAML frontmatter defined in REQ `seed-frontmatter-schema` and (b) a body whose first non-blank line is an H1 heading (`# <one-liner>`) containing the verbatim one-liner as the heading text. The skill MAY accept an optional `--body <markdown>` argument (or its programmatic equivalent) that places additional markdown content below the H1; if absent, only the H1 is written. The total body length (everything after the closing `---` of the frontmatter, inclusive of the H1 line) MUST NOT exceed 2000 characters; over-length bodies MUST be rejected at the skill layer with a clear error, the same way over-length one-liners are. The skill MUST create `spec/ideas/seeds/` lazily if absent. On slug collision (per REQ `seed-slug-derivation`), the skill MUST disambiguate by appending `-2`, `-3`, … to the slug; it MUST NOT overwrite an existing seed file under any circumstance, even on collision. The skill MUST return the relative seed path to the caller.

#### REQ: emits-captured-event

On successful write, the skill MUST emit a `sidekick-idea.captured` event with the payload defined in REQ `event-payload-schema`. On write failure (filesystem error, validation failure, collision-disambiguation exhaustion), the skill MUST NOT emit the event. Event emission MUST follow the existing event-bus convention; the exact transport (in-process call, file-watcher, direct HTTP) is an implementation choice and not constrained here.

### The shared capture directive

#### REQ: directive-location

The shared capture directive MUST live at `skills/shared/sidekick-capture.md`. The file MUST be a self-contained reference document that host skills link to from their own SKILL.md without copying the body. It MUST contain four sections, at minimum: (a) the enumerated heuristic-capture cues per REQ `heuristic-capture-cues`, (b) the write-and-continue discipline per REQ `write-and-continue-discipline`, (c) the invocation pattern for `specstudio:sidekick` from within a host skill, and (d) the same-session dedupe rule placed on the host (per REQ `write-and-continue-discipline`).

#### REQ: heuristic-capture-cues

The directive MUST enumerate at least the following cues as signals of a sideline-idea moment: `"would be nice if…"`, `"another approach is…"`, `"while we're here, we could…"`, `"we should also…"`, `"as a side-effect, …"`, `"tangentially, …"`, `"out of scope but…"`, `"this reminds me — …"`. The list is non-exhaustive guidance; the host skill makes the final call. The directive MUST also explicitly describe what is NOT a sideline idea (the host's own current task, in-scope refinements of the active Feature, clarifying questions to the user, and tool-call decisions for the current step).

#### REQ: write-and-continue-discipline

When a host skill captures a sideline idea (heuristic or explicit), it MUST: (1) invoke `specstudio:sidekick` with the one-liner, (2) acknowledge the capture in a single short line that references the seed path (e.g., `"Captured: <slug> at spec/ideas/seeds/<slug>.md"`), (3) return to the primary task immediately without further discussion of the sideline idea. The host MUST NOT pause to deliberate the merits of the sideline idea, ask the user about it, or branch into a discussion. The host SHOULD invoke `/sidekick` with just the one-liner; the optional body argument is reserved for cases where a single line genuinely cannot capture the idea (e.g., when a specific code snippet is the seed's point). Filling in a long body for routine captures defeats the write-and-continue discipline. Within the same conversation, the host MUST NOT re-invoke `/sidekick` for a one-liner it has already captured in that conversation; cross-session and cross-agent dedupe is explicitly *not* a Phase 0 concern and belongs to the Phase 1 consilium via content-hash on the event payload.

#### REQ: host-skill-references

The SKILL.md files for `specstudio:ideate` and `specstudio:specify` MUST contain a markdown link to `skills/shared/sidekick-capture.md` from inside the skill's checklist section. The reference MUST be a link to the directive, not a copy of its body. When this Feature ships, the existing `specstudio:ideate` and `specstudio:specify` SKILL.md files MUST be revised in the same change to add this link. When `specstudio:plan` eventually ships, its SKILL.md MUST include the same reference at authoring time.

### The seed artifact format

#### REQ: seed-path-convention

Every seed file MUST live at `spec/ideas/seeds/<slug>.md`, relative to project root. The `spec/ideas/seeds/` directory MUST be created lazily on first capture. Seeds MUST NOT be nested under further subdirectories of `seeds/`. Seeds MUST NOT live under `spec/ideas/` directly (those are full SpecScore Ideas, not seeds) or under any other path.

#### REQ: seed-frontmatter-schema

Each seed file MUST begin with a YAML frontmatter block containing exactly the following required keys at capture time:

```yaml
---
type: sidekick-seed                   # literal string; never any other value
slug: <kebab-case-string>             # matches filename without .md
captured_at: <ISO-8601 with timezone> # UTC preferred (e.g., 2026-05-18T14:32:00Z)
captured_by: <string>                 # convention: "<plugin>:<skill>" for skills
                                      # (e.g., "specstudio:specify"); literal "user"
                                      # for direct user invocation. Free-form; the
                                      # skill does not validate the format.
captured_during: <string or null>     # spec path of the active artifact at capture
                                      # time (e.g., "spec/features/skills/init"),
                                      # or null when no active spec context (e.g.,
                                      # a direct /sidekick outside a host session).
trigger: heuristic                    # one of: heuristic | explicit
status: queued                        # literal at capture time; populated downstream
                                      # (Phase 1 review; `promoted` on promotion — see below)
synchestra_task: null                 # literal null at capture time; populated downstream by Phase 1
---
```

The lint rule (per REQ `seed-lint-rule`) MUST reject unknown frontmatter keys and missing required keys.

**Promoted seeds (downstream extension).** When `specscore idea promote` consumes a seed via its cross-repo path (see the [`seed-to-idea-promotion`](../seed-to-idea-promotion/README.md) Feature), it sets `status: promoted` and adds exactly one **optional** key, `promoted_to: <idea-slug>`, pointing at the created Idea, then relocates the seed to `spec/ideas/archived/<slug>.md`. `promoted_to` is the only permitted optional key: it MUST be absent at capture time and present only on a promoted seed. A promoted seed is still a `sidekick-seed` document (not an Idea) and is validated as a seed regardless of whether it lives under `seeds/` or `archived/`. `promoted` is a distinct terminal `status` from `deprecated` (the consilium's reject outcome) — the two are never conflated.

#### REQ: seed-slug-derivation

The slug MUST be derived deterministically from the one-liner by: (a) lowercasing, (b) replacing any character outside `[a-z0-9]` with a single hyphen, (c) collapsing consecutive hyphens into one, (d) trimming leading and trailing hyphens, (e) truncating to at most 60 characters at the nearest preceding hyphen (word boundary). On slug collision with an existing file in `spec/ideas/seeds/`, the skill MUST append `-2` and retry; on further collision `-3`; and so on. The final slug MUST match the regex `^[a-z0-9]+(-[a-z0-9]+)*$` and MUST NOT exceed 64 characters (60 + room for `-NN` disambiguator).

#### REQ: seed-lint-rule

`specscore spec lint` MUST recognize files matching `spec/ideas/seeds/*.md` — and any `type: sidekick-seed` file under `spec/ideas/archived/*.md` (a promoted seed, per REQ `seed-frontmatter-schema`) — as the `sidekick-seed` document type and validate them against REQ `seed-frontmatter-schema`. The rule MUST: (a) reject unknown frontmatter keys, treating the optional `promoted_to` key as recognized (not unknown), (b) reject missing required keys, (c) reject `type` values other than `sidekick-seed`, (d) reject `trigger` values outside the enumerated set, (e) require the body's first non-blank line to be an H1 heading (`# <text>`), (f) reject body content (after the frontmatter, inclusive of the H1 line) exceeding 2000 characters. A `type: sidekick-seed` file under `archived/` MUST NOT be validated against the Idea schema (it is a seed, not an archived Idea). The rule's CLI implementation may land cross-repo in `specscore`; this Feature specifies the rule contract and behavior, not its source location.

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

### Source-artifact back-links

When a sideline idea is captured during work on a specific artifact (Feature, Idea, or Plan), the source artifact itself MUST gain a back-link to the seed so that a reader reviewing the source artifact sees the list of sideline ideas it spawned without having to grep `spec/ideas/seeds/`. The seed's `captured_during` field carries the source path; this layer turns that one-way link into a two-way link.

#### REQ: writes-back-link-to-source-artifact

On successful seed write, when the resolved `captured_during` value (per REQ `source-artifact-path-resolution`) points at an existing markdown file, the skill MUST append an entry to that file's `## Sidekick Seeds Generated` section. If the section does not exist in the file, the skill MUST create it per REQ `back-link-section-format` before appending. When `captured_during` is `null` or resolves to a non-existent path, the skill MUST NOT attempt back-link maintenance — the seed write and event emission proceed normally, and the absence of a back-link is not an error.

#### REQ: source-artifact-path-resolution

The skill resolves `captured_during` to a markdown file path as follows:

- If the value ends in `.md` and the file exists, use it directly.
- If the value is a directory and `<value>/README.md` exists, use that file.
- Otherwise, treat as non-existent and skip the back-link write (not an error).

The skill MUST NOT follow symlinks that resolve outside the repo root and MUST NOT traverse into hidden directories (any path component starting with `.`).

#### REQ: back-link-section-format

All three SpecScore artifact types (Feature, Idea, Plan) use the same footer convention; the rule below applies uniformly to each, and degrades to end-of-file insertion if a non-conforming source artifact has no footer.

**Placement (when the section does not yet exist):** the `## Sidekick Seeds Generated` section MUST be placed immediately before the SpecScore footer line (the line beginning `*This document follows the https://specscore.md/`) when such a footer is present in the source artifact, or at the end of the file otherwise.

**Placement (when the section already exists):** if a `## Sidekick Seeds Generated` section already exists at any location in the source artifact, the skill MUST append to it in place and MUST NOT relocate it. The placement rule above applies only when *creating* the section.

Each entry MUST follow the format:

    - [<slug>](<relative path from source artifact to seed file>) — captured <ISO-8601 date> by <captured_by>

Entries are append-only (newest at the bottom). The skill MUST NOT reorder existing entries, remove entries, or modify any content in the source artifact outside this section.

The entry format above (a bare relative-path target) applies when the seed and the source artifact live in the **same repo**. When they live in different repos, the repo-qualified variant in REQ `cross-repo-back-link-format` applies instead.

#### REQ: cross-repo-back-link-format

When the resolved destination repo (per the [destination-resolution](destination-resolution/README.md) child) differs from the repo containing the source artifact, the back-link entry the skill writes into the source artifact MUST use a **repo-qualified** target instead of a bare relative path, so downstream tooling (e.g., the `seed-to-idea-promotion` `promote` verb) can identify the seed's origin repo. The entry MUST follow the format:

    - [<slug>](<dest-repo-slug>:spec/ideas/seeds/<slug>.md) — captured <ISO-8601 date> by <captured_by> (cross-repo)

where `<dest-repo-slug>` is the destination repo's `project.repo` value from its `specscore.yaml`. The leading `<dest-repo-slug>:` prefix — a token containing no `/` before the colon — is the cross-repo discriminator: a target beginning with such a prefix denotes a cross-repo seed, whereas a bare relative path denotes a same-repo seed. All other entry rules (append-only ordering, section placement and creation per REQ `back-link-section-format`, no reordering, no out-of-section edits) apply identically.

#### REQ: back-link-best-effort

If the back-link write fails (parse error, write permission error, source artifact mutated unexpectedly mid-operation), the skill MUST report the failure to the caller as a warning but MUST NOT roll back the seed write and MUST proceed with event emission. The seed is the source of truth; the back-link is a discoverability convenience. Failed back-links are recoverable: a future `specscore spec lint --fix` rule that reconciles drift between `spec/ideas/seeds/` and source-artifact back-link sections is tracked as a follow-up (Outstanding Question), but the *immediate* write is in scope here so that users reviewing a freshly-spawned Feature, Idea, or Plan see the back-link list without waiting for a lint pass.

### Third-party adoption

#### REQ: third-party-adoption-contract

Host skills outside our control (e.g., `agent-skills:build`, `superpowers:brainstorming`) MAY adopt sidekick capture by referencing `skills/shared/sidekick-capture.md` from their own documentation. Adoption is informational and requires only that the third-party host (a) invokes `specstudio:sidekick` with a valid one-liner and (b) follows the write-and-continue discipline. This Feature does not mandate any change to third-party files; no coordination with the third-party plugin is required. When a third-party host invokes the skill, `captured_by` MUST reflect the calling skill's identifier as-supplied (the skill does not infer or rewrite this field).

## Architecture and Components

The Feature ships five components with explicit boundaries.

1. **`specstudio:sidekick` skill** — lives at `skills/sidekick/SKILL.md` plus any supporting reference files. Stateless. Single responsibility: validate input → derive slug → write seed (with disambiguation) → emit event → return path. No persistence beyond the seed file itself; no in-memory state across invocations.

2. **Shared capture directive** — lives at `skills/shared/sidekick-capture.md`. A reference document; no executable behavior. Linked from first-party host SKILL.md files (REQ `host-skill-references`) and referenced by the third-party adoption contract.

3. **Seed artifact format** — files under `spec/ideas/seeds/`. Plain markdown with YAML frontmatter conforming to REQ `seed-frontmatter-schema`. Owned by the project (committed alongside other spec artifacts); no separate datastore.

4. **Seed lint rule** — a new rule registered with `specscore spec lint` that targets `spec/ideas/seeds/*.md`. Source location for the rule's CLI implementation is the `specscore` CLI repository; this Feature specifies the contract only.

5. **`sidekick-idea.captured` event** — emitted on every successful capture via the event-bus convention (`skills/shared/events.md`). Payload defined in REQ `event-payload-schema`. Producer: this Feature; consumers: future Phase 1 consilium Feature.

The five components are loosely coupled. The skill produces the seed and the event; the directive instructs hosts how to invoke; the format and lint rule constrain what counts as a valid seed; the event lets downstream Features react without coupling to the skill's internals.

## Interaction with Other Features

- **`specstudio:ideate`** ([feature](../skills/ideate/)) — adds a checklist link to the shared directive (REQ `host-skill-references`). The ideate primary output (the SpecScore Idea artifact) is unchanged. Heuristic capture during an ideate session writes a seed without affecting the Idea draft.

- **`specstudio:specify`** ([feature](../skills/specify/)) — same shape: adds the checklist link. Specify's primary output (the Feature spec) is unchanged. Heuristic capture during specify writes a seed; the active Feature path becomes `captured_during`.

- **`specstudio:plan`** ([feature](../skills/plan/)) — when this skill ships, its SKILL.md MUST include the same shared-directive reference (REQ `host-skill-references`).

- **`specstudio:init`** ([feature](../skills/init/)) — no change required for Phase 0. The `spec/ideas/seeds/` directory is created lazily by the sidekick skill on first capture; init does not need to pre-create it. If a future revision wants `init` to pre-create the directory for discoverability, that is an additive change to the `init` Feature, not this one.

- **`third-party-integration`** ([feature](../third-party-integration/)) — establishes the snippet/integration pattern this Feature's `third-party-adoption-contract` REQ leans on. No change to that Feature required; the adoption-contract REQ is informational only.

- **Phase 1 consilium (future Feature)** — subscribes to `sidekick-idea.captured`, dedupes by `content_hash`, reads seeds from `spec/ideas/seeds/`, writes verdicts back to the seed. This Feature is the producer; Phase 1 is the consumer. The event-payload contract (REQ `event-payload-schema`) is the integration surface.

- **Phase 2 auto-promotion (future Feature)** — does not depend on this Feature directly; depends on the Phase 1 consilium's output.

## Acceptance Criteria

### AC: invocation-with-valid-one-liner-captures

**Given** a Claude Code session in a project where `specstudio:sidekick` is installed and `spec/ideas/seeds/` may or may not exist
**When** the user invokes `/sidekick We should persist debug logs across restarts`
**Then** a file is written at `spec/ideas/seeds/we-should-persist-debug-logs-across-restarts.md` with frontmatter containing exactly the eight keys from REQ `seed-frontmatter-schema`, `type: sidekick-seed`, `trigger: explicit`, `status: queued`, `synchestra_task: null`; the body's first non-blank line is an H1 (`# We should persist debug logs across restarts`) containing the verbatim one-liner; the total body length is ≤ 2000 characters; a `sidekick-idea.captured` event is emitted; the skill returns the relative seed path.

### AC: empty-or-whitespace-input-rejected

**Given** a Claude Code session
**When** the user invokes `/sidekick` with no argument, or with only whitespace
**Then** the skill exits with a clear error indicating an empty one-liner and the 1–500-character constraint; no seed file is created; no event is emitted.

### AC: over-length-input-rejected

**Given** a Claude Code session
**When** the user invokes `/sidekick` with a one-liner of 501 or more characters (after trimming)
**Then** the skill exits with an error indicating the 500-character limit; the over-length text is not silently truncated; no seed file is created; no event is emitted.

### AC: over-length-body-rejected

**Given** a Claude Code session
**When** the user invokes `/sidekick` with a valid one-liner and an optional body that, combined with the H1 line, produces total body content of 2001 or more characters
**Then** the skill exits with an error indicating the 2000-character body limit; the over-length body is not silently truncated; no seed file is created; no event is emitted.

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
**Then** it contains exactly the eight fields specified in REQ `event-payload-schema`, no more and no less; `content_hash` is the SHA-256 hex digest (lowercase) of the trimmed lowercase one-liner; the five mirrored fields (`slug`, `captured_at`, `captured_by`, `captured_during`, `trigger`) match the seed frontmatter exactly.

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

**Given** a seed file with any of: (a) an unknown frontmatter key, (b) a missing required key, (c) `type` other than `sidekick-seed`, (d) `trigger` outside the enumerated set, (e) a body whose first non-blank line is not an H1 heading, (f) a body exceeding 2000 characters
**When** `specscore spec lint` is run on the project
**Then** lint reports a violation pointing at the offending file and the specific rule fired; exit code is non-zero (per the SpecScore CLI exit-code contract).

### AC: slug-is-url-safe-lowercase

**Given** a one-liner containing mixed case, punctuation, and non-ASCII characters
**When** the slug is derived
**Then** the resulting slug matches the regex `^[a-z0-9]+(-[a-z0-9]+)*$`, is at most 60 characters before any disambiguator is appended, and 64 characters after; no uppercase, whitespace, underscore, or non-ASCII character appears in the slug.

### AC: back-link-appended-on-capture

**Given** an existing Feature at `spec/features/foo/README.md` containing a SpecScore footer line, and a `/sidekick` invocation with `captured_during: spec/features/foo`
**When** the skill writes the seed successfully
**Then** the Feature's README is modified to contain a `## Sidekick Seeds Generated` section (created if absent) positioned immediately before the footer line; the section contains a new entry referencing the new seed in the format `- [<slug>](<relative path>) — captured <ISO-8601 date> by <captured_by>`; no other content in the source artifact is modified.

### AC: back-link-section-created-when-absent

**Given** a source artifact whose markdown body has no `## Sidekick Seeds Generated` section
**When** the skill writes a seed pointing at that artifact
**Then** the section is created with exactly one heading line (`## Sidekick Seeds Generated`) and one entry bullet beneath it; the section is positioned immediately before the SpecScore footer line if present, otherwise at end-of-file.

### AC: cross-repo-back-link-repo-qualified (verifies REQ:cross-repo-back-link-format)

**Given** a sideline idea captured during work on a source artifact in repo `A`, with the seed written to sibling repo `B` whose `specscore.yaml` `project.repo` is `B`
**When** the skill writes the back-link into the source artifact in repo `A`
**Then** the new entry's link target is `B:spec/ideas/seeds/<slug>.md` with a trailing `(cross-repo)` marker; and a same-repo capture in the same repo still produces a bare relative-path target (no `<repo-slug>:` prefix).

### AC: back-link-skipped-on-null-captured-during

**Given** a `/sidekick` invocation with `captured_during: null`
**When** the skill writes the seed successfully
**Then** no file other than the seed file is modified; the `sidekick-idea.captured` event still fires; no error or warning about a missing source artifact is reported.

### AC: back-link-skipped-on-nonexistent-path

**Given** a `/sidekick` invocation with `captured_during: spec/features/nonexistent-feature`
**When** the skill is invoked
**Then** the skill writes the seed and emits the event normally; no back-link write is attempted; the skill's exit code is 0 (the absent source artifact is not an error condition).

### AC: back-link-write-failure-does-not-roll-back-seed

**Given** a source artifact at the resolved `captured_during` path that exists but cannot be written (e.g., read-only filesystem or insufficient permission on that single file)
**When** the skill is invoked with a valid one-liner
**Then** the seed file is still written at `spec/ideas/seeds/<slug>.md`; the `sidekick-idea.captured` event is still emitted; the skill reports the back-link write failure to the caller as a warning; the skill's exit code is 0 (success), because the seed and event — the load-bearing artifacts — are correctly written.

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
- **Long-form prose, design docs, or essays in the seed body.** The body cap is 2000 characters (REQ `writes-seed-artifact`), enforced by lint (REQ `seed-lint-rule`). The optional body is for cases where a one-liner genuinely cannot capture the idea (a specific code snippet, a short list of affected places). Routine captures SHOULD use the H1-only form (REQ `write-and-continue-discipline`). Seeds that need more than 2000 characters of context have outgrown "seed" status and belong as a full SpecScore Idea via `specstudio:ideate`.
- **Roster configuration for the consilium.** Not applicable at this layer; Phase 1.

## Rehearse Integration

Most ACs are testable via filesystem and event-payload observation; per the rehearse heuristic, those scaffold to Rehearse stubs at `spec/features/sidekick-capture/_tests/<req-or-ac-slug>.md` with `status: pending`. Stubs scaffold for:

- `invocation-with-valid-one-liner-captures` — write + event observable
- `empty-or-whitespace-input-rejected` — exit code + absence-of-write observable
- `over-length-input-rejected` — exit code + absence-of-write observable
- `over-length-body-rejected` — exit code + absence-of-write observable
- `unknown-flag-rejected` — exit code + error string
- `slug-collision-disambiguates-without-overwriting` — fixture seed dir, observe second-write path
- `event-emitted-only-on-successful-write` — induce write failure (read-only fs), observe absence of event
- `event-payload-conforms-to-schema` — schema check against emitted payload
- `host-skill-references-directive` — file-content check on host SKILL.md files
- `same-session-no-double-capture` — transcript-pattern check; observable via host-skill agent behavior
- `lint-rejects-malformed-seed` — fixture seeds + `specscore spec lint` invocation
- `slug-is-url-safe-lowercase` — pure-function check against the slug deriver
- `third-party-skill-can-invoke` — fixture invocation with synthetic `captured_by`
- `back-link-appended-on-capture` — fixture Feature with footer; assert section + entry appear at expected location after capture
- `back-link-section-created-when-absent` — fixture artifact without the section; assert section is created
- `back-link-skipped-on-null-captured-during` — assert no file other than seed is modified when `captured_during: null`
- `back-link-skipped-on-nonexistent-path` — assert no error, no back-link, normal seed + event when path is invalid
- `back-link-write-failure-does-not-roll-back-seed` — induce write failure on source artifact; assert seed + event present, warning reported, exit 0

Skipped (UX/discipline-shaped, not directly testable):

- `heuristic-capture-does-not-derail-host` — relies on agent behavior across a multi-turn session; manual transcript review covers it. A future Rehearse pattern for transcript-shape assertions could pick this up.

Rehearse stubs are scaffolded with `**Status:** pending` per the rehearse-heuristic; authoring the actual scenario steps follows the implementation plan.

## Open Questions

- **One-liner length cap (500 chars).** Picked to comfortably fit a "what + brief context" capture (e.g., "persist debug logs across restarts so post-mortems don't lose context — three places we wished we had session-level logs that survived a `/clear`") while still discouraging full paragraphs that should be the body, not the one-liner. Not anchored to a concrete external constraint. If real captures routinely brush the cap, raise it; if real captures average ≤ 80 chars, leave it. Validate after a week of use.
- **Seeds-directory pre-creation by `init`.** Deferred per Not Doing. If a future adopter is surprised by the directory appearing only on first capture, revisit and add to `specstudio:init`'s scaffolding.
- **Event transport mechanism.** REQ `emits-captured-event` requires emission via the event-bus convention but does not constrain how. Open: does Synchestra currently provide an in-process emission helper, or do skills emit by writing to a known path that a watcher consumes? Resolve when implementing the skill — likely by reading the existing convention from `skills/shared/events.md`.
- **Back-link drift reconciliation in `specscore spec lint --fix`.** Phase 0 writes back-links at capture time (REQ `writes-back-link-to-source-artifact`), but failed writes (REQ `back-link-best-effort`) and out-of-band edits to source artifacts can produce drift between `spec/ideas/seeds/` and the back-link sections. A future cross-repo lint rule should reconcile this (same pattern as ideas-index sync). Tracked at the [`sidekick-ideas`](../../ideas/sidekick-ideas.md) Idea level; not blocking Phase 0.
- **Concurrent capture against the same source artifact.** Two parallel `/sidekick` invocations whose `captured_during` resolves to the same file may race on the append. REQ `back-link-best-effort` handles this implicitly — one writer wins, the other warns and the seed still lands — but if concurrent capture turns out to be a common shape (e.g., a host fans out multiple captures in one turn), revisit with an explicit locking or retry rule. Defer until real evidence of impact.

---
*This document follows the https://specscore.md/feature-specification*
