# 09 · Low-Fidelity Wireframes

## Home

```
┌────────────────────────────────────────────────────────────────────────┐
│ ◆ sneat-ecosystem ▾   [ Ask anything… e.g. "what uses contacts?"     ] │
│ Products Capabilities Decisions Models Map Activity        Lens: 👤 ▾ │
├────────────────────────────────────────────────────────────────────────┤
│ HEALTH  ● facts fresh 94%  ⚠ 2 contradictions  ⚠ 3 drift  ✖ 1 CI red  │
│  ⚠ spec says Approved, deploys say live: eventius/rsvp (12d)  [resolve]│
│  ⚠ naming standard differs: backstage vs sneat-specs (11d)   [resolve]│
├──────────────────────────────┬─────────────────────────────────────────┤
│ THE SHAPE (attested 07-09)   │ ACTIVITY                                │
│ One platform, ~30 doors on a │ ● decision: sneat.team reversed → teams │
│ shared graph; consumer doors │ ● deploy: sneat.co ✅ (auto, 22:58)     │
│ + dev-tools second line.     │ ● gotcha: pnpm overrides pin @sneat/*   │
│ [mini-map]      [read more →]│ ● 0.22 floor: 9/11 repos done           │
├──────────────────────────────┴─────────────────────────────────────────┤
│ MY QUESTIONS                        ▲ answer changed                    │
│ • Which doors have conversion instruments?   (5 → was 2)  ●fresh       │
│ • What pins @sneat/* below 0.22?             (2 → was 11) ●fresh       │
└────────────────────────────────────────────────────────────────────────┘
```

## Answered question (the core interaction)

```
┌─ Ask: what breaks if facade changes? ──────────────────────────────────┐
│ ANSWER  facade (sneat-go-core) has fan-in 104 pkgs across 9 products.  │
│ Highest exposure: logistus (61 refs), core-modules (27), debtus (19).  │
│   evidence: [derived · codegraph 07-09] [verified · CI green 07-10]    │
│   how I know this ▾        freshness ● 11h                             │
│ FOLLOW-UPS: [impact by product] [who changed facade recently]          │
│             [is there a contract tier?]                 [save question]│
└────────────────────────────────────────────────────────────────────────┘
```

## Hover card → island (Product: Debtus)

```
hover:                              pinned island (right rail):
┌─ Debtus ── app-live ●─────┐       ┌─ Debtus ─────────────── ⊙ pin ✕ ──┐
│ Track who owes whom.      │       │ Facts · Evidence · Related · Ask  │
│ repo debtus · debtus.app ●│       │ ● live debtus.app (probe 09:12)   │
│ + splitus.app (2 brands)  │       │ ● platform 0.22 [derived 07-10]   │
│ uses: contacts sms money  │       │ ● uses contacts [import+declared] │
│ platform 0.22 ● (was 0.8) │       │ ⚠ GA: splitus prop shared? [claim]│
│ revenue: none · waitlist ✗│       │ gotcha: legacy bots stack in deps │
└───────────────────────────┘       │ [Ask about Debtus…]               │
                                    └───────────────────────────────────┘
```

## Entity page (Capability: contacts) — grouped by question

```
┌ contacts (capability) ● ────────────────────────────────────────────────┐
│ WHAT  Relationship layer: contactus ext + linkage. [narrative, 07-03]   │
│ WHO PROVIDES  ext-contactus(contract) · contactus(impl) · core(linkage) │
│ WHO USES  20 products ▾   version spread: 0.12.1–0.12.3 ●               │
│ RULES  ADR-0005 contactius=brand · ext-<id> naming [decisions →]        │
│ DATA  Contact · ContactBrief · linkage models [projections 3✓ 1?]      │
│ HEALTH  hotspot rank #2 · 0 contradictions · specs 8/10 fresh           │
│ TRACE  [ACs 14/17 verified ▓▓▓░] [tests 11] [open gaps 3 →]             │
└──────────────────────────────────────────────────────────────────────────┘
```

## Trace panel (AC level)

```
┌ feature contactus/sharing #ac:3 ────────────────────────────────────────┐
│ idea ✓→ feature ✓→ AC ✓→ commits(2) ✓→ symbols(5) ✓→ tests ⚠ none      │
│ [declared] [declared] [Verifies: a1b2c3] [derived]   GAP: no test refs  │
│                                            [attest manually] [ask AI]   │
└──────────────────────────────────────────────────────────────────────────┘
```

## Map (L5) — seeded, lens-filtered

```
┌ Map · seed: contacts · lens: dependencies · depth 2 ── [save view] ─────┐
│    (eventius)──uses──▶(calendar)◀──uses──(bookius)                      │
│         ╲                 ▲                                             │
│        uses            provides                                         │
│          ╲            (calendarius impl ⚠ in sneat-libs)                │
│           ▼               │                                             │
│        (contacts)◀──uses──┴──(20 more ▾ collapsed)                      │
│  edges: ── declared+derived  ┄┄ derived only  ⚠ contradiction           │
└──────────────────────────────────────────────────────────────────────────┘
```

Conventions: every `●/⚠` is the freshness/contradiction color primitive; every bracketed chip opens evidence; nothing on any screen is more than one hover away from "how do we know this".
