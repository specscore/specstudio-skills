## Contents

| Child | Description |
|---|---|
| [ambiguous](ambiguous/README.md) | TODO: Add description. |

# Sidekick Consilium Calibration Set

20 seed fixtures used by REQ `calibration-set-20-verdicts` to validate that the consilium produces verdicts matching post-hoc human judgment ≥95% of the time, and that adversaries correctly flag known-weak seeds ≥80% of the time.
| [out-of-domain](out-of-domain/README.md) | TODO: Add description. |
| [strong](strong/README.md) | TODO: Add description. |
| [weak](weak/README.md) | TODO: Add description. |

## Categories

| Category | Count | Expected verdict pattern |
|---|---|---|
| `strong/` | 5 | High confidence `should-implement` from most non-abstain roles |
| `weak/` | 5 | Adversary `should-not-implement` with high confidence in ≥4 of 5; overall verdict `should-not-implement` or `needs-human-review` |
| `out-of-domain/` | 5 | Multiple customer roles high-confidence abstain; verdict still produced (likely `should-implement` if builders converge) |
| `ambiguous/` | 5 | Verdict `needs-human-review`; split votes; rule_trace shows mixed signals |

## Running calibration

After the cross-repo arbiter and task type ship (see Task 11), run:

```bash
# Capture each calibration seed as a sidekick (against this calibration directory as captured_during)
for seed in spec/features/sidekick-consilium/_calibration/*/*.md; do
  /sidekick "$(grep -A1 '^# ' "$seed" | tail -1)"
done

# Drain the queue
/consilium

# Inspect the 20 verdicts
synchestra task list --type consilium-review --status complete
```

For each verdict, the human reviewer notes whether they would have made the same call. Calibration passes if ≥ 95% match.

## Open Questions

- None at this time.

---
*This document follows the https://specscore.md/calibration-specification*
