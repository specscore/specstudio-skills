# 07 · Search Architecture

## One bar, three behaviors

The omnipresent bar classifies input and routes:

1. **Lookup** (`sizeus`, `G-4VZZ470CMR`, `facade4contactus`, `debtus.app`) → entity resolution via the **alias table first** (brand/repo/domain/package/measurement-id all resolve to their entity), then jump-to with an L2 card preview. Sub-50ms path.
2. **Question** ("what uses contacts?", "why was synchestra parked?", "which products have no app?") → the answerer (06): retrieve → answer → cite. Backed by the question library cache: repeated questions are pre-answered and re-verified, so the common 80% is instant and cheap.
3. **Concept sweep** ("booking", "payments", "consent") → federated results *grouped by entity kind* — products, capabilities, specs, models, symbols, decisions, domains — each group ranked by the entity's centrality (fact in-degree), not by string frequency. A concept query is an orientation act; the grouping *is* the answer ("'payments': 0 capabilities, 1 idea, 2 landing claims" told the whole Sneat monetization story in one line).

## Indexes (all rebuildable, none authoritative)

- **Alias/entity index** — the crown jewel; small, curated + generated, hot in memory.
- **Fact index** — subject/predicate/object with evidence-class filters ("show only verified").
- **Symbol index** — CodeGrapher's per-repo FTS, federated at ingest with cross-repo resolution.
- **Full-text** — specs, decisions, narratives, READMEs (class-5 flagged), session ledgers.
- **Embedding index** — for concept sweeps and question-library matching; embeddings rank, facts answer (embeddings never generate content).

## Ranking principles

Centrality over frequency; freshness as tiebreaker; lens-aware boosts (Founder lens boosts products/revenue facts; Agent lens boosts gotchas/deploy facts). Every result shows its freshness dot — search must not launder stale knowledge into apparent relevance.

## Scoped search

Every entity page/island scopes the same bar ("search within contactus": its specs, symbols, consumers, decisions). Scope chips are removable — widening a search is one click, and the widening is recorded in the breadcrumb trail.
