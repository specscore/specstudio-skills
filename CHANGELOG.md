# Changelog

## 0.0.5

- **PRINCIPLES.md added** — top-level repo principles doc; first principle is "Respect the user's time and attention" with three operational sub-principles.
- **plan Feature revised** — additive revision adding `**Mode:** <full|stub>`, `**Status:**`, and `**Depends-On:**` task body fields; lint rules `P-003` (Depends-On cycle / dangling / self-ref) and `P-004` (placeholder body on done-status task in stub Plan); canonical placeholder body token `<!-- implement: pending -->`.
- **implement skill shipped** — `specstudio:implement` dispatches parallel subagents per task in batches computed from the Plan's dependency graph; stages all changes with a mandatory `Verifies: <feature-slug>#ac:<ac-slug>, ...` commit-message trailer; per-batch user-approval gate.
