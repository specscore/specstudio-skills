# 10 · Studio's Own Software Architecture

## Shape: a federated indexer with thin surfaces

```
per-repo artifacts (system of record, git)          ─ distributed, Go-modules style
  spec/ trees · codegraph/ snapshots · models ·
  decisions · .specscore/events.jsonl · registries
        │  (pull/webhook per repo)
        ▼
INGESTION ADAPTERS (one per artifact kind; independently versioned)
  specscore · codegraph/INGR · manifests(go.mod/package.json/wrangler) ·
  registry(domains/ecosystem.yaml) · probes(http/CI/deploy) · ledger/gotcha intake
        │  emit facts + evidence
        ▼
FACT STORE (per ecosystem; rebuildable cache — SQLite first, inGitDB-exportable)
  facts · entities · aliases · contradictions · attestation log (durable!) ·
  question library · embeddings
        │
QUERY & ANSWER ENGINE (DTQL for structured retrieval · verifier/freshness scheduler ·
  contradiction detector · confidence calculator)
        │
SURFACES (all consuming the same engine, no privileged path)
  Web UI (islands/lenses)  ·  MCP server (agents)  ·  CLI (`specscore studio ask`)  ·
  doc projections (generated pages)  ·  webhooks (drift alerts → chat)
```

## Principles

1. **Rebuildable cache, durable attestations.** Everything except the attestation log and question library can be deleted and re-derived from repos. Those two are themselves inGitDB collections — git-portable, self-hostable, OVDB-hostable. No other database of record exists.
2. **Adapters are the extension model.** New language/framework/artifact = new adapter emitting facts with evidence. Adapters declare their predicates and evidence classes; the core knows nothing about Go or Angular or Cloudflare. (This is where "hundreds of repos, many stacks" scales: adapter count, not core complexity.)
3. **Read-mostly core.** Studio's only writes: attestations, saved questions, gotcha/ledger intake. All artifact mutations route through the specscore CLI (05) — one lint gate, git as audit trail.
4. **Local-first, same binary.** `specscore studio serve` on a laptop over local clones = the full product (SQLite + local probes). Hosted multi-tenant = same engine, OVDB-backed stores, org auth. The Go-modules analogy holds: private proxy vs public index, one protocol.
5. **Budget-aware everywhere.** Every query path takes a cost budget (tokens for MCP, ms for UI). Degradation is designed (cards→lines→refs), not accidental.
6. **Scale envelope** (design targets): 500 repos · 50k specs · 5M symbols · 10M facts. SQLite with proper indexes handles this on one box; the fan-out is in ingestion, which is embarrassingly parallel per repo. Defer anything (graph DBs, queues, services) that this envelope doesn't force.

## Delivery form

Ship as part of the specscore CLI (`studio` subcommands: `index`, `serve`, `ask`, `mcp`) + a web bundle it serves. The existing Angular `specscore-studio-app` prototype (GitHub-browsing) becomes the seed of the web surface, repointed from "browse GitHub trees" to "render the fact store"; its deep-link scheme (`specscore.studio/app/<repo-uri>?op=…`) generalizes to entity/question URLs. No separate server product until multi-tenant demands it.

## Security & tenancy (enterprise-relevant from day one)

Identity: org SSO at the surface layer only. Authorization: capability grants on fact scopes (OVDB's model, reused) — per-lens, per-entity-kind, per-evidence-class (an auditor may read facts+evidence but not gotchas; a contractor agent may read one product's scope). Air-gap story: local-first mode already is one; probes degrade to class-2 evidence when the network is closed. Telemetry: the CLI's existing opt-in telemetry pattern extends to Studio, never on by default for self-host.
