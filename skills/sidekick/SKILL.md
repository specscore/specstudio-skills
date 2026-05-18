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

## When to Use

- A host skill (`specstudio:ideate`, `specstudio:specify`, third-party adopter) has detected a sideline idea per `shared/sidekick-capture.md` and is invoking on the host's behalf.
- A user has typed `/sidekick <one-liner>` directly to park an idea.

## Anti-Pattern: Deliberation at Capture

This skill MUST NOT deliberate the merits of the seed, ask the user follow-up questions, scan existing seeds for content duplicates, or trigger any review pipeline. Deliberation is the consilium's job (a separate Feature, not yet built). If a host invocation arrives that requires deliberation, that is a contract violation in the host — capture the seed and return.

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
