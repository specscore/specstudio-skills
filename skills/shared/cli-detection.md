# CLI Detection Convention

How every specstudio skill detects the `specscore` CLI. **One mechanism; only the response varies by skill class.**

## The one mechanism

Do **not** use a standalone `command -v` probe. Invoke the relevant `specscore` command and branch on its exit status. Four outcomes:

| Outcome | Meaning |
|---|---|
| **success** (exit `0`) | CLI present and the call worked — proceed. |
| **`127`** | Binary not on PATH (not installed). Provided by the shell, *not* by `specscore` — the command never ran, so nothing was mutated. |
| **exit `8`** (too old / missing subcommand) | Binary present but predates a required subcommand. `specscore`'s `UnsupportedCommand` (8) — distinct from `127` (absent) and from a generic failure (1). See `specscore`'s `docs/exit-codes.md`. |
| **any other non-zero** | The command ran and genuinely failed. |

`command -v` is never necessary: `specscore --version` is a cheap read-only call you can branch on, and skills that mutate state or hard-require the CLI run the real command regardless. A `127` cleanly means "absent" without a separate probe.

## The response table

Only the response to each outcome varies by class. The four classes are **creation, mandatory, capability-gated, wizard**:

| Class | success | `127` (absent) | exit `8` (too old) | other non-zero |
|---|---|---|---|---|
| **Creation** (`ideate`, `sidekick`, `specify`, `plan`, `init`) | use the CLI scaffold | install message (`/specscore:install`) → offer **install-then-retry** | **upgrade** message → offer **upgrade-then-retry** | surface error; **never** a direct-write fallback |
| **Mandatory** (`relocate-idea`) | proceed | install message → stop | (treat as other non-zero) | surface error |
| **Capability-gated** (`consilium`) | proceed | install message → stop | **upgrade** message | surface error |
| **Wizard** (`init`) | parse version, offer update if newer | offer install | (treat as other non-zero) | surface error |

Artifact creation goes through `specscore … new`, and the producer skills carry no embedded schema — the CLI scaffold is the single source of truth for artifact structure (see the [CLI Detection and Artifact-Creation Convention](../../spec/features/cli-detection-convention/README.md) Feature behavior).

## Two rules that follow

1. **Never fall back on a non-`127` error.** Falling back on *any* failure would mask a real CLI bug with a hand-written artifact and risk a double-write after a partial mutation. Creation-class skills never take a direct-write fallback at all: on `127` they offer install-then-retry, on exit `8` upgrade-then-retry, and on any other non-zero they surface the error.
2. **Check before expensive or mutating work.** Capability-gated skills (e.g. `consilium` needing `consilium verdict`) MUST run the detection branch *before* costly steps such as a multi-agent panel — never after.

## The install / upgrade messages

- **Absent (`127`)** — point the user at `/specscore:install`.
- **Too old (exit `8`)** — tell the user to upgrade, and name the missing subcommand.
