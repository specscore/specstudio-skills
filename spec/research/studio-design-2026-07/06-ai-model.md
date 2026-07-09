# 06 · AI Interaction Model

Premise: within Studio's lifetime, agents outnumber humans as users. Design for the agent first, render for the human — both consume the *same* facts, evidence, and questions. Divergent human/AI surfaces would recreate the docs-vs-reality split at a new layer.

## Four roles for AI in Studio

### 1. AI as reader (context API)
Studio exposes an **MCP server** — the productization of what every agent-session in the Sneat night had to rebuild by grepping:
- `answer(question, lens)` → answer + fact IDs + evidence + freshness (never uncited prose)
- `entity(ref)` → the L2 card as JSON; `facts(subject|predicate|object filters)`; `trace(ac|symbol|model)`
- `orient(ecosystem)` → the Shape card + load-bearing facts + alias table + gotchas — **the ten-minute onboarding, machine-shaped**; an agent's first call replaces ~50k tokens of self-orientation
- `gotchas(entity)` → operational warnings before touching a repo (the pnpm-overrides class)
- Contract: responses are budget-aware (caller passes a token budget; Studio degrades gracefully from cards to one-liners).

### 2. AI as explainer (inside the UI)
Every island's **Ask** box and the global bar route to an answerer that must: retrieve facts first, cite fact IDs inline, state confidence, and *refuse to exceed the evidence* ("no evidence links bookius to payments; nearest facts: …"). Explanations are cacheable knowledge: a good answer to "why does eventius not import calendarius?" is saved to the question library with its citation set, and **re-verified when cited facts change** — the mechanism that turns conversations into navigable knowledge.

### 3. AI as gardener (proposer)
Background jobs propose: missing links (projection matching, 04), alias candidates, narrative drafts for entities lacking one, contradiction diagnoses, question-library entries mined from real chat/session logs. Everything lands as **class-6 inferred, quarantined** in a proposals tray; humans (or authorized agents) attest or dismiss; attestation writes back into the durable stores. The gardener never edits declared artifacts directly — it opens CLI-verb changes (specscore commands) so git remains the audit trail.

### 4. AI as contributor (write-back)
Working agents deposit, via the same MCP:
- `record_gotcha(entity, note, evidence)` — mid-task discoveries (tonight this required a human relaying messages between agents)
- `record_decision(ruling, constrains, evidence)` — routed to a seed/ADR via CLI verbs, not raw facts
- `session_ledger(summary, facts_touched)` — the end-of-session memory pattern, standardized; ledgers are searchable Activity items linked to every entity they touched
Trust model: agent writes are class-6 (or class-3 when they go through lint-gated CLI verbs); an agent's attestations carry its session identity; repeated confirmed contributions can raise an agent principal's default weight (reputation, but earned and inspectable).

## Interaction grammar (human-facing)

- Ask is everywhere, always scoped ("in this ecosystem" / "about this entity").
- Answers render as: direct answer → evidence chips → "how I know this" expander → follow-up chips (generated next questions).
- **No free-floating chat log.** A conversation is a trail of answered questions, each independently addressable, shareable, and saved-able. The chat *is* navigation.

## What AI must never do here

Fabricate relationships into the substrate; answer without citations; silently refresh stale facts (staleness is information); summarize away contradictions (they are content, 01).
