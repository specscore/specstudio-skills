# CLI Detection Convention

How every specstudio skill detects the `specscore` CLI. **One mechanism; only the response varies by skill class.**

## The one mechanism

Do **not** use a standalone `command -v` probe. Invoke the relevant `specscore` command and branch on its exit status. Four outcomes:

| Outcome | Meaning |
|---|---|
| **success** (exit `0`) | CLI present and the call worked — proceed. |
| **`127`** | Binary not on PATH (not installed). Provided by the shell, *not* by `specscore` — the command never ran, so nothing was mutated. |
| **dedicated "too old / missing subcommand" code** | Binary present but predates a required subcommand. A documented `specscore` exit code (tracked in the `specscore-cli` repo); distinct from `127` and from a generic failure. |
| **any other non-zero** | The command ran and genuinely failed. |

`command -v` is never necessary: `specscore --version` is a cheap read-only call you can branch on, and skills that mutate state or hard-require the CLI run the real command regardless. A `127` cleanly means "absent" without a separate probe.

## The response table

Only the response to each outcome varies by class:

| Class | success | `127` (absent) | dedicated too-old code | other non-zero |
|---|---|---|---|---|
| **Optional** (`ideate`, `sidekick`) | use CLI | take the fallback | (treat as other non-zero) | surface error; do **not** fall back |
| **Mandatory** (`relocate-idea`) | proceed | install message → stop | (treat as other non-zero) | surface error |
| **Capability-gated** (`consilium`) | proceed | install message → stop | **upgrade** message | surface error |
| **Wizard** (`init`) | parse version, offer update if newer | offer install | (treat as other non-zero) | surface error |

## Two rules that follow

1. **Never fall back on a non-`127` error.** Falling back on *any* failure would mask a real CLI bug with a hand-written artifact and risk a double-write after a partial mutation. Optional-class skills take the fallback **only** on `127`.
2. **Check before expensive or mutating work.** Capability-gated skills (e.g. `consilium` needing `consilium verdict`) MUST run the detection branch *before* costly steps such as a multi-agent panel — never after.

## The install / upgrade messages

- **Absent (`127`)** — point the user at `/specscore:install`.
- **Too old (dedicated code)** — tell the user to upgrade, and name the missing subcommand.
