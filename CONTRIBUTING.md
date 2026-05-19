# Contributing

## Overview

SpecScore Studio is built on a spec-driven contribution model: every change traces back to an approved Idea, Feature, and Plan before code is written. Contributions should respect the lifecycle skills that make this model auditable end-to-end — pre-spec one-pagers, lint-clean Features with Given/When/Then acceptance criteria, ordered Plans with AC coverage, and verification that each task satisfies the ACs it claims to. Drive-by code changes without a corresponding spec artifact are out of scope; if you have an idea, start by capturing it as a SpecScore Idea.

## Development workflow

The canonical lifecycle for any non-trivial change is:

`ideate ⇒ specify ⇒ plan ⇒ implement ⇒ verify ⇒ recap ⇒ review ⇒ ship`

Each arrow is a skill invocation gated on the previous artifact being approved and lint-clean. Before contributing, read:

- [`PRINCIPLES.md`](./PRINCIPLES.md) — the foundational principles that govern how SpecScore Studio skills behave and how contributors should reason about changes.
- [`skills/shared/philosophy.md`](./skills/shared/philosophy.md) — the shared philosophy that all lifecycle skills inherit from.

Both are required reading before opening a pull request.
