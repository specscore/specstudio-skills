# Sidekick Capture Directive

**Status:** Contract — shared by `specstudio:ideate`, `specstudio:specify`, and any host skill that opts in.

## Purpose

Capture promising sideline ideas during focused work **without derailing the primary task**. Hosts write a one-line seed, the seed is durable in `spec/ideas/seeds/`, and the host resumes its checklist immediately. Deliberation, dedupe across sessions, and auto-promotion belong to the consilium (Phase 1) and later phases — never to capture.

## When to capture

The host skill captures a sideline idea when it notices any of these cues during its work, *and* the idea is genuinely out of scope for the current task:

- "would be nice if…"
- "another approach is…"
- "while we're here, we could…"
- "we should also…"
- "as a side-effect, …"
- "tangentially, …"
- "out of scope but…"
- "this reminds me — …"

The list is non-exhaustive. The host makes the final judgment.

## What is NOT a sideline idea

- The host's own current task or any in-scope refinement of the active artifact.
- A clarifying question to the user about the current task.
- Tool-call decisions for the current step.
- An idea the user has already articulated and is actively working on.

When in doubt: if engaging with the idea would *change the next step of the current checklist*, it is not a sideline — surface it normally. If engaging would *defer the current step*, capture it.

## Invocation pattern

When a sideline idea surfaces:

1. Invoke `specstudio:sidekick` with the one-liner. Skill signature:

       /sidekick <one-liner>                              # H1-only seed
       /sidekick --body <markdown> <one-liner>            # one-liner + optional body

   The host SHOULD invoke with **only** the one-liner. The `--body` form is reserved for cases where a single line genuinely cannot capture the idea — for example, when the seed's point is a specific code snippet or a short list of affected places. Routine long bodies defeat the write-and-continue discipline.

2. Acknowledge the capture in a single short line in the host's running output:

       Captured: <slug> at spec/ideas/seeds/<slug>.md

3. Return to the primary task immediately. The host MUST NOT:
   - pause to deliberate the merits of the sideline idea
   - ask the user about it
   - branch into a discussion of it
   - re-invoke `/sidekick` for the same one-liner within the same conversation

## Same-session dedupe responsibility

The host tracks what it has captured in the current conversation. If a cue would re-fire `/sidekick` with an already-captured one-liner, the host MUST NOT re-invoke. It MAY reference the existing seed path in passing.

Cross-session and cross-agent dedupe is **not** a Phase 0 concern. The Phase 1 consilium dedupes panel runs by content-hash on the event payload; duplicate seed *files* across sessions are useful provenance, not clutter.

## Defense-in-depth on body cap

The sidekick skill enforces the 2000-char body cap at write time. `specscore spec lint` enforces it again on every linted seed. This is deliberate redundancy — the skill catches the common case, the lint catches hand-edited seeds. Neither path is sufficient on its own.

## Adoption by third-party host skills

Host skills outside this repo (e.g., `agent-skills:build`, `superpowers:brainstorming`) may adopt this directive by linking to it from their own documentation and following the invocation pattern. No coordination is required.
