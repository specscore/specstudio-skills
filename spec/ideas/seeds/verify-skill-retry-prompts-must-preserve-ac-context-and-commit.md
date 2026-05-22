---
type: sidekick-seed
slug: verify-skill-retry-prompts-must-preserve-ac-context-and-commit
captured_at: 2026-05-22T00:00:00Z
captured_by: user
captured_during: skills/verify/SKILL.md
trigger: heuristic
status: queued
synchestra_task: null
---

# verify skill's malformed-verdict retry must preserve full AC body + commit context — subagents are isolated, and a corrective prompt that strips context forces honest 'error' verdicts even when the implementation actually satisfies the AC (surfaced during meta-dogfood run on skills/verify itself: AC 7 errored solely due to retry-context loss)
