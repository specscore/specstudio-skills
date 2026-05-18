---
type: rehearse-stub
status: pending
ac: slug-is-url-safe-lowercase
feature: sidekick-capture
---

# Rehearse: slug-is-url-safe-lowercase

## Scenario (from AC)

**Given** a one-liner containing mixed case, punctuation, and non-ASCII characters
**When** the slug is derived
**Then** the resulting slug matches the regex `^[a-z0-9]+(-[a-z0-9]+)*$`, is at most 60 characters before any disambiguator is appended, and 64 characters after; no uppercase, whitespace, underscore, or non-ASCII character appears in the slug.

## Verification approach

Invoke with a one-liner containing mixed case, punctuation, and non-ASCII (e.g., `"Refactor: Connection Pool — 高速化 (Phase 2!)"`); assert the resulting slug matches `^[a-z0-9]+(-[a-z0-9]+)*$`; assert length ≤ 60.

---
*This document follows the https://specscore.md/scenario-specification*
