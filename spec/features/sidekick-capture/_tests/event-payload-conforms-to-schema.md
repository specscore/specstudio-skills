---
type: rehearse-stub
status: pending
ac: event-payload-conforms-to-schema
feature: sidekick-capture
format: https://specscore.md/scenario-specification
---

# Rehearse: event-payload-conforms-to-schema

## Scenario (from AC)

**Given** a successful capture
**When** the emitted `sidekick-idea.captured` event payload is inspected
**Then** it contains exactly the eight fields specified in REQ `event-payload-schema`, no more and no less; `content_hash` is the SHA-256 hex digest (lowercase) of the trimmed lowercase one-liner; the five mirrored fields (`slug`, `captured_at`, `captured_by`, `captured_during`, `trigger`) match the seed frontmatter exactly.

## Verification approach

Invoke with a known one-liner; parse the most recent line of `.specscore/events.jsonl` as JSON; assert envelope has `event`, `version`, `uuid`, `timestamp`, `actor`, `artifact`; assert payload has exactly `slug`, `captured_during`, `trigger`, `content_hash`; assert `content_hash` equals the SHA-256 of the trimmed lowercase one-liner.

---
*This document follows the https://specscore.md/scenario-specification*
