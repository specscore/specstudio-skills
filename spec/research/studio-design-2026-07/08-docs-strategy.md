# 08 · Generated Documentation Strategy

## The rule

**Anything a machine can observe is generated; humans write only intent.** Docs become *projections* of the fact substrate — rendered, timestamped, and disposable — never a second place where truth is maintained.

| Always generated (projection) | Always hand-written (narrative) |
|---|---|
| Catalogues (products/repos/specs/domains) — the review's four catalogue files rot from the moment of writing; as projections they can't | Why an entity exists (purpose paragraph) |
| Dependency/consumer matrices, version-pin tables | Decision rulings + their reasoning (ADRs) |
| Status/maturity reports, coverage bars, drift lists | Tutorials & onboarding paths ("read next") |
| Deploy/CI/ops runbook *facts* (how X deploys, which secrets) | Vision & strategy documents |
| API/model references from ModelSpec + code | Gotcha *explanations* (the fact links; the prose teaches) |
| The "Start here" ecosystem overview skeleton | The one paragraph of the Shape card (attested, versioned) |

## Mechanics

- Every generated page carries `generated-from: [fact query] · as-of: timestamp · regenerate: command`. A generated page found edited by hand is a **lint error** (the CLI already has the lint muscle for this).
- Narrative blocks live in git as SpecScore artifacts; Studio ingests them as class-3 facts. Editing happens via the normal repo/CLI flow — Studio links to the edit surface, keeping git the audit trail.
- **README policy** for repos: a thin generated header (what this implements, deploy method, gotchas — projected) + hand-written body. The header's honesty is Studio-enforced; the body is human.
- Retirement path for existing docs: each hand-maintained table/list found in the dataset gets classified — *projection candidate* (replace with generated block), *narrative* (keep), *dead* (archive). The Sneat review effectively performed this classification once by hand; Studio makes it a report.

## The deeper point

The Sneat night's most effective knowledge artifacts were: a generated registry (ecosystem.yaml), lint-gated declarations (spec frontmatter), and tiny hand-written rulings (seed decisions, brand rule). The least effective were long hand-written descriptions of observable state. The documentation strategy is just that observation, institutionalized.
