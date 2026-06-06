---
type: rehearse-skip-record
feature: sidekick-consilium
format: https://specscore.md/scenario-specification
---

# Rehearse: Skipped ACs

## AC: `calibration-set-passes-95-percent`

**Skip reason:** This AC is a *quality gate on the Phase 1 ship decision*, not a runtime test of skill behavior. Rehearse stubs assert against deterministic observables; calibration assertions require human post-hoc judgment ("would you have made the same call?") on each of 20 verdicts. Not automatable in Phase 1.

**Coverage approach:** manual gate per Task 10 of `spec/plans/sidekick-consilium.md`. A future Rehearse pattern for "AC verified by human review of a Synchestra task field" could pick this up automatically. Tracked as a Rehearse roadmap item; not blocking Phase 1.

---
*This document follows the https://specscore.md/scenario-specification*
