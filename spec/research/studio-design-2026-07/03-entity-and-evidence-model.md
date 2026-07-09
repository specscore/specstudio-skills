# 03 · Knowledge Architecture: Entity Model & Evidence Model

## The substrate: facts

```
Fact {
  id            stable hash(subject, predicate, object)
  subject       EntityRef
  predicate     from a small controlled vocabulary (see below)
  object        EntityRef | scalar
  evidence      [Evidence]          // ≥1; the fact's reason to exist
  confidence    derived from evidence classes (not hand-set)
  verified_at   timestamp of last successful re-verification
  status        active | contradicted | retracted
  provenance    which pipeline/human/agent asserted it
}
```

Facts are **append-only with supersession** (a contradicting fact doesn't delete — it links). This preserves the "was this ever true / when did it change" questions that Git answers for code.

### Predicate vocabulary (deliberately small, v1 ≤ 25)

`implements` (repo→product/capability), `provides` (repo→capability), `uses` (product/capability→capability), `fronts` (domain→product), `publishes` (repo→package), `consumes` (repo→package@version), `specifies` (spec→entity), `promotes-to`/`sourced-from` (idea↔feature), `decides-about` (decision→entity), `supersedes`, `verifies` (commit/test→AC), `models` (modelspec→concept), `projects-to` (modelspec→symbol), `deploys-via`, `serves` (domain live-check), `owned-by`, `member-of` (vertical bundling), `aliased-as`, `parked`, `gotcha` (entity→note — the operational-warning predicate; the pnpm-overrides discovery becomes `trackus gotcha "pnpm-workspace overrides pin @sneat/*"` with evidence = the commit that fixed it).

New predicates require a decision record. Vocabulary sprawl is how knowledge graphs die.

## Entity model (deliverable 6)

Entities are **stable-ID'd nouns**; everything else about them is facts. Kinds (v1):

| Kind | ID scheme | Source of existence |
|---|---|---|
| Ecosystem | slug | config |
| Product | slug (+alias table) | registry (ecosystem-map.yaml class) |
| Capability | slug | registry + GraphSpec modules |
| Repository | host/org/name | git |
| Package/Module | registry coords (npm/go) | manifests |
| Specification | repo+path-slug (SpecScore identity rule) | spec trees |
| AcceptanceCriterion | spec-slug#ac:id (exists in SpecScore today) | spec trees |
| Decision | repo+number/slug | decision dirs + seeds with rulings |
| Model / ModelEntity | modelspec URI (`modelspec:///<module>.<Name>` exists today) | ModelSpec files |
| Symbol | repo+package+name (CodeGrapher node) | codegraph snapshots |
| Domain | fqdn | domains.json / registrar |
| Deployment | worker/site id | wrangler/workflows/CF API |
| Segment, Vertical, RevenueModel | slug | registry |
| Person / AgentSession | handle / session-id | git identity, ledgers |
| Question | slug | question library |
| Note/Gotcha | id | write-back API |

**Entity pages are generated views over facts** — there is no separate "page content" to rot. The only authored text on an entity is its *narrative block* (purpose/why), which is itself a fact (`narrative`) with evidence = the authored file + author + date.

## Evidence model (deliverable 8) — the heart of the product

```
Evidence {
  class       one of the ladder below
  pointer     URL/path/commit/query that reproduces the observation
  observed_at timestamp
  observer    pipeline id | human | agent-session
  detail      e.g. "import edge sneat-go-backend/pkg/x → contactus/briefs"
}
```

### The confidence ladder (classes, strongest first)

1. **verified-behavior** — an executed check: live HTTP probe, CI run, test pass, deploy success. *(Tonight: curl of book-online.app beat every document.)*
2. **derived-from-code** — CodeGrapher edges, go.mod/package.json parses, wrangler configs, GA config. Machine-read, deterministic, re-runnable.
3. **declared** — GraphSpec relationships, SpecScore frontmatter (`Source Ideas`, `Promotes To`, `Verifies:` trailers), specscore: annotations, registry entries. Human-authored *in a lintable format* — trustworthy but can lag.
4. **attested** — a human/agent explicitly confirmed an inferred fact (one click; recorded who/when). Attestation *upgrades* an inference; it decays (re-attest after N months).
5. **claimed** — free-text sources: READMEs, landing copy, marketing. Displayed with the grey chip. *(The datatug-core README's false claim is this class's poster child.)*
6. **inferred** — AI-proposed from patterns/similarity. **Quarantined**: visible only in "proposed" trays and to agents who ask for it explicitly, until attested.

**Confidence = f(best class, agreement count, recency, contradiction absence).** Multiple independent classes agreeing is the strong signal ("bookius uses contacts": import edge + GraphSpec + spec reference → effectively certain). Class-5-only facts render with visible caution; class-6 never renders as truth.

### Contradiction handling

When evidence disagrees (spec status="Approved" vs deploy log=live; README says X, go.mod says not-X): Studio creates a **Contradiction item** — a first-class object on the home health strip with both evidence sets, age, and one-click resolutions (fix the doc via CLI verb / retract the fact / attest an exception). The Sneat review's entire "docs-vs-reality" chapter is this feature, running continuously instead of annually.

### Freshness

Every evidence class has a re-verification cadence (behavior: hours; code-derived: on push via snapshot ingest; declared: on repo change; attested: quarterly nag; claimed: never — flagged as decaying). `verified_at` drives the UI's freshness dots. The KPI: **% of rendered facts with green/grey freshness** — Studio's own SLA.

## Link classification (generated / discovered / manual / inferred / verified)

The prompt's question mapped to the ladder: **generated** = class 2 (never hand-maintain what a parser can read); **discovered** = class 1 probes + class 2 scans; **manually maintained** = class 3 declarations, kept small and lint-enforced (SpecScore's existing sync rules are exactly this); **inferred** = class 6, quarantined; **verified** = class 1 or attested. Design rule: *for any relationship, prefer moving it DOWN the maintenance ladder* — if a manual link can become derivable (e.g. product↔repo via feature registrations), build the deriver and retire the manual copy.
