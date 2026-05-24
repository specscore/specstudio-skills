# Consilium-Review Task Type — Cross-Repo Companion Plan Stub

**Status:** Stub. This plan exists in *this* repo to record the dependency. The actual implementation work happens in [`specscore/synchestra`](https://github.com/specscore/synchestra).

**Source contract:** REQs `consilium-review-task-lifecycle`, `idempotent-task-creation`, and `single-writer-claim-semantics` in [`spec/features/sidekick-consilium/README.md`](../features/sidekick-consilium/README.md).

## What needs to ship in synchestra

A new task type `consilium-review` registered with Synchestra. The type:

1. Supports the state machine: `queued → claimed → in_review → complete | failed | aborted`.
2. Atomically transitions `queued → claimed` (REQ `single-writer-claim-semantics`).
3. Keys tasks by `content_hash` for idempotent creation — a second `orchestrator task create consilium-review` with the same `content_hash` returns the existing task ID (REQ `idempotent-task-creation`).
4. Stores the structured verdict payload from REQ `verdict-source-of-truth-in-task` and the transcript from REQ `pipeline-transcript-capture` as task fields.

## How to verify the type is live

After the task type ships and a Synchestra project upgrades:

```bash
orchestrator task create consilium-review \
  --content-hash abc123 \
  --seed-path spec/ideas/seeds/test.md
# Expected: returns the task ID. Second invocation with same content_hash returns the same ID.

orchestrator task claim <task-id>
# Expected: transitions queued → claimed. Second claim from another process returns "already claimed".
```

## Tracking

- **Upstream issue:** [specscore/synchestra#5](https://github.com/specscore/synchestra/issues/5)
- Until the task type ships, the consilium skill (Task 8) cannot claim tasks or write verdicts.

---
*This document follows the https://specscore.md/plans-index-specification*
