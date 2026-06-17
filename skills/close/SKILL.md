---
name: close
description: |
  Retires a single SpecScore artifact — an Idea, Feature, or sidekick seed —
  by driving the appropriate `specscore <kind> change-status` CLI verb with an
  optional/required `--note`, never by hand-editing. One artifact, one
  change-status call: the status transition, the `## Resolution` note, any
  relocation, and the index sync all happen atomically in that single verb.
  Negative transitions (e.g. a seed `Rejected`) require a reason. Close
  confirms the terminal status, drives the verb, and branches on its exit
  code — it never edits the `**Status:**` line, frontmatter, `type:`, or an
  index row, even on CLI failure.
  Trigger: "close", "/close", "specstudio:close", "retire this artifact".
aliases: [close]
---

# Close

Retire one SpecScore artifact when its work is done. `close` is a thin driver
over the `specscore <kind> change-status` CLI verbs: it resolves the artifact's
kind, confirms the terminal status with the user, captures a reason when one is
required, and performs **exactly one** `change-status` call. That single call
does the status rewrite, writes any `## Resolution` note (`--note`), relocates a
seed to `archived/`, and syncs the index — atomically, with rollback on failure.

**Load-bearing invariant:** `close` NEVER hand-edits status. Every transition
goes through the CLI verb — not as a primary path, and not as a fallback on
failure. A hand-edit anywhere is a contract violation.

Implements the [Close Skill Feature](../../spec/features/skills/close/README.md).

## When to Use

- A single Idea, Feature, or sidekick seed has reached end-of-life (shipped,
  superseded, rejected, or parked) and the user wants to retire it.
- The user types `/close <artifact>` or asks to retire/close/reject an artifact.

**Refuse and redirect when:**

- The invocation does not resolve to exactly one existing artifact, or resolves
  ambiguously to more than one kind → stop and ask the user which artifact;
  write nothing. (AC: `resolves-kind-and-verb`)
- The transition would be illegal for the resolved artifact (the CLI returns
  exit `4`) → surface the current status and the legal source set; do not
  retry, do not hand-edit. (AC: `surfaces-illegal-transition`)

## Kind → verb → terminal statuses

`close` resolves the artifact's **kind by location**, which selects the verb and
the candidate terminal statuses:

| Kind | Resolved at | Verb | Candidate terminals |
|---|---|---|---|
| Idea | `spec/ideas/<slug>.md` | `specscore idea change-status` | `Implemented`, `Archived` |
| Feature | `spec/features/<id>/README.md` | `specscore feature change-status` | `Deprecated` |
| sidekick seed | `spec/ideas/seeds/<slug>.md` | `specscore sidekick change-status` | `Implemented`, `Rejected`, `Archived` |

(Resolving the kind is a filesystem check — **not** a CLI call. `close` does not
query the CLI and then call it again.)

## The close flow

1. **Resolve the artifact and kind** (REQ `resolve-artifact-kind`). Take the
   slug / feature-id / path argument and resolve it by location to exactly one
   of the three kinds above. If it resolves to nothing, or ambiguously to more
   than one kind, STOP and ask the user — never guess.

2. **Present the candidate terminals and confirm** (REQ
   `confirm-terminal-before-close`). Closing is terminal, so present the
   candidate terminal status(es) for the resolved kind and obtain an **explicit**
   user choice. Never auto-select a terminal status.

3. **Capture the reason** (REQ `reason-for-negative-transitions`).
   - For a **negative** transition (a seed `Rejected`), a reason is
     **mandatory**. Collect it up front and pass it via `--note`. Do not invoke
     the verb without it — the CLI also enforces this with exit `2`, but
     collecting it first avoids a wasted round-trip.
   - For a **positive/neutral** transition (`Implemented` / `Archived` /
     `Deprecated`), OFFER to record an optional `--note` (the rationale or where
     the work shipped).
   - **When `close` is invoked by another skill** (non-interactive), the reason
     MUST be supplied by the caller as an explicit argument. `close` MUST NOT
     fabricate or auto-generate a reason for a reason-required transition; if the
     caller passed none, refuse and surface the requirement. (AC:
     `ai-caller-must-pass-reason`)

4. **Drive the verb — one call** (REQ `cli-only-transition`). Invoke exactly one
   `specscore <kind> change-status <id> --to=<status> [--note <markdown>]`. That
   single call atomically performs the status rewrite, the `## Resolution` note
   write, any seed relocation + `type:` tag, and the `spec lint --fix` index
   sync, with rollback on any failure. Do **not** issue a second mutating call,
   a follow-up `edit`, or a manual index fix-up.

   > **Single-call guardrail.** If a future need ever makes one call
   > insufficient to close atomically (multi-artifact, cross-repo, or a
   > transition the CLI can't express as one verb), that is the signal to add a
   > dedicated `specscore close` command in the CLI — **not** to multi-call from
   > this skill.

5. **Branch on the exit code** (REQ `branch-on-cli-exit`) — and NEVER fall back
   to a hand-edit on any non-zero:

   | Exit | Meaning | Action |
   |---|---|---|
   | `0` | success | Surface the verb's `<id>: <from> → <to>` line. Done. (AC: `closes-via-cli-on-success`) |
   | `1` | archive collision | Surface the conflict; stop. |
   | `2` | invalid args / missing required reason | Collect the reason and retry, or correct `--to`. |
   | `3` | artifact not found | Surface; stop. |
   | `4` | illegal transition | Surface the current status + legal source set; stop. |
   | `10` | rollback applied (lint/IO) | Surface the error; stop. |
   | `127` | CLI missing | `/specscore:install`, then retry. |
   | `8` | CLI too old | upgrade, then retry. |

## Kind availability (REQ `seed-close-requires-verb`)

Closing a **seed** depends on the `cli/sidekick/change-status` verb (shipped in
`specscore-cli` ≥ v0.12.0). If the installed `specscore` lacks it (exit `127` /
`8` on the `sidekick change-status` call), surface install/upgrade guidance —
do NOT hand-edit the seed as a workaround. Idea and Feature closes are
unaffected (their `change-status` verbs predate this).

## Not Doing

- **Bulk close.** One artifact per invocation. Closing several in one call
  (e.g. a batch of seeds) is out of scope for the MVP — it is N invocations.
- **Reopen / un-close.** `close` only moves an artifact to a terminal status;
  reversing a close is not its job.
- **Deploy, verify, recap, or plan side effects.** `close` records a terminal
  status and nothing else.

## Verification

- [ ] Resolved the argument to exactly one artifact + kind; ambiguity stopped and asked
- [ ] Presented candidate terminals and got an explicit user choice (no auto-select)
- [ ] Negative transition collected a reason; AI-caller closes required a passed-in reason (no fabrication)
- [ ] Performed exactly one `specscore <kind> change-status` call (no second mutating call, no hand-edit)
- [ ] Branched on the verb's exit code; never hand-edited on non-zero
- [ ] Seed close surfaced install/upgrade guidance when the verb was absent

## Red Flags

- Editing the `**Status:**` line, frontmatter `status:`/`type:`, or an index row directly — ever, including as a fallback
- Issuing more than one mutating CLI call for a single close (see the single-call guardrail)
- Auto-selecting a terminal status instead of confirming with the user
- Invoking the verb for a reason-required transition without a reason
- Fabricating a reason for an AI-driven (non-interactive) close
- Retrying after exit `4` (illegal transition) instead of surfacing and stopping
- Hand-editing a seed as a workaround when the `sidekick change-status` verb is missing
