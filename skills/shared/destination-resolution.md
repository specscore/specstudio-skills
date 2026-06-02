# Destination Resolution Prompt

**Status:** Contract — shared deliberation-prompt template for skills that must resolve a multi-repo destination before writing an artifact.

## Purpose

When a skill (currently `specstudio:sidekick`; future skills MAY adopt the same pattern) detects that the user's workspace contains multiple SpecScore-managed repos, it MUST ask the host AI agent to deliberate about which repo the artifact belongs in *before* writing. This file holds the prompt template the skill substitutes candidate-repo identity signals into.

Two audiences read this file:

- **The calling skill** — substitutes the template variables and passes the assembled prompt to the host agent. See "Notes for skill authors" at the bottom.
- **The host AI agent** — reads the assembled prompt, deliberates, and responds per the output format contract.

Wording in this file is intentionally iterable per [REQ:helper-prompt-iteration](../../spec/features/sidekick-capture/destination-resolution/README.md#req-helper-prompt-iteration) — every revision MUST preserve the four contract sections (a)–(d) in their declared order. Wording changes do NOT require a Feature revision; contract changes do.

---

## (a) Deliberation instruction

The calling skill substitutes `<seed-one-liner>` with the user's actual one-liner. The agent reads this verbatim.

> You are writing a sidekick seed for the one-liner: **`<seed-one-liner>`**.
>
> The current workspace contains multiple SpecScore-managed repos (listed in section (b) below). Before the seed is written to disk, deliberate about which repo this idea belongs in. Consider:
>
> 1. The seed's content — what subject area does it touch?
> 2. Each candidate repo's identity signals — its `project.repo` value and the names of its top-level Feature directories under `spec/features/*/`.
> 3. The repo whose existing Features and Ideas this seed most naturally extends.
>
> Do not invent capabilities the candidates don't have. If the seed's home is genuinely unclear, use the escape clause in section (d) below — guessing costs the user a relocate ritual.

## (b) Candidate-repo identity contract

The calling skill substitutes this section with one row per candidate. The source project (cwd's repo) MUST be in the candidate list — it is not automatically privileged.

> **Candidates:**
>
> | # | `project.repo` | Features under `spec/features/*/` (top-level + immediate sub-dirs) |
> |---|---|---|
> | 1 | `<repo-slug-1>` | `<comma-separated-feature-dir-names-and-subdirs>` |
> | 2 | `<repo-slug-2>` | `<...>` |
> | … | … | … |

### `project.repo` fallback

When a candidate's `specscore.yaml` does NOT set `project.repo` (the field is optional), the calling skill MUST substitute the candidate's directory basename as the slug. The agent treats the fallback identifier identically to a yaml-supplied `project.repo` — it appears in the table, it's the value the agent picks in section (c)'s `<repo>` slot, and the calling skill's parser accepts it as a valid candidate match.

### Feature-list expansion

The Features cell lists BOTH:

- The names of top-level directories under `spec/features/*/` (one entry each).
- The names of immediate sub-directories under each top-level Feature dir (where they exist), formatted `<top-level>/<sub-dir>`.

Together these MUST be comma-separated and capped at 20 entries per candidate (truncate with `, …` when more exist). If a candidate has no `spec/features/*/` directories, the cell content MUST be the literal `(no Features)`.

The sub-dir enrichment surfaces nested-Feature names — notably the individual skills in a `skills/*/` umbrella, or sub-Features of a parent like `sidekick-capture/destination-resolution`. Without them, a seed naming a sub-Feature by slug would route ambiguously because the agent only sees the umbrella name.

## (c) Output format contract

> Respond with **exactly one line, ≤120 characters total**, of the shape:
>
> ```
> <repo>; <reason>
> ```
>
> Where `<repo>` is one of the `project.repo` values from the table above and `<reason>` is a single short sentence justifying the pick. Do NOT add explanatory paragraphs, multiple lines, or any prefix/suffix outside this line.

The calling skill parses the response per [REQ:parses-agent-response](../../spec/features/sidekick-capture/destination-resolution/README.md#req-parses-agent-response): exactly one line, ≤120 chars total (counting the `; ` separator and the reason), `<repo>` matches a candidate's `project.repo` case-insensitively after whitespace-trim, separator is `;`. Any deviation routes to the retry path.

## (d) Escape clause

> If you cannot confidently pick one repo from the candidate list, respond with the single token:
>
> ```
> UNCERTAIN
> ```
>
> (case-sensitive, alone on its line, no other content). The skill will surface a numbered list to the user for manual selection.
>
> **Prefer UNCERTAIN over guessing in any of these cases:**
>
> - The seed's content describes work that genuinely spans multiple candidates (e.g., a rebrand affecting two repos, a contract change that requires coordinated edits in producer + consumer).
> - The seed names a specific candidate's slug, but ALSO names a second candidate's slug — the routing is multi-target and there's no single "primary" home.
> - No candidate's identity signals (its `project.repo` or its Feature directory names) materially differ from another's for this seed — every candidate is an equally plausible (or implausible) host.
> - The seed's subject area is outside every candidate's apparent scope — none of the candidates' Feature directories suggest a home, and the agent would be inventing a connection.
>
> A wrong pick costs the user a relocate ritual; `UNCERTAIN` costs them one extra prompt. The asymmetry favors `UNCERTAIN` whenever any of the above holds.

---

## Notes for skill authors

When invoking this prompt:

1. **Read this file once.** Substitute `<seed-one-liner>` (section (a)) and the candidates table (section (b)) inline; pass the resulting text to the host AI agent. Substituted strings are written verbatim — do not trim, capitalize, or otherwise normalize the user's one-liner.
2. **Parse the response** per section (c)'s contract. The full parsing rules live in [REQ:parses-agent-response](../../spec/features/sidekick-capture/destination-resolution/README.md#req-parses-agent-response).
3. **On malformed or `UNCERTAIN`, retry exactly once** with the corrective instruction appended to the prompt body:
   > Your previous response was unparseable; reply EXACTLY in `<repo>; <reason>` ≤120 chars, where `<repo>` is one of: `<comma-separated-candidate-slugs>`.
4. **After a second malformed/`UNCERTAIN` response**, fall through to the calling skill's user-facing ask-without-pre-fill prompt — do NOT retry the agent further. The agent has had its two chances; the user picks next.

The prompt wording above may be iterated against captured-seed replay tests per the Feature's [Open Questions](../../spec/features/sidekick-capture/destination-resolution/README.md#open-questions). The four contract sections (a)–(d) and their ordering are stable; wording inside each is iterable.
