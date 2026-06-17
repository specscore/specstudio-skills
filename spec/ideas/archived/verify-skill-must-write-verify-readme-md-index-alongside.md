---
type: sidekick-seed
captured_by: user
status: Implemented
---

# verify skill must write _verify/README.md index alongside per-run report — lint readme-exists rule fails without it (surfaced during first dogfood run on sidekick-capture)

## Resolution

Implemented in skills/verify/SKILL.md (steps 8 + Report Format → Index README): creates/updates spec/features/<slug>/_verify/README.md so the readme-exists lint rule passes.
