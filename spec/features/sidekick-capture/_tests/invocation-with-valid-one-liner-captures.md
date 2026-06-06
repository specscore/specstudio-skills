---
type: rehearse-stub
status: pending
ac: invocation-with-valid-one-liner-captures
feature: sidekick-capture
format: https://specscore.md/scenario-specification
---

# Rehearse: invocation-with-valid-one-liner-captures

## Scenario (from AC)

**Given** a Claude Code session in a project where `specstudio:sidekick` is installed and `spec/ideas/seeds/` may or may not exist
**When** the user invokes `/sidekick We should persist debug logs across restarts`
**Then** a file is written at `spec/ideas/seeds/we-should-persist-debug-logs-across-restarts.md` with frontmatter containing exactly the eight keys from REQ `seed-frontmatter-schema`, `type: sidekick-seed`, `trigger: explicit`, `status: queued`, `synchestra_task: null`; the body's first non-blank line is an H1 (`# We should persist debug logs across restarts`) containing the verbatim one-liner; the total body length is ≤ 2000 characters; a `sidekick-idea.captured` event is emitted; the skill returns the relative seed path.

## Verification approach

Run `specstudio:sidekick "We should persist debug logs across restarts"` in a fixture project; assert `spec/ideas/seeds/we-should-persist-debug-logs-across-restarts.md` exists with the expected frontmatter; assert one line was appended to `.specscore/events.jsonl`.

---
*This document follows the https://specscore.md/scenario-specification*
