# Rehearse stubs — `sidekick-capture` Feature

Rehearse stubs validating the [Sidekick Capture feature](../README.md). Each stub references an Acceptance Criterion from the Feature spec, copies its Given/When/Then verbatim, and sketches a 2–4-sentence verification approach. Stubs land with `status: pending`; authoring the actual scenario steps follows the implementation plan.

## Index

| Stub | Validates |
|---|---|
| [invocation-with-valid-one-liner-captures.md](invocation-with-valid-one-liner-captures.md) | `sidekick-capture#ac:invocation-with-valid-one-liner-captures` |
| [empty-or-whitespace-input-rejected.md](empty-or-whitespace-input-rejected.md) | `sidekick-capture#ac:empty-or-whitespace-input-rejected` |
| [over-length-input-rejected.md](over-length-input-rejected.md) | `sidekick-capture#ac:over-length-input-rejected` |
| [over-length-body-rejected.md](over-length-body-rejected.md) | `sidekick-capture#ac:over-length-body-rejected` |
| [unknown-flag-rejected.md](unknown-flag-rejected.md) | `sidekick-capture#ac:unknown-flag-rejected` |
| [slug-collision-disambiguates-without-overwriting.md](slug-collision-disambiguates-without-overwriting.md) | `sidekick-capture#ac:slug-collision-disambiguates-without-overwriting` |
| [event-emitted-only-on-successful-write.md](event-emitted-only-on-successful-write.md) | `sidekick-capture#ac:event-emitted-only-on-successful-write` |
| [event-payload-conforms-to-schema.md](event-payload-conforms-to-schema.md) | `sidekick-capture#ac:event-payload-conforms-to-schema` |
| [host-skill-references-directive.md](host-skill-references-directive.md) | `sidekick-capture#ac:host-skill-references-directive` |
| [same-session-no-double-capture.md](same-session-no-double-capture.md) | `sidekick-capture#ac:same-session-no-double-capture` |
| [lint-rejects-malformed-seed.md](lint-rejects-malformed-seed.md) | `sidekick-capture#ac:lint-rejects-malformed-seed` |
| [slug-is-url-safe-lowercase.md](slug-is-url-safe-lowercase.md) | `sidekick-capture#ac:slug-is-url-safe-lowercase` |
| [third-party-skill-can-invoke.md](third-party-skill-can-invoke.md) | `sidekick-capture#ac:third-party-skill-can-invoke` |
| [back-link-appended-on-capture.md](back-link-appended-on-capture.md) | `sidekick-capture#ac:back-link-appended-on-capture` |
| [back-link-section-created-when-absent.md](back-link-section-created-when-absent.md) | `sidekick-capture#ac:back-link-section-created-when-absent` |
| [back-link-skipped-on-null-captured-during.md](back-link-skipped-on-null-captured-during.md) | `sidekick-capture#ac:back-link-skipped-on-null-captured-during` |
| [back-link-skipped-on-nonexistent-path.md](back-link-skipped-on-nonexistent-path.md) | `sidekick-capture#ac:back-link-skipped-on-nonexistent-path` |
| [back-link-write-failure-does-not-roll-back-seed.md](back-link-write-failure-does-not-roll-back-seed.md) | `sidekick-capture#ac:back-link-write-failure-does-not-roll-back-seed` |

## Skipped ACs

See [_skipped.md](_skipped.md) for ACs whose verification depends on multi-turn agent behavior and is covered by manual transcript review.

## Outstanding Questions

None at this time.

---
*This document follows the https://specscore.md/scenarios-index-specification*
