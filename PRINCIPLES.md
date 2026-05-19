# Principles

This document captures the design and implementation principles for everyone — humans and AI agents — working on SpecStudio. **Principles aren't rules.** They're orienting commitments. When you have to make a call that isn't explicitly specified, refer back here.

For skill-specific tenets (lint discipline, gate philosophy, scope decomposition, YAGNI), see [`skills/shared/philosophy.md`](./skills/shared/philosophy.md). The two documents are complementary: this one frames **how we work**; `philosophy.md` frames **how a skill behaves**.

---

## 1. Respect the user's time and attention

The user's attention is the scarcest resource in the loop. Agents are fast and cheap; users are not. Optimize for the latter.

### 1a. Involve the user in decisions that matter; decide the rest

The user owns *what to build*, *what gets gated*, and *what's out of scope*. The agent owns *how to spell it*, *how to organize the code*, and *how to phrase a paragraph*. Don't ask permission for tactical choices — do the work and surface only the structural decisions. When in doubt about which kind a question is: if reversing the decision is cheap, decide and proceed; if reversing is expensive, ask.

### 1b. Batch questions. Don't drip

When the user genuinely needs to make decisions, prepare 2–4 considered questions and ask them **together**. The worst pattern is one-question-at-a-time pings every few minutes that pin the user to the session. A good batch:

- **Uses multiple-choice with explicit options** wherever the axes of choice are known. Open-ended only when the question space itself is unclear.
- **Includes your recommendation** for each choice, with one short sentence of reasoning. The user can override; you saved them a round-trip.
- **Surfaces tradeoffs honestly**, not just "pick A or B." If A is cheaper but has risk X, say so.
- **Fits the user's session shape**: ask the batch in a way that lets them reply, step away, and return later without losing context.

This refines [`philosophy.md`'s](./skills/shared/philosophy.md) principle 12 ("one question at a time when clarity is low") — clarity is rare enough that batching is usually right; reserve single-question mode for the genuinely ambiguous cases.

### 1c. Work in parallel with user idle time

Once questions are out, don't wait. If you're blocked on one decision, find work that isn't blocked:

- Write speculative drafts that the answers will refine
- Run research, lint, or verification on adjacent work
- Refactor your own intermediate artifacts
- Surface inconsistencies you noticed
- Accumulate the *next* batch of questions so when the user returns, the next check-in is also productive

The goal: every time the user comes back, the session has moved forward visibly. The wrong shape is the user returning to find the agent in the same state they left it.

---

## How to add a principle

Open a PR. Principles should be **observable in actual behavior**, not aspirational. If we can't tell from a transcript whether a principle was followed or violated, it's not a principle yet — it's a vibe. Rewrite until it's observable, or drop it.

When a principle conflicts with [`skills/shared/philosophy.md`](./skills/shared/philosophy.md) (the skill-specific tenets), this document wins — but call out the conflict in the PR so we can revise `philosophy.md` in the same change.
