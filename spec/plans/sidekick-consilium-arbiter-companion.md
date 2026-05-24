# Sidekick Consilium Arbiter — Cross-Repo Companion Plan Stub

**Status:** Stub. This plan exists in *this* repo to record the dependency. The actual implementation work happens in [`specscore/specscore-cli`](https://github.com/specscore/specscore-cli).

**Source contract:** REQs `specscore-consilium-verdict-subcommand`, `arbiter-gate-rules`, `arbiter-reproducibility`, and `roster-validation` in [`spec/features/sidekick-consilium/README.md`](../features/sidekick-consilium/README.md).

## What needs to ship in specscore-cli

A new subcommand `specscore consilium verdict` that:

1. Accepts inputs: `--votes <file>`, `--roster <file>`, `--gate <file>` (optional, defaults to strict baseline), `--seed <file>`.
2. Validates the roster per REQ `roster-validation` (≥ 1 per group post-exclude/add, ≤ 12 total, no name collisions, custom-role files parse).
3. Validates each vote against REQ `vote-schema`; rejects malformed votes with a clear error.
4. Applies the 13-step gate algorithm from REQ `arbiter-gate-rules`.
5. Emits YAML output to stdout: `verdict`, `rule_trace`, `excluded_votes`, `denominators`.
6. Returns exit code 0 on successful verdict (including `should-not-implement` and `needs-human-review`); non-zero on validation failure.
7. Is snapshot-testable: same inputs always produce identical stdout (REQ `arbiter-reproducibility`).

## How to verify the rule is live

After the subcommand ships and a SpecScore project upgrades:

```bash
specscore consilium verdict \
  --votes tests/fixtures/votes-unanimous-strong.yaml \
  --roster tests/fixtures/roster-default.yaml \
  --gate tests/fixtures/gate-strict.yaml \
  --seed tests/fixtures/seed-1.md
# Expected stdout includes: verdict: should-implement
```

## Tracking

- **Upstream issue:** [specscore/specscore-cli#8](https://github.com/specscore/specscore-cli/issues/8)
- Until the subcommand ships, the consilium skill (Task 8) cannot complete its arbiter stage and the calibration set (Task 10) cannot run.

---
*This document follows the https://specscore.md/plans-index-specification*
