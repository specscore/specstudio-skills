---
captured_by: user:alexander.trakhimenok
status: queued
---

# specstudio:implement should trust Go toolchain over stale gopls diagnostics during parallel subagent dispatch

Observed pattern across both `cli-event` (5 batches) and `cli-event-emit` (5 batches) implementation sessions: every batch fired `<new-diagnostics>` system reminders flagging types/functions as `undefined: X` immediately after subagents staged their work. In every case the diagnostics were stale — the language server hadn't re-indexed sibling-written files yet — while `go build`, `go vet`, and `go test -count=1` were always clean.

The implement skill currently has no guidance about this pattern. Each batch wasted cycles investigating the diagnostic, running the toolchain commands manually, confirming staleness, and explaining the discrepancy. Adding one paragraph to the skill that says "trust `go build` / `go vet` / `go test -count=1` over `<new-diagnostics>` LSP messages when the two disagree; LSP diagnostics fired during parallel dispatch are routinely stale" would short-circuit this dance in future sessions.
