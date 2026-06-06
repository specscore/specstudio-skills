---
type: rehearse-stub
stub_status: pending
ac: slug-collision-disambiguates-without-overwriting
feature: sidekick-capture
format: https://specscore.md/scenario-specification
---

# Rehearse: slug-collision-disambiguates-without-overwriting

## Scenario (from AC)

**Given** an existing file `spec/ideas/seeds/add-caching-to-search.md`
**When** the skill is invoked with a one-liner whose slug derives to `add-caching-to-search`
**Then** a new file is written at `spec/ideas/seeds/add-caching-to-search-2.md`; the existing file is byte-identical before and after; a second such collision produces `-3`; the event payload's `slug` field reflects the disambiguated slug.

## Verification approach

Pre-seed `spec/ideas/seeds/add-caching-to-search.md`; invoke with a one-liner that derives the same slug; assert new file at `spec/ideas/seeds/add-caching-to-search-2.md`; assert original file's content and mtime unchanged; assert event's `slug` field is `add-caching-to-search-2`.

---
*This document follows the https://specscore.md/scenario-specification*
