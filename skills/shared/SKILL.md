---
name: shared
description: |
  Internal SpecStudio reference material used by the lifecycle skills. Do not
  invoke directly; load specific files from this directory only when another
  SpecStudio skill links to them.
aliases: []
---

# Shared

This directory contains shared SpecStudio reference material for lifecycle skills.
It is exposed as a minimal skill only so Codex plugin ingestion can validate the
existing `skills/` tree, where every first-level directory is expected to contain
a `SKILL.md`.

Do not invoke this directly. Use the concrete lifecycle skills instead:
`init`, `ideate`, `specify`, `plan`, `implement`, `verify`, `recap`, `ship`,
`sidekick`, `consilium`, `relocate-idea`, `survey`, `score`, or `pull-request`.
