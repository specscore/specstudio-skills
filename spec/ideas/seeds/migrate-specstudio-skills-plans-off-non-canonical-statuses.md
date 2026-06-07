---
captured_by: user
status: queued
---
# Migrate specstudio-skills plans off non-canonical statuses to the plan-status-lifecycle vocabulary

specstudio-skills plans currently use non-canonical statuses (`Completed`, `Implementing`, `Deprecated`, `Archived`) that violate the specscore plan status enum (`draft`/`in_review`/`approved`). Once the `plan-status-lifecycle` Idea (specscore) lands — adding execution (`executing`/`blocked`/`implemented`/`failed`) and disposition (`withdrawn`/`superseded`) statuses — migrate/canonicalize these plan files to the new lifecycle vocabulary.
