---
format: https://specscore.md/scenario-specification
---

# Scenario: change_summary stays factual — no speculation or editorializing

**Validates:** [ideate#req:change-summary-discipline](../README.md#req-change-summary-discipline)

## Steps

GIVEN the user has just replaced two paragraphs in `Recommended Direction`, switching the proposed approach from CRUD to event sourcing
AND the edit lints clean
WHEN the skill emits the next event
THEN the `change_summary` field reads similar to: "Replaced two paragraphs in Recommended Direction; the proposed approach now uses event sourcing instead of CRUD."
AND the summary is at most two sentences

GIVEN the user has only adjusted whitespace and added a trailing newline
AND the edit lints clean
WHEN the skill emits the next event
THEN the `change_summary` field reads similar to: "Whitespace and trailing-newline cleanup; no semantic content changed."
AND the summary acknowledges the trivial nature of the change explicitly

GIVEN the skill is generating a `change_summary`
WHEN the summary would otherwise speculate about the user's motivation (e.g., "User is rethinking the approach because of performance concerns")
THEN the skill MUST NOT include the speculation
AND MUST instead state only the observable change (e.g., "Replaced two paragraphs in Recommended Direction")

GIVEN the skill is generating a `change_summary`
WHEN the summary would otherwise editorialize (e.g., "This change improves alignment with industry best practices" or "Important changes to the recommended direction")
THEN the skill MUST NOT include the editorial language
AND MUST instead describe what content changed in factual terms

GIVEN the skill produces a draft `change_summary` exceeding two sentences
WHEN the skill is about to emit the event
THEN the skill MUST shorten the summary to at most two sentences before emission

---
*This document follows the https://specscore.md/scenario-specification*
