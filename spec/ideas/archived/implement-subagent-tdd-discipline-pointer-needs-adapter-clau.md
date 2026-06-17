---
type: sidekick-seed
captured_by: user
status: Implemented
---

# implement subagent TDD discipline pointer needs an adapter clause for tasks where TDD does not apply (docs, config, deletes)

## Observed problem (dogfood finding #4)

The implement dogfood ran against three pure-documentation tasks (create CONTRIBUTING.md, create CHANGELOG.md, edit README.md). `REQ:subagent-prompt-full` item 6 mandates a "TDD discipline pointer" in every subagent's prompt — but TDD has no surface for "file exists with H2 headings." In the dogfood I adapted each subagent prompt by telling them to "verify against the AC's Then clause directly" — all three handled it gracefully.

## Root cause

`REQ:subagent-prompt-full` describes TDD as the default discipline without an adapter for tasks lacking a testable surface (docs, config, deletes, renames, formatting-only). Strict reading would force degenerate "tests" or REQ violations.

## Suggested fix

Refine item 6: "The TDD discipline pointer applies when the task involves behavior change. For tasks with no testable surface (pure-documentation edits, file renames, deletions, formatting-only changes), the prompt SHOULD substitute 'verify against the AC's Then clause directly' — re-read the artifact post-edit and confirm each predicate in Then. This preserves the verification discipline while honoring the actual task shape."

Add an AC: `Given a Plan task whose Verifies: ACs all have Then clauses observable purely by file state, When implement constructs the subagent prompt, Then the prompt includes the AC-verification adapter clause instead of the TDD pointer.`

## Why this matters

The AC-verification adapter is the natural generalization of TDD: verify against the contract, degrade gracefully when no execution surface exists. Captured during the implement dogfood; all three subagents adapted (see commits 52c4a80, 46612aa).
