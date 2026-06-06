---
type: rehearse-skip-record
feature: sidekick-capture
format: https://specscore.md/scenario-specification
---

# Rehearse: Skipped ACs

## AC: `heuristic-capture-does-not-derail-host`

**Skip reason:** This AC relies on multi-turn agent behavior across a `specstudio:specify` session — specifically, that the host writes the seed, acknowledges in one line, and returns to the next checklist step *in the same agent turn*. Rehearse stubs as currently designed assert against deterministic observables (file contents, event JSON, exit codes); they cannot assert against the *shape* of an agent's transcript-level behavior.

**Coverage approach:** manual transcript review during host-skill QA. A future Rehearse pattern for transcript-shape assertions (e.g., "no user-facing question between the capture line and the next checklist step") would pick this up automatically. Tracked as a Rehearse roadmap item, not blocking Phase 0.

---
*This document follows the https://specscore.md/scenario-specification*
