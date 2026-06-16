---
captured_by: user
status: queued
---
# Sidekick skill should detect GitHub-hosted repos and offer to raise an issue, then reference that issue URL from the seed

Enhancement to the sidekick capture flow. When writing a seed, detect whether the destination repo is hosted on GitHub (via the git remote origin URL and/or specscore.yaml project.host == github.com). If yes, offer to open a GitHub issue for the seed (e.g. gh issue create) and record the resulting issue URL in the seed so the seed references the issue (a frontmatter field such as 'github_issue:' or a link line in the body). This keeps the SpecScore seed and the public issue tracker cross-linked, so contributors discover the seed from the issue and vice versa. Keep it offer-and-confirm (never auto-file); skip silently for non-GitHub or offline hosts.
