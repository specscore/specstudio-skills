# Recap Reports — `skills/recap`

Per-AC drift reports for the [Recap Skill feature](../README.md). Each report is a snapshot at a specific revision of git HEAD that compares what each AC requires against what the implementation actually delivered, classifying any divergence as `no-drift`, `spec-tighter-than-code`, `code-tighter-than-spec`, or `contradiction` (with orchestrator-produced `unmapped` and `error`). The YAML summary block at the top of each report is the grep target downstream skills (`review`, eventually `ship`) consume.

## Contents

| Report | Run revision | Verify revision | Drift summary |
|---|---|---|---|
| [81143e4.md](81143e4.md) | `81143e4` | `a05f52b` | 15 no-drift · 0 spec-tighter · 0 code-tighter · 0 contradiction · 0 unmapped |

## Open Questions

None at this time.

---
*This document follows the https://specscore.md/index-specification*
