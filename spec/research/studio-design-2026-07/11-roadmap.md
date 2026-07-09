# 11 · Roadmap & Commercial Perspective

Staging rule (anti-over-engineering guard): **each phase must answer more benchmark questions (01) than the last, on the Sneat dataset, before the next begins.** No phase invents a new format; every phase consumes artifacts that already exist or that the CLI already lints.

## Phase 0 — "Read what exists" (weeks)
Indexer over: spec trees (19 sneat repos), codegraph snapshots (11 repos), ecosystem.yaml + domains.json, go.mod/package.json/wrangler parses. Surfaces: `specscore studio ask` CLI + minimal web (entity cards, search-lookup, home health strip with the two known invariant checks: status-vs-live drift, standards contradictions). **Exit test: 25 of the 50 benchmark questions answered with citations.** Dogfood customer: the founder's own agent sessions via MCP `orient()`.

## Phase 1 — Evidence & freshness (1–2 months)
Probes (live domains, CI, deploys), confidence calculator, contradiction items, freshness dots, islands UX, alias table, question library with re-verification. Exit: 40/50 questions; a fresh agent session onboards from `orient()` alone (measured token cost vs tonight's baseline).

## Phase 2 — Agents as co-users (1–2 months)
Full MCP surface incl. write-back (gotchas, ledgers, proposals), attestation flow, gardener jobs (projection matching, alias mining). Exit: two consecutive multi-agent working sessions where zero gotchas had to be relayed by a human (tonight's counter: 4+).

## Phase 3 — Semantic traceability (quarter)
ModelSpec projection discovery + attest-to-annotation codemods; trace panels with gap-marking; coverage-of-spec verb; cross-repo symbol resolution productized in the ingester. Exit: the AC→test gap report for one product is trusted enough to drive a release gate.

## Phase 4 — Multi-tenant & enterprise (quarters)
OVDB-backed hosted mode, capability grants, SSO, Diligence lens, audit exports. Sneat's public repos become the live public demo instance (marketing = the product, running on itself).

## Commercial perspective

**Positioning:** "The answer layer for your engineering organization — every fact cited, every answer fresh, agents onboard in one call." Category-adjacent to internal developer portals (Backstage-the-Spotify-one, Port, Cortex) but differentiated on three axes those cannot follow easily: **evidence/provenance as a primitive**, **agents as first-class users (MCP-native)**, and **decentralized git-native adoption** (no central catalog YAML to force-feed an org — the Go-modules property).

**Who pays (founder direction 2026-07-10: sellable to big corporations, potentially big):**
- *Enterprise platform/architecture groups* — drowning in repos+turnover; buy trust and onboarding speed. Entry: one division, self-host, per-seat + per-agent pricing.
- *AI-heavy engineering orgs* — every agent session currently re-derives context; Studio is directly measurable in token spend (tonight's data: ~50k tokens of orientation per fresh agent, dozens of sessions — the ROI slide writes itself).
- *Technical due diligence* (funds, acquirers) — the Diligence lens as a paid short-term workspace per deal.
- Sneat itself — first customer, public reference instance, and the source of the "one person + agents ran 190 repos with this" story.

**Pricing sketch** (aligned with the sneat.team playbook: real numbers, launch framing): Free (public repos / single user local), Team (per-seat/month, hosted), Enterprise (self-host license + support, from $30–50k/yr), Diligence (per-deal). Agent seats priced separately from human seats — a genuinely new line item competitors don't have language for yet.

**Sequencing with the Sneat wedge:** Studio Phase 0–2 primarily *serves* the Sneat operation (agent productivity = the founder's leverage), so it isn't a distraction from the SMB wedge — it's the factory's own tooling, later sold. The seed-raise narrative gets both: consumer graph story + "we sell the factory's answer layer" as the dev-tools second line's flagship.

## Long-term (the decades framing)
Studio's terminal ambition: the **shared memory layer of software organizations** — where git is the record of code, Studio's substrate is the record of understanding. Standards path: the fact/evidence format published openly (SpecScore family), so ecosystems can exchange knowledge the way Go modules exchange code. If that lands, the moat isn't the viewer — it's being the reference implementation of the format everyone's agents speak.
