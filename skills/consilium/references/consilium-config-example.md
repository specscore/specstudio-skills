# `specscore.yaml` — `consilium:` Configuration Reference

Phase 1 of the consilium reads optional configuration from a top-level `consilium:` block in `specscore.yaml`. The block has two sub-keys: `roster` and `gate`. **Both are optional.** A project with no `consilium:` block gets the 9-role default roster and the strict-baseline gate.

This reference documents the schema; the canonical contract lives in REQs `roster-exclude-and-custom`, `gate-knob-set`, and `roster-validation` of the [`sidekick-consilium` Feature](../../../spec/features/sidekick-consilium/README.md).

## Minimal configuration (defaults — no `consilium:` block needed)

```yaml
# specscore.yaml — defaults inherited; nothing to write
```

## Roster customization

```yaml
consilium:
  roster:
    exclude:
      - marketing                       # drop the default Marketing role
      - security-ops                    # drop the default Security/Ops role
    custom:
      - name: accessibility
        group: customers
        path: .specscore/roles/accessibility.md
      - name: domain-expert
        group: customers
        path: .specscore/roles/domain-expert.md
```

After this configuration, the active roster has 9 roles:
- **Builders** (3): engineer, architect, qa (defaults)
- **Customers** (4): pm, ux, accessibility, domain-expert (1 default + 2 custom)
- **Adversaries** (2): yagni-cop, skeptic (default; security-ops excluded)

The arbiter validates this roster per REQ `roster-validation`: every group has ≥ 1 member, no name collisions, ≤ 12 total, custom-role files exist and parse.

## Gate-knob customization

```yaml
consilium:
  gate:
    adversary_veto_confidence: high          # high | medium | low
    cost_ceiling: medium                     # low | medium  (🟢-only vs 🟢+🟡)
    complexity_ceiling: medium               # low | medium
    min_median_confidence: medium            # low | medium | high
    require_all_builders: true               # true | false (unanimous vs supermajority ⅔)
    require_all_customers: true              # true | false
```

The defaults above are the strict baseline used by the parent Idea and validated by the 20-verdict calibration set. Projects with different risk tolerance can loosen knobs (e.g., `require_all_customers: false` for projects where customer-side voices are less reliable). The discrete enums match the 3-step ordinal used elsewhere; continuous numeric thresholds are not supported in Phase 1.

## Reading the active configuration

To inspect what configuration is active for a project:

```bash
specscore consilium verdict --print-config
```

(Implementation lands in the cross-repo `specscore-cli` per the [arbiter companion plan](../../../spec/plans/sidekick-consilium-arbiter-companion.md).)

## Future extensions (Phase 2+)

Phase 2 will additively add `consilium.auto_promote:` to the same block (action knobs for what to do on `should-implement` verdicts: draft Feature, draft plan, dry-run, enabled/disabled). Phase 1's schema is designed for non-disruptive extension.
