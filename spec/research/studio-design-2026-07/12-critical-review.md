# 12 · Critical Review of This Proposal

Written as if reviewing a competitor's design. No punches pulled.

## Where it could fail

1. **The evidence machinery is this founder's classic over-investment shape.** Confidence ladders, contradiction objects, attestation decay — it's beautiful, and it could consume two quarters before answering a single question faster than `grep`. *Mitigation is structural, not aspirational:* the phase gates are benchmark-question counts on a real dataset, and Phase 0 ships with exactly two invariant checks, not the full ladder. If Phase 0 doesn't beat grep-with-an-agent on 25 questions, stop.
2. **Cold start on any ecosystem that isn't Sneat.** Sneat is uniquely well-artifacted (spec trees in 19 repos, snapshots in 11). A typical enterprise repo has none of it. The distributed-adoption story softens this (value from manifests+probes alone: deps, deploys, live-checks, hotspots — no SpecScore adoption required), but the demo-gap between "Sneat-grade corpus" and "average corpus" must be measured honestly, or sales demos will overpromise. Adapter-first onboarding (manifests, CI, git history) must be the *first* enterprise experience, with spec adoption as the upsell — not the prerequisite.
3. **Freshness pipelines are an ops product.** Probes, re-verification schedules, webhook ingestion — this is a running service with pager duty, sold by a company whose muscle is CLIs and static workers. Local-first mode limits the blast radius, but hosted-mode ops cost is a real line item the pricing must carry.
4. **The question benchmark can overfit.** The 50 questions came from one night, one ecosystem, one very unusual user (an AI running an audit). Rebuild the benchmark per design-partner org early, or the product optimizes for archaeology over daily development.
5. **Write-back trust is unsolved in the limit.** Agent-deposited gotchas/ledgers are gold until an agent is wrong at scale. Quarantine + attestation covers v1; reputation systems are hand-waved. Keep write scopes narrow (gotchas, ledgers, proposals — never declared artifacts) until failure data exists.
6. **Competition asymmetry.** Backstage/Port/Cortex have distribution and "catalog" mindshare; GitHub could ship a shallow version of islands+search overnight. The defensible core is the evidence substrate + agent-native MCP + git-decentralization — if the team ever drifts into building "a nicer portal UI," the moat evaporates. The UI must remain a thin client of the substrate, provably (same engine serving MCP and UI is the architectural enforcement).
7. **Dogfooding bias.** Studio assumes the SpecScore lifecycle discipline that even its own author's ecosystem only achieved *tonight*, under an agent's supervision. The product must be excellent for orgs that will never run `specscore change-status` — hence probes and manifests as first-class evidence, not just spec ingestion. Watch this bias in every design review.

## What I'd cut first under pressure

Embedding search (lookup + question library covers most), Map/L5 (islands carry the load; graph lovers are a minority), attestation decay (annual nag is fine), lens count (ship Contributor + Agent, add Founder when a founder asks).

## What must not be cut

Evidence chips + freshness dots (the identity), MCP `orient()` + `gotchas()` (the agent wedge and the measurable ROI), contradiction items (the differentiator no portal has), the rebuildable-cache property (the trust story).

## The uncomfortable question

Is this a product or a feature of the CLI? Honest answer: **Phases 0–1 are a feature of the CLI** (`specscore studio serve`) — and that's the right way to ship them. It becomes a product when agents-as-users (Phase 2) works, because that value compounds across every agent session and no CLI-invocation model captures it. If Phase 2's exit test fails, fold Studio back into the CLI and sell the standards. That would be a good outcome too — the design should be worth building even in its smallest honest form.
