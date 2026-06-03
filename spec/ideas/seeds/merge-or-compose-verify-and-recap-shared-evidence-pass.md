---
type: sidekick-seed
slug: merge-or-compose-verify-and-recap-shared-evidence-pass
captured_at: 2026-06-03T00:00:00Z
captured_by: specstudio:verify
captured_during: spec/features/skills/verify/README.md
trigger: explicit
status: queued
synchestra_task: null
---

# merge or compose verify + recap — both run a per-AC serial-subagent pass over the SAME diffs (verify → pass/fail/error/unmapped gate; recap → no-drift/spec-tighter/code-tighter/contradiction advisory), and recap already requires+parses verify's report yet pays a second cold re-read; evaluate (a) a shared per-AC "evidence pass" both consume, (b) enriching verify's report into structured priming input for recap instead of one-line justification, (c) a thin combined entrypoint that runs the evidence pass once and emits both reports — leaning DON'T fully merge (gate-vs-advisory + distinct vocabularies/events/transitions argue for two skills) but DO formalize the handoff; needs ideate with options + recommendation
