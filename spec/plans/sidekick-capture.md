# Sidekick Capture Implementation Plan (Phase 0)

> **For agentic workers:** REQUIRED SUB-SKILL: Use `superpowers:subagent-driven-development` (recommended) or `superpowers:executing-plans` to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ship Phase 0 of the `sidekick-ideas` Idea: capture infrastructure that lets host skills durably record sideline ideas as lint-clean seed files without derailing primary work.

**Architecture:** A new `specstudio:sidekick` skill writes seed files at `spec/ideas/seeds/<slug>.md` per a documented schema (8-key YAML frontmatter + H1 heading + optional ≤2000-char markdown body); a shared directive at `skills/shared/sidekick-capture.md` instructs host skills (`specstudio:ideate`, `specstudio:specify`) on heuristic and explicit capture paths; the `sidekick-idea.captured` event is emitted on successful write via the existing envelope+payload convention in `skills/shared/synchestra-events.md`. The seed lint rule's CLI implementation is cross-repo (`synchestra-io/specscore-cli`) and tracked by a companion plan stub.

**Tech Stack:** Markdown skill authoring (Claude Code skill format); YAML frontmatter; bash for slug derivation and file writes; SHA-256 for content hashing (e.g., `shasum -a 256`); `.synchestra/events.jsonl` for event transport per existing convention; `specscore spec lint` for verification.

---

## Source Spec

This plan implements the Approved Feature at [`spec/features/sidekick-capture/README.md`](../features/sidekick-capture/README.md). All REQ/AC references in this plan resolve to that document.

## In Scope (this plan ships)

- New skill at `skills/sidekick/SKILL.md` plus references
- New shared directive at `skills/shared/sidekick-capture.md`
- Seed template at `skills/sidekick/references/seed-template.md`
- Event-shape addendum to `skills/shared/synchestra-events.md` (extend with `sidekick-idea.captured`)
- Wiring of `specstudio:ideate` and `specstudio:specify` checklists to reference the directive (REQ `host-skill-references`)
- 17 Rehearse stubs at `spec/features/sidekick-capture/_tests/<slug>.md` (12 original + 5 new for back-link ACs)
- Companion plan stub for the cross-repo lint rule

## Out of Scope (deferred to later phases or companion plans)

- The `sidekick-seed` lint rule's CLI implementation in `specscore-cli` (tracked in `spec/plans/sidekick-capture-lint-rule-companion.md` after Task 8)
- Phase 1 consilium (worker, researcher, scribe, CLI arbiter)
- Phase 2 auto-promotion
- Phase 3 hook ergonomics
- Modifications to third-party host skills (`agent-skills:build`, `superpowers:brainstorming`) — adoption is opt-in per REQ `third-party-adoption-contract`

## Cross-Cutting Decisions Resolved Here

These decisions surface in multiple tasks; resolving them once up front avoids drift:

### 1. Event-shape reconciliation with existing convention

REQ `event-payload-schema` lists 8 flat fields. The existing `synchestra-events.md` convention is envelope+payload. The 8 fields map as:

| Spec field (REQ-14) | Envelope or payload | Maps to |
|---|---|---|
| `event` | envelope | `event` (top-level) |
| `seed_path` | envelope | `artifact.path` |
| `slug` | envelope **and** payload | `artifact.id` (envelope) **and** `slug` (payload, for direct consumer use) |
| `captured_at` | envelope | `timestamp` |
| `captured_by` | envelope | `actor.kind` + `actor.id` |
| `captured_during` | payload | `captured_during` |
| `trigger` | payload | `trigger` |
| `content_hash` | payload | `content_hash` |

Plus envelope-required: `version: 1`, `uuid: <generated>`, `artifact.type: idea-seed`, `artifact.revision: <git SHA>`.

The full emitted event is *richer* than REQ-14's 8 fields, but every one of REQ-14's fields appears (at envelope or payload level), so AC `event-payload-conforms-to-schema` is satisfied. This reconciliation is documented in Task 4.

### 2. H1 with markdown-special characters (advisory from reviewer v2)

A one-liner like `Use the # comment syntax in YAML` must produce a valid H1. The skill MUST escape or use a literal-quoting strategy. Decision: **the H1 text is written verbatim, preceded by `# `**. Markdown-special characters in the one-liner are written as-is. Renderers may interpret them; that is acceptable for a "notebook" artifact and matches what users see when they typed the one-liner. Backtick-only one-liners (e.g., starting with `` ` ``) and one-liners that would produce a malformed heading per CommonMark are not gracefully handled in Phase 0; this is noted as a known limitation in the skill.

### 3. Skill-layer + lint-layer redundancy (advisory from reviewer v2)

REQ-4 enforces the 2000-char body cap at skill write time; REQ-13 (f) enforces it again at `specscore spec lint` time. This is **deliberate defense-in-depth**: the skill catches the common case (live invocation); the lint catches the rare case (hand-edited seed). Documented in the skill's behavior section and the directive.

### 4. Seed-slug normalization for SHA-256 content hashing

`content_hash` in REQ-14 is "SHA-256 of the trimmed lowercase one-liner." For determinism across emitters, "normalized one-liner" means: (a) trim leading/trailing whitespace, (b) lowercase via Unicode default casefolding (`tr '[:upper:]' '[:lower:]'` is sufficient for ASCII; for full Unicode, document with `python3 -c "import sys; sys.stdout.write(sys.stdin.read().strip().lower())"`). Tested in Task 3.

---

## File Structure

**Created:**

```
skills/
├── sidekick/
│   ├── SKILL.md                     # ~200 lines, the skill itself
│   └── references/
│       └── seed-template.md         # ~30 lines, example seed file
└── shared/
    └── sidekick-capture.md          # ~80 lines, shared directive

spec/
├── features/sidekick-capture/
│   └── tests/                       # NEW dir
│       ├── invocation-with-valid-one-liner-captures.md
│       ├── empty-or-whitespace-input-rejected.md
│       ├── over-length-input-rejected.md
│       ├── over-length-body-rejected.md
│       ├── unknown-flag-rejected.md
│       ├── slug-collision-disambiguates-without-overwriting.md
│       ├── event-emitted-only-on-successful-write.md
│       ├── event-payload-conforms-to-schema.md
│       ├── host-skill-references-directive.md
│       ├── same-session-no-double-capture.md
│       ├── lint-rejects-malformed-seed.md
│       ├── slug-is-url-safe-lowercase.md
│       └── third-party-skill-can-invoke.md
└── plans/
    └── sidekick-capture-lint-rule-companion.md   # companion-plan stub
```

**Modified:**

```
skills/
├── ideate/SKILL.md                  # add checklist item linking to directive
├── specify/SKILL.md                 # add checklist item linking to directive
└── shared/synchestra-events.md      # add sidekick-idea.captured section
```

18 Rehearse stub files = 17 testable ACs + 1 skipped (`heuristic-capture-does-not-derail-host`). The skipped AC gets a separate `_skipped.md` file documenting the reason. Total = 18 files (17 stubs + 1 skip-reason). Updated post-spec-revision per Task 7.

---

## Task 1: Author the shared capture directive

**Files:**
- Create: `skills/shared/sidekick-capture.md`

**Why first:** Both the sidekick skill (Task 3) and the host skills (Tasks 5, 6) reference this directive. Authoring it first means subsequent tasks have a stable link target.

- [ ] **Step 1: Create the directive file with full content**

Write `skills/shared/sidekick-capture.md`:

```markdown
# Sidekick Capture Directive

**Status:** Contract — shared by `specstudio:ideate`, `specstudio:specify`, and any host skill that opts in.

## Purpose

Capture promising sideline ideas during focused work **without derailing the primary task**. Hosts write a one-line seed, the seed is durable in `spec/ideas/seeds/`, and the host resumes its checklist immediately. Deliberation, dedupe across sessions, and auto-promotion belong to the consilium (Phase 1) and later phases — never to capture.

## When to capture

The host skill captures a sideline idea when it notices any of these cues during its work, *and* the idea is genuinely out of scope for the current task:

- "would be nice if…"
- "another approach is…"
- "while we're here, we could…"
- "we should also…"
- "as a side-effect, …"
- "tangentially, …"
- "out of scope but…"
- "this reminds me — …"

The list is non-exhaustive. The host makes the final judgment.

## What is NOT a sideline idea

- The host's own current task or any in-scope refinement of the active artifact.
- A clarifying question to the user about the current task.
- Tool-call decisions for the current step.
- An idea the user has already articulated and is actively working on.

When in doubt: if engaging with the idea would *change the next step of the current checklist*, it is not a sideline — surface it normally. If engaging would *defer the current step*, capture it.

## Invocation pattern

When a sideline idea surfaces:

1. Invoke `specstudio:sidekick` with the one-liner. Skill signature:

       /sidekick <one-liner>                              # H1-only seed
       /sidekick --body <markdown> <one-liner>            # one-liner + optional body

   The host SHOULD invoke with **only** the one-liner. The `--body` form is reserved for cases where a single line genuinely cannot capture the idea — for example, when the seed's point is a specific code snippet or a short list of affected places. Routine long bodies defeat the write-and-continue discipline.

2. Acknowledge the capture in a single short line in the host's running output:

       Captured: <slug> at spec/ideas/seeds/<slug>.md

3. Return to the primary task immediately. The host MUST NOT:
   - pause to deliberate the merits of the sideline idea
   - ask the user about it
   - branch into a discussion of it
   - re-invoke `/sidekick` for the same one-liner within the same conversation

## Same-session dedupe responsibility

The host tracks what it has captured in the current conversation. If a cue would re-fire `/sidekick` with an already-captured one-liner, the host MUST NOT re-invoke. It MAY reference the existing seed path in passing.

Cross-session and cross-agent dedupe is **not** a Phase 0 concern. The Phase 1 consilium dedupes panel runs by content-hash on the event payload; duplicate seed *files* across sessions are useful provenance, not clutter.

## Defense-in-depth on body cap

The sidekick skill enforces the 2000-char body cap at write time. `specscore spec lint` enforces it again on every linted seed. This is deliberate redundancy — the skill catches the common case, the lint catches hand-edited seeds. Neither path is sufficient on its own.

## Adoption by third-party host skills

Host skills outside this repo (e.g., `agent-skills:build`, `superpowers:brainstorming`) may adopt this directive by linking to it from their own documentation and following the invocation pattern. No coordination is required.
```

- [ ] **Step 2: Lint passes**

Run: `specscore spec lint --severity warning`
Expected: `0 violations found`

- [ ] **Step 3: Commit**

```bash
git add skills/shared/sidekick-capture.md
git commit -m "feat(skills/shared): add sidekick-capture directive

Shared capture directive referenced by specstudio:ideate, specstudio:specify,
and any third-party host skill that opts in. Defines: heuristic cues for
sideline-idea detection, the write-and-continue discipline, the
specstudio:sidekick invocation pattern, same-session dedupe responsibility,
and the defense-in-depth note for the 2000-char body cap.

Per REQs directive-location, heuristic-capture-cues, write-and-continue-
discipline (Feature sidekick-capture).
"
```

---

## Task 2: Author the seed template reference

**Files:**
- Create: `skills/sidekick/references/seed-template.md`

**Why second:** The sidekick skill (Task 3) shows the user this template in its references. Authoring it first means the skill has a stable target to link.

- [ ] **Step 1: Create the directory**

```bash
mkdir -p skills/sidekick/references
```

- [ ] **Step 2: Write the seed-template file**

Write `skills/sidekick/references/seed-template.md`:

````markdown
# Seed Template

This is a reference file showing the exact shape of a seed at `spec/ideas/seeds/<slug>.md`. It is documentation, not a real seed — it does not live under `spec/ideas/seeds/`, so the lint rule does not target it.

## Minimal seed (H1 only)

```markdown
---
type: sidekick-seed
slug: persist-debug-logs-across-restarts
captured_at: 2026-05-18T14:32:00Z
captured_by: specstudio:specify
captured_during: spec/features/skills/init
trigger: heuristic
status: queued
synchestra_task: null
---

# Persist debug logs across Claude Code restarts so post-mortems don't lose context
```

## Seed with optional body

```markdown
---
type: sidekick-seed
slug: caching-strategy-for-search-index
captured_at: 2026-05-18T15:01:00Z
captured_by: user
captured_during: null
trigger: explicit
status: queued
synchestra_task: null
---

# Caching strategy for the search index

## Why it surfaced
Three places in `search/indexer.py` and `search/query.py` re-compute the
same shard map within a single request lifecycle.

## Affected files
- `search/indexer.py:42-78`
- `search/query.py:91-104`

## Out of scope for current task
This is a follow-up to the current refactor; capturing so we don't lose it.
```

## Frontmatter contract

The 8 keys in the YAML block are required. Their values follow REQ `seed-frontmatter-schema` in the [`sidekick-capture` Feature](../../../spec/features/sidekick-capture/README.md):

| Key | Type | Notes |
|---|---|---|
| `type` | string | Literal `sidekick-seed`; never any other value |
| `slug` | string | Kebab-case; matches filename without `.md` |
| `captured_at` | ISO-8601 | UTC preferred |
| `captured_by` | string | `<plugin>:<skill>` for skills, literal `"user"` for direct user invocation |
| `captured_during` | string or null | Spec path of the active artifact, or null when no active spec context |
| `trigger` | enum | `heuristic` or `explicit` |
| `status` | string | Literal `queued` at capture time; Phase 1 consilium modifies the value |
| `synchestra_task` | null | Literal `null` at capture time; Phase 1 consilium populates with task ID |

## Body contract

- First non-blank line MUST be an H1 (`# <one-liner>`) containing the verbatim one-liner.
- Optional markdown content may follow (subheadings, lists, code blocks, etc.).
- Total body length (everything after the closing `---`, inclusive of the H1 line) MUST NOT exceed 2000 characters.
- Bodies that need more context have outgrown "seed" status and should become full SpecScore Ideas via `specstudio:ideate`.
````

- [ ] **Step 3: Lint passes**

Run: `specscore spec lint --severity warning`
Expected: `0 violations found`

- [ ] **Step 4: Commit**

```bash
git add skills/sidekick/references/seed-template.md
git commit -m "feat(skills/sidekick): add seed-template reference

Reference documentation showing the exact shape of a seed at
spec/ideas/seeds/<slug>.md — minimal (H1-only) and with optional body.
Documents the 8-key frontmatter contract and the 2000-char body cap.

Per REQs seed-frontmatter-schema, writes-seed-artifact (Feature
sidekick-capture).
"
```

---

## Task 3: Author the `specstudio:sidekick` skill

**Files:**
- Create: `skills/sidekick/SKILL.md`

**Why this task is big:** The skill body covers input validation, slug derivation, frontmatter assembly, body assembly (with H1), event emission, and error handling. Steps split it into independently verifiable sections.

- [ ] **Step 1: Write the YAML frontmatter and intro**

Create `skills/sidekick/SKILL.md` with this opening:

```markdown
---
name: sidekick
description: |
  Captures a sideline idea as a lint-clean seed at spec/ideas/seeds/<slug>.md
  without derailing the host task. Validates a one-liner, writes the seed with
  required frontmatter and an H1 heading, emits the sidekick-idea.captured
  event, and exits. No deliberation, no dedupe across sessions, no
  auto-promotion happens at this layer — those belong to later Features that
  subscribe to the event.
  Triggers: "specstudio:sidekick", "/sidekick", "capture a sidekick idea",
  "side-kick this", "park this idea".
aliases: [sidekick]
---

# Sidekick

Single-mode skill: **capture-and-exit**. Writes one seed, emits one event, returns. The host invokes this skill from inside its own task to durably record a sideline idea without breaking flow.

For *when* a host should invoke this skill, read [`shared/sidekick-capture.md`](../shared/sidekick-capture.md). This SKILL.md describes the skill's own behavior — what it accepts, what it writes, and what it emits.
```

- [ ] **Step 2: Add the "When to Use" and "Anti-Pattern" sections**

Append:

```markdown
## When to Use

- A host skill (`specstudio:ideate`, `specstudio:specify`, third-party adopter) has detected a sideline idea per `shared/sidekick-capture.md` and is invoking on the host's behalf.
- A user has typed `/sidekick <one-liner>` directly to park an idea.

## Anti-Pattern: Deliberation at Capture

This skill MUST NOT deliberate the merits of the seed, ask the user follow-up questions, scan existing seeds for content duplicates, or trigger any review pipeline. Deliberation is the consilium's job (a separate Feature, not yet built). If a host invocation arrives that requires deliberation, that is a contract violation in the host — capture the seed and return.
```

- [ ] **Step 3: Add input-validation rules**

Append:

```markdown
## Input

The skill accepts:

- **One-liner** (required): a string of 1–500 characters after trimming leading/trailing whitespace.
- **`--body <markdown>`** (optional): additional markdown content that follows the H1 heading. Total body length (H1 line + body) MUST NOT exceed 2000 characters.

## Validation rules (REQs `input-validation`, `writes-seed-artifact`)

Reject with a clear error message and exit non-zero in any of these cases:

| Case | Error message |
|---|---|
| One-liner is empty or whitespace-only | `Empty one-liner. Provide a one-line description, 1–500 chars.` |
| One-liner exceeds 500 chars after trimming | `One-liner too long (<N> chars). Max is 500.` |
| Body, combined with the H1 line, exceeds 2000 chars total | `Body too long (<N> chars). Max body (incl. H1 line) is 2000 chars.` |
| Unknown flag (anything starting with `--` that is not `--body`) | `Unknown flag: <flag>. Supported: --body.` |

On rejection, the skill MUST NOT create any file, MUST NOT emit any event, and MUST exit non-zero.
```

- [ ] **Step 4: Add slug derivation pseudocode**

Append:

````markdown
## Slug derivation (REQ `seed-slug-derivation`)

Given a one-liner `S`:

1. Lowercase `S` using Unicode default casefolding.
2. Replace every character outside `[a-z0-9]` with `-`.
3. Collapse runs of `-` into a single `-`.
4. Trim leading and trailing `-`.
5. If length > 60, truncate to the nearest preceding `-` boundary that produces a slug ≤ 60 chars.

Pseudocode (bash with GNU coreutils; adapt to environment):

```bash
slug=$(printf '%s' "$ONE_LINER" \
  | tr '[:upper:]' '[:lower:]' \
  | sed -E 's/[^a-z0-9]+/-/g' \
  | sed -E 's/^-+|-+$//g')
if [ ${#slug} -gt 60 ]; then
  # truncate at the nearest hyphen ≤ 60
  slug=$(printf '%s' "$slug" | cut -c1-60)
  slug="${slug%-*}"   # drop trailing partial word
fi
```

## Collision disambiguation (REQ `writes-seed-artifact`)

If `spec/ideas/seeds/<slug>.md` already exists:

1. Try `<slug>-2.md`. If that exists, try `-3`, `-4`, …
2. Use the first available suffix.
3. The skill MUST NEVER overwrite an existing file.

The final file name's slug (with any `-N` disambiguator) is used in the frontmatter `slug` field and in the event payload.
````

- [ ] **Step 5: Add frontmatter assembly template**

Append:

````markdown
## Frontmatter assembly (REQ `seed-frontmatter-schema`)

After validation and slug derivation, assemble the YAML frontmatter:

```yaml
---
type: sidekick-seed
slug: <derived-slug-with-any-disambiguator>
captured_at: <ISO-8601 UTC, e.g., 2026-05-18T14:32:00Z>
captured_by: <invoker identifier — see "Determining captured_by" below>
captured_during: <active spec path, or null>
trigger: <heuristic|explicit>
status: queued
synchestra_task: null
---
```

### Determining `captured_by`

- If invoked from inside another skill, `captured_by` is the invoking skill's id in `<plugin>:<skill>` form (e.g., `specstudio:specify`).
- If invoked directly by the user (typed `/sidekick`), `captured_by` is the literal string `user`.
- The skill does not validate the format — the caller supplies the value. The skill writes it verbatim. (Free-form per REQ `seed-frontmatter-schema`.)

### Determining `captured_during`

- Invoking skill or caller supplies the spec path of the active artifact (e.g., `spec/features/skills/init`), or `null` if there is no active spec context (e.g., a bare `/sidekick` from the user outside a host session).
- The skill writes the supplied value verbatim.

### Determining `trigger`

- If the invocation came from a host skill's heuristic-capture path (matching a cue from `sidekick-capture.md`), `trigger: heuristic`.
- If the invocation came from `/sidekick` (user-typed) or an explicit `specstudio:sidekick` invocation in any host context, `trigger: explicit`.
````

- [ ] **Step 6: Add body assembly + write logic**

Append:

```markdown
## Body assembly (REQ `writes-seed-artifact`)

The body has two parts, both optional in form but the first is required in content:

1. **H1 line** (required): exactly `# <one-liner>` where `<one-liner>` is the trimmed user input, written verbatim. Markdown-special characters in the one-liner are written as-is; the file's renderer interprets them. (Known limitation: one-liners starting with backticks or containing constructs that would break CommonMark heading parsing produce malformed headings; this is not a Phase 0 concern.)
2. **Optional body** (only if `--body` was provided): a blank line, then the supplied markdown content.

Total length of the body region (everything after the closing `---`, inclusive of the H1 line and any optional content) MUST be ≤ 2000 characters.

## Writing the seed file (REQ `writes-seed-artifact`)

1. Ensure `spec/ideas/seeds/` exists; create it if not:

   ```bash
   mkdir -p spec/ideas/seeds
   ```

2. Write the file at `spec/ideas/seeds/<final-slug>.md` using atomic write semantics (write to a temporary file in the same directory, then rename). This prevents readers from observing a half-written seed.

3. The skill MUST return the relative seed path on success.
```

- [ ] **Step 7: Add event emission**

Append:

````markdown
## Event emission (REQ `emits-captured-event`, REQ `event-payload-schema`)

On successful write — and only on successful write — emit `sidekick-idea.captured` via the convention in [`shared/synchestra-events.md`](../shared/synchestra-events.md).

The event uses the standard envelope+payload structure. REQ `event-payload-schema` lists 8 conceptual fields; they map to the envelope and payload as follows:

```yaml
event: sidekick-idea.captured
version: 1
uuid: <generated>
timestamp: <captured_at>
actor:
  kind: skill | user
  id: <captured_by>          # e.g., "skill:specstudio:specify" or "user:<username>"
artifact:
  type: idea-seed
  id: <slug>
  path: <seed_path>
  revision: <git SHA at time of emission, or "uncommitted">
payload:
  slug: <slug>
  captured_during: <captured_during or null>
  trigger: <heuristic|explicit>
  content_hash: <SHA-256 lowercase hex of normalized one-liner>
```

### Computing `content_hash`

```bash
content_hash=$(printf '%s' "$ONE_LINER" \
  | python3 -c "import sys; sys.stdout.write(sys.stdin.read().strip().lower())" \
  | shasum -a 256 \
  | awk '{print $1}')
```

(Python is used to ensure full Unicode casefolding; a pure-bash `tr` is acceptable for ASCII-only one-liners.)

### Transport

Per `synchestra-events.md`:

- **Default:** append the event as a single line of JSON to `.synchestra/events.jsonl` at repo root.
- **Hook:** if `command -v synchestra` resolves, prefer `synchestra emit <event.yaml>` (CLI). Otherwise fall back to the file append.

### Failure semantics

- If the file write succeeds but event emission fails, the skill MUST report the emission failure to the caller but MUST NOT roll back the seed file. The seed exists; the event is recoverable by re-emission, but a missing seed is not recoverable. (The lint rule will discover the seed regardless; the consilium can dedupe.)
- If the file write fails, no event is emitted. The skill exits non-zero with the write error.
````

- [ ] **Step 8: Add the output and error contracts**

Append:

````markdown
## Output (success)

On success, the skill prints one short line that the host echoes verbatim:

```
Captured: <slug> at spec/ideas/seeds/<slug>.md
```

It returns the relative seed path as its programmatic return value.

## Output (error)

On any validation or write failure, the skill prints one error line per the validation table above and exits non-zero. The host should propagate this to the user in a single short acknowledgement (e.g., `Sidekick capture failed: <error>`) and continue the primary task — capture failure is never a reason to derail.

## Red Flags

These patterns indicate misuse of this skill; refuse or refactor:

- Capture invocation that includes a long-form analysis or planning content in the one-liner (the one-liner is for the *what*, not the *why*).
- Repeated invocations in the same conversation for the same one-liner (host should dedupe; see `shared/sidekick-capture.md`).
- Invocation with `--body` content longer than ~500 chars for routine captures (defeats the write-and-continue discipline).
- Invocation that produces a slug requiring `-10` or higher disambiguator (signals the host is capturing the same family of ideas repeatedly; the consilium would dedupe these later, but during heavy capture-flooding, suggest the user pause and reflect).

## References

- [`shared/sidekick-capture.md`](../shared/sidekick-capture.md) — when and why hosts invoke this skill.
- [`shared/synchestra-events.md`](../shared/synchestra-events.md) — event envelope and emission transport.
- [`references/seed-template.md`](references/seed-template.md) — example seed files.
- [Feature: `sidekick-capture`](../../spec/features/sidekick-capture/README.md) — the spec this skill implements.
````

- [ ] **Step 9: Lint and visual review**

Run: `specscore spec lint --severity warning`
Expected: `0 violations found`

Open the file in your editor and verify:
- YAML frontmatter is well-formed
- All 8 sections are present (When to Use, Anti-Pattern, Input, Validation rules, Slug derivation, Collision disambiguation, Frontmatter assembly, Body assembly, Writing the seed file, Event emission, Output, Red Flags, References)
- Links to `../shared/sidekick-capture.md`, `../shared/synchestra-events.md`, `references/seed-template.md`, and the Feature spec all resolve relative to the SKILL.md path

- [ ] **Step 10: Commit**

```bash
git add skills/sidekick/SKILL.md
git commit -m "feat(skills/sidekick): add specstudio:sidekick skill

Single-mode capture-and-exit skill. Validates a one-liner (1–500 chars),
optionally accepts a --body argument (total body ≤ 2000 chars), derives
a slug, disambiguates collisions with -2/-3/... suffix, writes the seed
to spec/ideas/seeds/<slug>.md with the 8-key frontmatter and an H1 line,
then emits sidekick-idea.captured per synchestra-events.md envelope.

Implements 5 REQs from Feature sidekick-capture:
- invocation-triggers
- skill-single-mode
- input-validation
- writes-seed-artifact
- emits-captured-event

Event-shape reconciliation: REQ-14's 8 fields map onto the existing
envelope+payload convention as documented in the skill's Event Emission
section.
"
```

---

## Task 3.5: Extend the skill with source-artifact back-link logic

**Added after Task 3 was already committed**, in response to a spec revision (`spec(features): revise sidekick-capture with source-artifact back-links`, commit `4eb4b89`). The spec now requires that when a seed's `captured_during` resolves to an existing source artifact (Feature/Idea/Plan), the skill update that artifact's `## Sidekick Seeds Generated` section so reviewers see the generated seed alongside the source. This task adds the corresponding logic to the already-committed `skills/sidekick/SKILL.md` and inserts a new behavior section into it.

**Files:**
- Modify: `skills/sidekick/SKILL.md`

**Why a separate task:** Task 3 is already committed against the previous spec. Rather than amend that commit (which would obscure history), add a delta commit that brings the SKILL.md into compliance with the revised spec. The new behavior section is inserted between "Writing the seed file" and "Event emission" — that ordering matches the runtime sequence (validate → derive slug → write seed → update back-link → emit event).

- [ ] **Step 1: Locate the insertion point**

Open `skills/sidekick/SKILL.md`. Find the existing `## Writing the seed file (REQ writes-seed-artifact)` section and the `## Event emission (REQ emits-captured-event, REQ event-payload-schema)` section that follows it. The new section goes between them.

- [ ] **Step 2: Insert the source-artifact back-link section**

Insert this content between the "Writing the seed file" section and the "Event emission" section:

````
## Source-artifact back-link (REQs `writes-back-link-to-source-artifact`, `source-artifact-path-resolution`, `back-link-section-format`, `back-link-best-effort`)

After the seed file is written but **before** emitting the event, the skill updates the source artifact's back-link section so reviewers see the generated seed alongside the source. The skill performs the back-link write only when the resolved `captured_during` path points at an existing markdown file; otherwise it skips and proceeds to event emission. Back-link write failures do not block the seed or the event.

### Resolving `captured_during` to a markdown file

Apply these rules in order:

1. If `captured_during` is `null`, skip the back-link write.
2. If the value ends in `.md` and that file exists, use it directly.
3. If the value is a directory and `<value>/README.md` exists, use that file.
4. Otherwise, treat as non-existent and skip (not an error).

Reject (skip back-link write, no error) for paths that:
- resolve outside the repo root via symlinks
- traverse into hidden directories (any path component starting with `.`)

### Locating the section

In the source artifact's markdown body:

1. Search for an existing `## Sidekick Seeds Generated` H2 heading anywhere in the file.
2. If found, append the new entry as the last bullet in that section, **in place**. Do NOT relocate the section.
3. If not found, create the section. Placement:
   - If the file contains a SpecScore footer line (begins with `*This document follows the https://specscore.md/`), place the new section immediately before that footer line.
   - Otherwise, place at end-of-file.

### Entry format

Each entry is a single bullet line:

    - [<slug>](<relative path from source artifact to seed file>) — captured <YYYY-MM-DD> by <captured_by>

- `<slug>` matches the seed's frontmatter `slug` (after any `-N` disambiguator).
- The relative path is computed from the source artifact's directory to `spec/ideas/seeds/<slug>.md`. For a source at `spec/features/foo/README.md`, the relative path is `../../ideas/seeds/<slug>.md`.
- The date is the date portion of `captured_at` (YYYY-MM-DD only, no time).
- `<captured_by>` is the verbatim frontmatter value.

Append-only: newest entry at the bottom of the section. The skill MUST NOT reorder existing entries, remove entries, or modify any content in the source artifact outside this section.

### Failure semantics

If the back-link write fails (filesystem error, parse error, concurrent modification, write permission denied on the source artifact), the skill:

1. MUST NOT roll back the seed write.
2. MUST proceed with event emission as if the back-link write had succeeded.
3. MUST report the back-link write failure to the caller as a warning, e.g., `Warning: back-link write to <source-path> failed: <error>; seed and event are recorded.`
4. MUST exit 0 (success). The seed and event are the load-bearing artifacts; the back-link is a discoverability convenience that a future `specscore spec lint --fix` rule will reconcile (deferred per the Feature's Outstanding Questions).
````

- [ ] **Step 3: Lint passes**

Run: `specscore spec lint --severity warning`
Expected: `0 violations found`

- [ ] **Step 4: Manual section-order check**

Confirm the SKILL.md section order is now:
1. (frontmatter)
2. # Sidekick (H1 + intro)
3. ## When to Use
4. ## Anti-Pattern: Deliberation at Capture
5. ## Input
6. ## Validation rules
7. ## Slug derivation
8. ## Collision disambiguation
9. ## Frontmatter assembly
10. ## Body assembly
11. ## Writing the seed file
12. **## Source-artifact back-link** ← new
13. ## Event emission
14. ## Output (success)
15. ## Output (error)
16. ## Red Flags
17. ## References

- [ ] **Step 5: Commit**

```bash
git add skills/sidekick/SKILL.md
git commit -m "$(cat <<'EOF'
feat(skills/sidekick): add source-artifact back-link logic

Extends the already-committed specstudio:sidekick skill with the back-
link writing behavior introduced in the spec revision at commit 4eb4b89.
After writing the seed and before emitting the event, the skill resolves
captured_during to a markdown file (via the .md-direct or directory-
README.md rule) and appends an entry to the source artifact's
## Sidekick Seeds Generated section, creating the section if absent.

Implements 4 new REQs from the revised Feature sidekick-capture:
- writes-back-link-to-source-artifact
- source-artifact-path-resolution
- back-link-section-format
- back-link-best-effort

Failure semantics: best-effort. Back-link write failure does not roll
back the seed, does not block event emission, and exits 0 with a warning.

Co-Authored-By: Claude Opus 4.7 <noreply@anthropic.com>
EOF
)"
```

---

## Task 4: Extend `synchestra-events.md` with `sidekick-idea.captured`

**Files:**
- Modify: `skills/shared/synchestra-events.md`

**Why now:** The skill (Task 3) references this section. Adding it after the skill keeps task scope focused but means the link from Task 3 is broken until Task 4 lands. That is acceptable because the link is documentary, not enforced. (Alternative would be to swap Tasks 3 and 4; either order works.)

- [ ] **Step 1: Locate the insertion point**

Open `skills/shared/synchestra-events.md`. The file has sections per emitting skill (`Events Emitted by specstudio:ideate`, `Events Emitted by specstudio:specify`). Add a new sibling section `Events Emitted by specstudio:sidekick` after the last existing skill section.

- [ ] **Step 2: Append the section**

Add this content:

````markdown
## Events Emitted by `specstudio:sidekick`

### `sidekick-idea.captured`
Fired exactly once per successful seed write at `spec/ideas/seeds/<slug>.md`. The skill writes the seed file before emitting; if emission fails after a successful write, the seed remains on disk and is recoverable by re-emission (see the skill's failure semantics). The event is not fired on validation or write failure.

```yaml
event: sidekick-idea.captured
version: 1
uuid: <generated>
timestamp: <ISO-8601 of capture moment; mirrors seed frontmatter `captured_at`>
actor:
  kind: skill | user
  id: <invoker — "<plugin>:<skill>" form for skills, "user:<username>" for direct user invocation>
artifact:
  type: idea-seed
  id: <slug>                         # matches the seed's frontmatter `slug` and filename
  path: <seed_path>                  # e.g., spec/ideas/seeds/persist-debug-logs.md
  revision: <git SHA at emission, or "uncommitted">
payload:
  slug: <slug>                       # duplicated for direct consumer access
  captured_during: <string or null>  # mirrors seed frontmatter; spec path or null
  trigger: heuristic | explicit       # mirrors seed frontmatter
  content_hash: <SHA-256 lowercase hex of normalized one-liner>
```

**Normalization for `content_hash`:** the one-liner is trimmed (leading/trailing whitespace removed) and lowercased via Unicode default casefolding before hashing. Different emitters compute the same hash for the same idea, which lets the Phase 1 consilium dedupe panel runs across sessions.

**Consumer:** the Phase 1 consilium subscribes here. It dedupes by `content_hash` over a rolling window, then enqueues a `consilium-review` task for the seed. Consumers that want a flat 8-field view (per Feature `sidekick-capture` REQ `event-payload-schema`) project: `event`, `seed_path` (from `artifact.path`), `slug`, `captured_at` (from `timestamp`), `captured_by` (from `actor.id`), `captured_during`, `trigger`, `content_hash`.
````

- [ ] **Step 3: Lint passes**

Run: `specscore spec lint --severity warning`
Expected: `0 violations found`

- [ ] **Step 4: Commit**

```bash
git add skills/shared/synchestra-events.md
git commit -m "feat(skills/shared): add sidekick-idea.captured event

Extend synchestra-events.md with the event emitted by specstudio:sidekick
on successful seed write. Maps the 8 conceptual fields from Feature
sidekick-capture REQ event-payload-schema onto the existing envelope+
payload structure (event, version, uuid, timestamp, actor, artifact,
payload). Documents content_hash normalization (trim + Unicode casefold).

Per REQ emits-captured-event (Feature sidekick-capture).
"
```

---

## Task 5: Wire `specstudio:ideate` to the directive

**Files:**
- Modify: `skills/ideate/SKILL.md`

**Insertion point:** the `## Checklist` section currently ends at item 10 (`Emit events`) before the next H2 (`## Phase 1`). Add a new item 11 as a "throughout" reminder.

- [ ] **Step 1: Read the current checklist to confirm structure**

```bash
sed -n '45,60p' skills/ideate/SKILL.md
```

Verify item 10 ends `... See [synchestra-events.md](../shared/synchestra-events.md).` and the next section is `## Phase 1 — Understand & Expand (Divergent)`.

- [ ] **Step 2: Append item 11 to the checklist**

Insert this line immediately after item 10 (before the blank line that precedes `## Phase 1`):

```markdown
11. **Throughout** — watch for sidekick ideas per [sidekick-capture.md](../shared/sidekick-capture.md). When an out-of-scope improvement surfaces, invoke `specstudio:sidekick` with a one-liner, acknowledge in one line, and return to the current checklist step immediately. Do not derail to discuss the sideline idea.
```

- [ ] **Step 3: Lint passes**

Run: `specscore spec lint --severity warning`
Expected: `0 violations found`

- [ ] **Step 4: Verify the link resolves**

```bash
ls skills/shared/sidekick-capture.md
```

Expected: file lists (created in Task 1).

- [ ] **Step 5: Commit**

```bash
git add skills/ideate/SKILL.md
git commit -m "feat(skills/ideate): wire sidekick-capture directive

Add checklist item 11 referencing skills/shared/sidekick-capture.md.
Instructs the ideate agent to capture sideline ideas via
specstudio:sidekick without derailing the ideate flow.

Per REQ host-skill-references (Feature sidekick-capture).
"
```

---

## Task 6: Wire `specstudio:specify` to the directive

**Files:**
- Modify: `skills/specify/SKILL.md`

**Insertion point:** the `## Checklist` section ends at item 14 (`Transition to writing-plans`). Add item 15 as the "throughout" reminder.

- [ ] **Step 1: Read the current checklist**

```bash
sed -n '47,68p' skills/specify/SKILL.md
```

Verify item 14 ends `Transition to writing-plans.` and the next section is `## Spec Sections (scale to complexity)`.

- [ ] **Step 2: Append item 15**

Insert immediately after item 14:

```markdown
15. **Throughout** — watch for sidekick ideas per [sidekick-capture.md](../shared/sidekick-capture.md). When an out-of-scope improvement surfaces, invoke `specstudio:sidekick` with a one-liner, acknowledge in one line, and return to the current checklist step immediately. Do not derail to discuss the sideline idea.
```

- [ ] **Step 3: Lint passes**

Run: `specscore spec lint --severity warning`
Expected: `0 violations found`

- [ ] **Step 4: Commit**

```bash
git add skills/specify/SKILL.md
git commit -m "feat(skills/specify): wire sidekick-capture directive

Add checklist item 15 referencing skills/shared/sidekick-capture.md.
Instructs the specify agent to capture sideline ideas via
specstudio:sidekick without derailing the specify flow.

Per REQ host-skill-references (Feature sidekick-capture).
"
```

---

## Task 7: Scaffold the 17 Rehearse stubs + 1 skip-reason

**Files:**
- Create: `spec/features/sidekick-capture/_tests/<ac-slug>.md` × 12
- Create: `spec/features/sidekick-capture/_tests/_skipped.md` × 1

**Why bundled into one task:** the stubs follow a uniform template; writing them one-by-one would be repetitive. Each stub is small (~15 lines). They commit as a unit.

- [ ] **Step 1: Create the tests directory**

```bash
mkdir -p spec/features/sidekick-capture/_tests
```

- [ ] **Step 2: Define the stub template**

Each stub uses this format. Substitute `<ac-slug>` and the Given/When/Then text from the corresponding AC in the Feature spec.

```markdown
---
type: rehearse-stub
status: pending
ac: <ac-slug>
feature: sidekick-capture
---

# Rehearse: <ac-slug>

## Scenario (from AC)

**Given** <precondition copied verbatim from the AC>
**When** <action copied verbatim from the AC>
**Then** <expected outcome copied verbatim from the AC>

## Verification approach

<2–4 sentences describing how to actually run this scenario: what command, what fixture, what assertion against what observable.>
```

- [ ] **Step 3: Write all 17 stubs**

Create one file per AC. For each, copy the Given/When/Then verbatim from `spec/features/sidekick-capture/README.md`'s Acceptance Criteria section, then write a 2–4-sentence verification approach.

**12 stub files** (filename = AC slug + `.md`):

1. `invocation-with-valid-one-liner-captures.md`
   - Verification: run `specstudio:sidekick "We should persist debug logs across restarts"` in a fixture project; assert `spec/ideas/seeds/we-should-persist-debug-logs-across-restarts.md` exists with the expected frontmatter; assert one line was appended to `.synchestra/events.jsonl`.

2. `empty-or-whitespace-input-rejected.md`
   - Verification: invoke `specstudio:sidekick ""` and `specstudio:sidekick "   "`; assert non-zero exit; assert no file was created under `spec/ideas/seeds/`; assert `.synchestra/events.jsonl` line count is unchanged.

3. `over-length-input-rejected.md`
   - Verification: invoke with a fixture one-liner of 501 chars; assert non-zero exit with the 500-char message; assert no file or event change.

4. `over-length-body-rejected.md`
   - Verification: invoke with a valid one-liner and `--body` content that pushes the total body length to 2001 chars; assert non-zero exit with the 2000-char body message; assert no file or event change.

5. `unknown-flag-rejected.md`
   - Verification: invoke `specstudio:sidekick --review "x"`; assert non-zero exit with `Unknown flag: --review`; assert no file or event change.

6. `slug-collision-disambiguates-without-overwriting.md`
   - Verification: pre-seed `spec/ideas/seeds/add-caching-to-search.md`; invoke with a one-liner that derives the same slug; assert new file at `spec/ideas/seeds/add-caching-to-search-2.md`; assert original file's content and mtime unchanged; assert event's `slug` field is `add-caching-to-search-2`.

7. `event-emitted-only-on-successful-write.md`
   - Verification: chmod `spec/ideas/` to read-only; invoke; assert write fails with a clear error; assert `.synchestra/events.jsonl` line count is unchanged.

8. `event-payload-conforms-to-schema.md`
   - Verification: invoke with a known one-liner; parse the most recent line of `.synchestra/events.jsonl` as JSON; assert envelope has `event`, `version`, `uuid`, `timestamp`, `actor`, `artifact`; assert payload has exactly `slug`, `captured_during`, `trigger`, `content_hash`; assert `content_hash` equals the SHA-256 of the trimmed lowercase one-liner.

9. `host-skill-references-directive.md`
   - Verification: grep `skills/ideate/SKILL.md` and `skills/specify/SKILL.md` for the link `../shared/sidekick-capture.md`; assert both contain it inside their `## Checklist` section.

10. `same-session-no-double-capture.md`
    - Verification: drive a host session (`specstudio:specify`) through a transcript that contains two identical sideline cues; assert exactly one capture invocation in the transcript; assert the second occurrence is mentioned without re-invoking. (Multi-turn agent behavior; manual review of transcript is acceptable for Phase 0.)

11. `lint-rejects-malformed-seed.md`
    - Verification: write six fixture seeds, each triggering one of the rejection conditions (a–f from REQ `seed-lint-rule`); run `specscore spec lint`; assert non-zero exit and one violation per fixture; assert the violation message references the specific rule.

12. `slug-is-url-safe-lowercase.md`
    - Verification: invoke with a one-liner containing mixed case, punctuation, and non-ASCII (e.g., `"Refactor: Connection Pool — 高速化 (Phase 2!)"`); assert the resulting slug matches `^[a-z0-9]+(-[a-z0-9]+)*$`; assert length ≤ 60.

13. `third-party-skill-can-invoke.md`
    - Verification: from a fixture third-party skill (mocked), invoke `specstudio:sidekick` with `captured_by="agent-skills:build"`; assert the seed's frontmatter `captured_by` is verbatim `agent-skills:build`; assert the emitted event's `actor.id` reflects the same value.

14. `back-link-appended-on-capture.md`
    - Verification: pre-create a fixture Feature at `spec/features/foo/README.md` containing a SpecScore footer line and no existing back-link section. Invoke the skill with `captured_during=spec/features/foo` and a known one-liner. Assert: (a) the seed file exists, (b) the Feature's README now contains a `## Sidekick Seeds Generated` section positioned immediately before the footer line, (c) the section contains exactly one entry in the format `- [<slug>](../../ideas/seeds/<slug>.md) — captured <YYYY-MM-DD> by <captured_by>`, (d) no other content in the Feature's README has changed (diff outside the new section is empty).

15. `back-link-section-created-when-absent.md`
    - Verification: pre-create two fixtures: (a) a source artifact with a footer and no existing section, (b) a source artifact with no footer and no existing section. Invoke against each. Assert: in case (a), the new section is immediately before the footer; in case (b), the new section is at end-of-file. In both cases, the section heading is exactly `## Sidekick Seeds Generated` and contains exactly one bullet beneath it.

16. `back-link-skipped-on-null-captured-during.md`
    - Verification: invoke with `captured_during=null`. Assert: seed exists, `.synchestra/events.jsonl` line added, NO other file modified, exit code 0, no warning about a missing source artifact.

17. `back-link-skipped-on-nonexistent-path.md`
    - Verification: invoke with `captured_during=spec/features/this-feature-does-not-exist`. Assert: seed exists, event emitted, exit code 0, no back-link write attempted, no error reported.

18. `back-link-write-failure-does-not-roll-back-seed.md`
    - Verification: pre-create a fixture Feature and `chmod 0444` its README.md (or otherwise make it unwritable while the parent dir remains writable). Invoke the skill pointing at it. Assert: seed file exists, event emitted, warning surfaced about back-link failure (substring "back-link write to <source-path> failed"), exit code 0. Reset permissions after the test.

That is 17 testable stubs + 1 skip-reason record (`_skipped.md` for `heuristic-capture-does-not-derail-host`). The skipped AC is documented in Step 4.

- [ ] **Step 4: Write the skip-reason file**

The Feature's Rehearse Integration explicitly skips `heuristic-capture-does-not-derail-host`. Document the skip:

Create `spec/features/sidekick-capture/_tests/_skipped.md`:

```markdown
---
type: rehearse-skip-record
feature: sidekick-capture
---

# Rehearse: Skipped ACs

## AC: `heuristic-capture-does-not-derail-host`

**Skip reason:** This AC relies on multi-turn agent behavior across a `specstudio:specify` session — specifically, that the host writes the seed, acknowledges in one line, and returns to the next checklist step *in the same agent turn*. Rehearse stubs as currently designed assert against deterministic observables (file contents, event JSON, exit codes); they cannot assert against the *shape* of an agent's transcript-level behavior.

**Coverage approach:** manual transcript review during host-skill QA. A future Rehearse pattern for transcript-shape assertions (e.g., "no user-facing question between the capture line and the next checklist step") would pick this up automatically. Tracked as a Rehearse roadmap item, not blocking Phase 0.
```

- [ ] **Step 5: Lint passes**

Run: `specscore spec lint --severity warning`
Expected: `0 violations found`

- [ ] **Step 6: Commit**

```bash
git add spec/features/sidekick-capture/_tests/
git commit -m "test(features/sidekick-capture): scaffold 17 Rehearse stubs + 1 skip

One stub per testable AC, status: pending, scenario copied verbatim from
the Feature's Acceptance Criteria, plus a 2–4-sentence verification
approach. One skip-reason record for heuristic-capture-does-not-derail-host
(transcript-shape AC; manual review).

Per Rehearse Integration section (Feature sidekick-capture).
"
```

---

## Task 8: Companion plan stub for the cross-repo lint rule

**Files:**
- Create: `spec/plans/sidekick-capture-lint-rule-companion.md`

**Why a stub:** REQ `seed-lint-rule` in the Feature specifies the lint rule's *contract*, but the *implementation* lives in `synchestra-io/specscore-cli` (a different repo). A full implementation plan for that work belongs in the specscore-cli repo. This stub in *this* repo flags the dependency, lists the rule's behavior succinctly, and is what a maintainer reads to understand "the lint rule isn't here yet — it goes there."

- [ ] **Step 1: Write the companion stub**

Create `spec/plans/sidekick-capture-lint-rule-companion.md`:

```markdown
# Sidekick Capture Lint Rule — Cross-Repo Companion Plan Stub

**Status:** Stub. This plan exists in *this* repo to record the dependency. The actual implementation work happens in [`synchestra-io/specscore-cli`](https://github.com/synchestra-io/specscore-cli).

**Source contract:** REQ `seed-lint-rule` in [`spec/features/sidekick-capture/README.md`](../features/sidekick-capture/README.md).

## What needs to ship in specscore-cli

A new lint rule registered against the SpecScore CLI that:

1. Targets files matching `spec/ideas/seeds/*.md`.
2. Recognizes them as the `sidekick-seed` document type.
3. Rejects:
   - (a) unknown frontmatter keys
   - (b) missing required keys (8 keys from REQ `seed-frontmatter-schema`)
   - (c) `type` values other than `sidekick-seed`
   - (d) `trigger` values outside `{heuristic, explicit}`
   - (e) bodies whose first non-blank line is not an H1 (`# <text>`)
   - (f) bodies (after frontmatter, inclusive of H1) exceeding 2000 characters

## Why it's not implemented here

This repo (`specstudio-skills`) authors the *contract*. The CLI authors the *enforcement*. Keeping the rule in the CLI repo means every SpecScore project gets the rule by upgrading the CLI, without per-project plumbing.

## How to verify the rule is live

After the rule ships in specscore-cli and a SpecScore project upgrades:

```bash
# Write a fixture seed with an unknown key
cat > spec/ideas/seeds/_test.md <<EOF
---
type: sidekick-seed
slug: test
captured_at: 2026-05-18T00:00:00Z
captured_by: user
captured_during: null
trigger: explicit
status: queued
synchestra_task: null
unknown_key: oops
---

# Test seed
EOF

specscore spec lint
# Expected: violation flagging unknown_key under spec/ideas/seeds/_test.md
rm spec/ideas/seeds/_test.md
```

## Tracking

- Open an issue in `synchestra-io/specscore-cli` titled "Add `sidekick-seed` lint rule" with a link to this stub and to REQ `seed-lint-rule`.
- Until the rule ships, the contract is enforceable only by visual review; Phase 0 still functions because the sidekick skill enforces frontmatter and body shape at write time (defense-in-depth per the directive).
```

- [ ] **Step 2: Lint passes**

Run: `specscore spec lint --severity warning`
Expected: `0 violations found`

- [ ] **Step 3: Open the upstream issue** (manual; not automated by this plan)

In `synchestra-io/specscore-cli`, open:

> **Title:** Add `sidekick-seed` lint rule
>
> **Body:** New lint rule contract: see [`specstudio-skills` plan stub](https://github.com/synchestra-io/specstudio-skills/blob/main/spec/plans/sidekick-capture-lint-rule-companion.md) and the source REQ at [`spec/features/sidekick-capture/README.md` REQ `seed-lint-rule`](https://github.com/synchestra-io/specstudio-skills/blob/main/spec/features/sidekick-capture/README.md#req-seed-lint-rule).

Note the issue URL in the commit message in the next step.

- [ ] **Step 4: Commit**

```bash
git add spec/plans/sidekick-capture-lint-rule-companion.md
git commit -m "plan(sidekick-capture): companion stub for cross-repo lint rule

The seed-lint-rule contract specified in Feature sidekick-capture lives
in this repo; the implementation lives in synchestra-io/specscore-cli.
This stub records the dependency, restates the rule's behavior, and
points at the upstream tracking issue.

Per REQ seed-lint-rule (Feature sidekick-capture).
"
```

---

## Task 9: Manual rehearsal and final verification

**Why:** Tasks 1–8 each had local verification, but no end-to-end exercise has fired the full capture path. This task is a single live invocation that exercises the happy path against the actual files.

**Files:** none created or modified. This task is observational.

- [ ] **Step 1: Confirm working tree is clean**

```bash
git status --short
```

Expected: empty output (everything from Tasks 1–8 committed).

- [ ] **Step 2: Verify all expected files exist**

```bash
ls skills/sidekick/SKILL.md \
   skills/sidekick/references/seed-template.md \
   skills/shared/sidekick-capture.md \
   spec/features/sidekick-capture/_tests/_skipped.md \
   spec/plans/sidekick-capture-lint-rule-companion.md
ls spec/features/sidekick-capture/_tests/*.md | wc -l
```

Expected: all five files list; the wc shows `18` (17 stubs + 1 skip record).

- [ ] **Step 3: Verify host-skill wiring**

```bash
grep -l 'sidekick-capture.md' skills/ideate/SKILL.md skills/specify/SKILL.md
```

Expected: both files list.

- [ ] **Step 4: Verify event-shape addendum**

```bash
grep -A 2 '^### `sidekick-idea.captured`' skills/shared/synchestra-events.md
```

Expected: the heading shows with the descriptive line below.

- [ ] **Step 5: Final project-wide lint**

```bash
specscore spec lint --severity warning
```

Expected: `0 violations found`

- [ ] **Step 6: Live invocation rehearsal** (manual)

In a Claude Code session in this repo, invoke `/sidekick --captured-during=spec/features/sidekick-capture We should test the sidekick skill end-to-end` (with `captured_during` pointing at this Feature) and verify:

- A file appears at `spec/ideas/seeds/we-should-test-the-sidekick-skill-end-to-end.md` with the 8-key frontmatter (note `captured_during: spec/features/sidekick-capture`) and an H1 line containing the verbatim one-liner.
- A line is appended to `.synchestra/events.jsonl` (create the file with `touch .synchestra/events.jsonl` first; ensure `.synchestra/` is in `.gitignore` if it isn't already — check `cat .gitignore | grep synchestra`).
- **The source artifact (`spec/features/sidekick-capture/README.md`) now contains a `## Sidekick Seeds Generated` section** positioned immediately before the SpecScore footer line, containing one entry: `- [we-should-test-...](../../ideas/seeds/we-should-test-the-sidekick-skill-end-to-end.md) — captured <YYYY-MM-DD> by user`. This is the back-link in action — verifies REQs `writes-back-link-to-source-artifact`, `source-artifact-path-resolution`, `back-link-section-format`.
- The skill's output is one short line: `Captured: ... at spec/ideas/seeds/...`.

After verifying, revert the test changes (both the seed and the back-link entry):

```bash
rm spec/ideas/seeds/we-should-test-the-sidekick-skill-end-to-end.md
# Revert the Feature README's back-link section
git checkout -- spec/features/sidekick-capture/README.md
git status --short  # confirm seed gone, README restored, no other changes
```

(Alternative: keep the back-link entry as a dogfood example committed; the user decides at rehearsal time.)

- [ ] **Step 7: Update the Feature's status if appropriate**

If the rehearsal passes, transition the Feature's `**Status:**` from `Approved` to `Implementing`:

```bash
sed -i.bak 's/^\*\*Status:\*\* Approved$/**Status:** Implementing/' spec/features/sidekick-capture/README.md && rm spec/features/sidekick-capture/README.md.bak
specscore spec lint --fix
specscore spec lint --severity warning
```

Expected: status flipped; lint clean (autofix may also sync the features index).

- [ ] **Step 8: Final commit**

```bash
git add -A
git commit -m "spec(features): transition sidekick-capture to Implementing

End-to-end rehearsal of the capture path passed (manual invocation
created a seed and emitted an event per the contract). Phase 0 is
behaviorally complete in this repo; the seed-lint-rule's CLI
implementation remains tracked in the companion plan stub.

Per Feature sidekick-capture verification.
"
```

- [ ] **Step 9: Push** (optional)

```bash
git push
```

---

## Self-Review Checklist

After implementing all 9 tasks, run through this list:

**1. Spec coverage** — every REQ should trace to one or more tasks:

| REQ | Task | Verification |
|---|---|---|
| `invocation-triggers` | T3 step 1 (frontmatter triggers) | T9 step 6 (live invocation) |
| `skill-single-mode` | T3 step 2 (Anti-Pattern section) | Code review |
| `input-validation` | T3 step 3 | T7 stubs 2, 3, 4, 5 |
| `writes-seed-artifact` | T3 steps 5, 6 | T7 stubs 1, 6 |
| `emits-captured-event` | T3 step 7, T4 | T7 stubs 7, 8 |
| `directive-location` | T1 | T9 step 2 |
| `heuristic-capture-cues` | T1 (cues section) | Code review of directive |
| `write-and-continue-discipline` | T1 (discipline section) | T7 stub 10 |
| `host-skill-references` | T5, T6 | T7 stub 9, T9 step 3 |
| `seed-path-convention` | T3 step 6 | T7 stub 1 |
| `seed-frontmatter-schema` | T2, T3 step 5 | T7 stubs 1, 8 |
| `seed-slug-derivation` | T3 step 4 | T7 stubs 6, 12 |
| `seed-lint-rule` | T8 (contract); cross-repo implementation | T7 stub 11 |
| `event-payload-schema` | T3 step 7, T4 | T7 stub 8 |
| `third-party-adoption-contract` | T1 (adoption section) | T7 stub 13 |
| `writes-back-link-to-source-artifact` | T3.5 (back-link section) | T7 stubs 14, 16, 17 |
| `source-artifact-path-resolution` | T3.5 (resolution rules) | T7 stubs 14, 17 |
| `back-link-section-format` | T3.5 (locating + entry format) | T7 stubs 14, 15 |
| `back-link-best-effort` | T3.5 (failure semantics) | T7 stub 18 |

**2. Placeholder scan** — search the plan for forbidden patterns:

```bash
grep -nE 'TBD|TODO|fill in|placeholder|implement later|appropriate error handling' spec/plans/sidekick-capture.md
```

Expected: no matches (the only allowed `TODO`-shaped references are in the seed-template's optional example body, which is documentation of a hypothetical seed, not a plan placeholder).

**3. Type consistency** — verify cross-task naming:

- Slug derivation algorithm matches between T3 step 4 and the AC reference in T7 stub 12.
- Frontmatter keys match between T2 (template), T3 step 5 (skill), and T7 stub 1 (verification).
- Event field names match between T3 step 7, T4, and T7 stub 8.
- The literal string `sidekick-idea.captured` is identical wherever it appears.

**4. File-path consistency** — verify every path in the plan is correct:

- `skills/sidekick/SKILL.md` (singular, lowercase, matches the convention of `skills/ideate/SKILL.md`)
- `skills/shared/sidekick-capture.md` (singular hyphenated)
- `spec/ideas/seeds/<slug>.md` (subdirectory of `spec/ideas/`)
- `spec/features/sidekick-capture/_tests/<ac-slug>.md` (sibling of the Feature's README)
- `spec/plans/sidekick-capture-lint-rule-companion.md` (sibling of this plan)

If any task references a path that doesn't match the plan-wide convention, fix it now.

---

## Execution Handoff

Two execution options for this plan:

**1. Subagent-Driven (recommended)** — A fresh subagent per task with two-stage review. Best for: maximum review density; catching cross-task drift; keeping the main session's context light. Sub-skill: `superpowers:subagent-driven-development`.

**2. Inline Execution** — Tasks run in this session with checkpoints between each. Best for: low-overhead linear walks; preserving the user's existing context. Sub-skill: `superpowers:executing-plans`.

The 9 tasks have no inter-task dependencies that prevent ordering changes beyond:
- T1 (directive) before T3 (skill references it)
- T3 (skill) before T9 (rehearsal exercises it)
- T2 (template) before T3 if you want the skill's references link to resolve at commit time
- T4 (event addendum) can be before or after T3 — the link is documentary

Suggested execution order (matches the task numbering): **T1 → T2 → T3 → T3.5 → T4 → T5 → T6 → T7 → T8 → T9.**

(T3.5 was added post-T3-commit when the spec was revised to include source-artifact back-links; it extends the already-committed skill and must land before T4 so the end-to-end rehearsal in T9 can verify the back-link path.)
