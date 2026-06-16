---
captured_by: specstudio:specify
status: queued
---
# specify should hard-gate the scope-decomposition evaluation instead of leaving it a skippable checklist item

Today the decomposition check is Pre-Flight step 2 / checklist item 2 in specify — a soft item the agent can skip. In a real session the agent scaffolded one monolithic Feature (PolyModel core model spanning Entity, Collection, Recordset) without ever raising decomposition; the user had to catch it. Unlike lint and the reviewer gate, decomposition is not hard-gated. Proposal: make decomposition an explicit hard checkpoint before `specscore feature new` — the skill must affirmatively assess whether the intent spans multiple concepts/subsystems and, when it plausibly does, surface decomposition options and get a user decision before creating the artifact. Likely applies to ideate (Idea decomposition) and plan too.
