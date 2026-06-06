---
format: https://specscore.md/scenario-specification
---

# Scenario: idea.drafted and idea.updated payloads carry change-context fields

**Validates:** [ideate#req:event-payload-change-context](../README.md#req-event-payload-change-context)

## Steps

GIVEN no Idea exists yet at `spec/ideas/my-idea.md`
WHEN the skill writes the artifact for the first time and lint passes
THEN the emitted `idea.drafted` event payload contains `changed_sections: null`
AND `previous_revision: null`
AND `change_summary: null`

GIVEN the same Idea is edited (still `**Status:** Draft`) — the user adds a new entry to `Outstanding Questions`
AND the edit lints clean
WHEN the skill emits the next `idea.drafted` event
THEN the payload contains `changed_sections: ["Outstanding Questions"]`
AND `previous_revision` is the git SHA of the prior emission's revision
AND `change_summary` is a non-null string of at most two sentences describing the change

GIVEN the user approves the Idea
AND `idea.approved` has fired
AND the user then edits `Recommended Direction` and `MVP Scope` in a single write
AND the edit lints clean
WHEN the skill emits `idea.updated`
THEN the payload contains `changed_sections: ["Recommended Direction", "MVP Scope"]`
AND `previous_revision` is the git SHA at which `idea.approved` fired
AND `change_summary` is a non-null string of at most two sentences

GIVEN an H3 sub-section under `Behavior` is the only thing edited
WHEN the skill emits the event
THEN `changed_sections` lists the parent H2 (`Behavior`) only
AND H3 changes are NOT separately enumerated

---
*This document follows the https://specscore.md/scenario-specification*
