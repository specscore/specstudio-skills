# Verify Reports — `reviewer-gates`

Per-AC verify-verdict reports for the [Reviewer Gates feature](../README.md). Each report is a snapshot at a specific revision of git HEAD, listing every AC ID with its verdict (`pass` / `fail` / `unmapped` / `error`) and a one-line justification. The YAML summary block at the top of each report is the grep target downstream skills (`recap`, `review`, `ship`) consume.

## Contents

| Report | Run revision | Verdict summary |
|---|---|---|
| [e3e71af.md](e3e71af.md) | `e3e71af` | 16 pass / 0 fail / 0 unmapped / 0 error (manually emulated — `specstudio:verify` skill not loaded in session) |

## Open Questions

None at this time.

---
*This document follows the https://specscore.md/verify-reports-index-specification*
