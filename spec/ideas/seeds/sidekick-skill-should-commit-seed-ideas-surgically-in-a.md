---
captured_by: user
status: queued
---
# Sidekick skill should commit seed ideas surgically in a separate, seed-only commit

Enhancement to the sidekick capture flow. When the sidekick skill (or its host) commits a captured seed, it should do so surgically: a dedicated commit containing ONLY the new seed file(s) and any seed-index updates, never bundled with unrelated working-tree changes. This keeps seed captures isolated, easy to review, and trivial to revert or relocate independently. Pairs with the GitHub-issue seed idea (detect github host, offer to raise an issue, reference it from the seed).
