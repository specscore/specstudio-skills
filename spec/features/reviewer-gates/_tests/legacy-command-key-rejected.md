# Scenario: a legacy command-keyed gate key is rejected with a migration error

**Status:** pending
**Validates:** [reviewer-gates#ac:legacy-command-key-rejected](../README.md#ac-legacy-command-key-rejected)

## Steps

GIVEN a `specscore.yaml` whose `gates:` block uses the legacy command-keyed form `gates.specify` (a bare skill name) rather than an event key
WHEN a consumer loads the gate (loader Step 1.5 inspects the `gates:` child keys before resolving the gate)
THEN the loader rejects the key with an error pointing at the [reviewer-gates](../README.md) Feature and naming the event key to migrate to (`gates.feature.approved`)
AND no reviewer is dispatched
AND the consumer exits non-zero

---
*This document follows the https://specscore.md/scenario-specification*
