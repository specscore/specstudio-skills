---
type: rehearse-stub
status: pending
ac: reviewed-event-emitted-on-success
feature: sidekick-consilium
---

# Rehearse: reviewed-event-emitted-on-success

## Scenario (from AC)

**Given** a successful pipeline completion against one queued task
**When** `.specscore/events.jsonl` is inspected after the run
**Then** exactly one new line has been appended: a JSON event with `event: sidekick-idea.reviewed`, the envelope fields from REQ `event-reviewed-emitted`, and a payload containing `verdict`, `roster_snapshot`, and `tokens_total`.

## Verification approach

Capture the line count of `.specscore/events.jsonl` before the run; invoke `/consilium` against a fixture project with one queued task; assert the file grew by exactly one line, parse that line as JSON, and verify `event == "sidekick-idea.reviewed"`, the envelope fields required by REQ `event-reviewed-emitted` are present, and `payload` contains `verdict`, `roster_snapshot`, and `tokens_total`.

---
*This document follows the https://specscore.md/scenario-specification*
