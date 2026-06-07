---
captured_by: user
status: queued
---

# Move sidekick body-size cap enforcement from lint-time to capture-time

## Observation

The 2000-char body cap on sidekick seeds today fires at `specscore spec lint` time — *after* the seed has been written, committed, and pushed. By then, context has moved on; trimming requires loading state that's no longer fresh.

Concrete instance: 2026-05-20 session captured `idea-skills-should-pick-destination-repo-not-default-to-cwd` at 2571 chars (28% over). The cap was correct to flag it — the seed contained Idea-shaped deliberation content. But the failure surfaced only when a parent `specstudio:ideate` workflow ran lint hours later, by which time the seed was already in git history.

## Suggested change

Move the body-size check inline into `specstudio:sidekick` itself, before the file write. Reject over-cap captures at the capture moment with the existing error message ("Body too long (<N> chars). Max body (incl. H1 line) is 2000 chars."). Author trims immediately while context is hot.

The cap *value* stays at 2000 — it's a forcing function for the workflow boundary between sidekick (capture) and ideate (deliberate). Recent data point: the over-cap seed genuinely had ideate-shaped content in it, validating the boundary.

## Affected skill

- `specstudio:sidekick` (this repo) — body-length validation moves from "discovered at lint time" to "rejected at capture time"

## Not in scope

- Raising the cap (forcing function works; 28% over caught real over-deliberation)
- Removing the cap (collapses the sidekick/ideate distinction)
- Changing the lint rule (lint should still catch seeds that somehow bypass capture-time enforcement)

---
*This document follows the https://specscore.md/sidekick-seed-specification*
