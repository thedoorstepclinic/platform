# Phase 0 Research: TDC PHR Prototype

Decisions and rationale for the unknowns surfaced in `plan.md`. Format:
Decision → Rationale → Alternatives rejected.

**Replanned 2026-08-15:** R4–R6 were Expo-specific and are replaced with
Flutter equivalents below; R9–R11 are new, closing the Principle X/XI/XIII
gaps a `/speckit-analyze` pass found undesigned in the original plan.

## R1. Two-service split (Core API + Fastlane)
- **Decision:** Keep the emergency responder in a separate FastAPI service on
  its own subdomain (`e.thedoorstepclinic.com`) with an independent uptime
  budget.
- **Rationale:** Constitution Principle I — the responder MUST survive Core API
  being down. Physical separation makes that guarantee real and demoable
  (airplane-mode / Core-down test on the checklist).
- **Rejected:** Single Django app serving `/e/{uid}` — couples the crown jewel
  to the heaviest, most-likely-to-break service.

## R2. Denormalized `emergency_payload` snapshot vs live joins
- **Decision:** Fastlane reads a per-card denormalized snapshot, refreshed on
  profile/emergency/med/contact save via a Core API signal.
- **Rationale:** Sub-2s render on 4G with zero external dependencies at read
  time; survives Core downtime; trivially cacheable.
- **Rejected:** Live DRF joins at scan time — adds latency and a hard runtime
  dependency on Core exactly when reliability matters most.

## R3. NTAG 424 DNA SDM verification
- **Decision:** Chip mirrors `?ctr={counter}&cmac={cmac}`; Fastlane verifies the
  CMAC with the per-card key (`sdm_key_ref`) and rejects `ctr ≤ last_seen`.
- **Rationale:** This is prototype-grade but real enough to be the story
  ("replay kill"). Monotonic counter defeats naive URL replay.
- **Rejected:** Plain UID URL with no MAC (trivially forgeable, undermines the
  trust narrative even for a demo).

## R4. Reminders — Flutter local notifications
- **Decision:** Schedule local notifications on-device via
  `flutter_local_notifications` from med `times[]`; low-stock badge computed
  as `stock_count / doses-per-day ≤ 5`.
- **Rationale:** No server push infra needed for reminders; works offline on the
  demo phone; the 8pm reminder is scriptable for the golden path; the package
  is the standard, actively-maintained choice for Android+web Flutter local
  scheduling (boring/already-paid-for per Working Conventions).
- **Rejected:** Server-scheduled FCM reminders — over-engineered for a demo and
  fragile on stage.

## R5. Health Summary PDF
- **Decision:** Template render (`pdf` package → PDF bytes) from structured
  emergency/profile fields + record list; share via `share_plus` (WhatsApp
  target). Not AI-generated.
- **Rationale:** Deterministic, clinically reviewable (Sharvari sign-off D6),
  fast, offline. `pdf` + `share_plus` is the standard Flutter pairing for
  generate-then-share flows, avoiding a server round-trip for a static
  template render.
- **Rejected:** LLM-generated summary — non-deterministic, un-reviewable, and
  out of scope (no AI extraction). Server-rendered PDF fetched over the wire —
  unnecessary network dependency for a document built entirely from data
  already on the device.

## R6. FCM family blast — direct FCM
- **Decision:** On valid scan, Fastlane sends FCM directly (`firebase-admin`
  server SDK) to all device tokens linked to the profile's family; include
  coarse geo when the scanner shares it, omit gracefully otherwise. The
  Flutter app registers for push via `firebase_messaging` (no Expo push
  intermediary — dropped with the Expo stack, per `CLAUDE.md` 2026-08-15).
- **Rationale:** The "money shot" — family phones buzz live. Direct FCM avoids
  depending on infrastructure that no longer exists in the app's stack.
- **Rejected:** SMS blast (slower, costs, less magical); requiring geo (breaks
  when the responder denies location).

## R7. Auth — mock OTP in demo mode
- **Decision:** Phone-OTP flow with mock code `000000` behind a `DEMO_MODE`
  flag; real OTP infra is Track B.
- **Rationale:** Keeps the onboarding beat in the story without provisioning an
  SMS gateway. Clearly tagged so Track B replaces it (Principle XIV format:
  what's unsafe + the Track B replacement, same or next line).
- **Rejected:** Real SMS OTP (cost, deliverability risk on stage, out of scope).

## R8. ABHA number
- **Decision:** Free-text field in the emergency profile by default. ABHA M1
  create-via-Aadhaar is P1, buildable if P0 time allows — the sandbox-creds
  condition that gated it is **resolved** (creds arrived 2026-08-17).
- **Rationale:** Keeps the emergency profile de-risked from ABDM integration
  timing either way — creds arriving removed the credential-delay risk, but
  M1 is still cut-bottom-up P1, not P0, so a day slip still falls back to the
  manual field. Copy says "built on India's ABDM framework (integration in
  development)". No FHIR touched, so Principle XII stays inert for this
  feature.
- **Rejected:** Blocking the emergency profile on live ABHA integration.

## R9. Authorization scoping (Principle X)
- **Decision:** Every DRF viewset touching `profiles`, `records`,
  `medications`, `emergency_profiles`, `cards`, or `scan_events` overrides
  `get_queryset()` to `Profile.objects.filter(Q(owner=user) |
  Q(id__in=active_caregiver_grant_profile_ids(user)))` (or the equivalent
  join for child tables). `get_object()` always resolves out of that scoped
  queryset — never an unscoped `.get(pk=...)` followed by a permission check.
  A revoked grant excludes the profile on the very next request (no cached
  scope).
- **Rationale:** Constitution Principle X is `[A]`-binding: a scoping bug is
  OWASP API #1 and, on stage, would show a second family's data. Cheap to
  build as a mixin/base viewset now; expensive to retrofit across every
  endpoint later.
- **Rejected:** Per-endpoint manual permission checks after an unscoped
  fetch — the exact pattern Principle X names as a build failure, because it's
  easy to forget on any new endpoint.

## R10. Access logging (Principle XI)
- **Decision:** New `access_logs` table (see `data-model.md`), written in the
  same DB transaction as the PHI read/write it records — a failed log write
  fails the request. Core API writes one row per PHI-touching request
  (actor, subject profile, action, object type+id, purpose, `source_service =
  "core"`). Fastlane writes one row per **valid** scan (`actor = null`,
  subject profile = the card's bound profile, action `read`, object type
  `emergency_payload`, `source_service = "fastlane"`) in the same
  request/transaction as the snapshot read — neutral-page (invalid/revoked/
  replay) responses write nothing, since no PHI was accessed.
- **Rationale:** `access_logs` is already named in `CLAUDE.md`'s global data
  model (added when the constitution went to v2.0.0); this plan is what wires
  it into `001`. Distinct from `scan_events` (card mechanics: counter, replay
  state) — both tables are kept per constitution guidance. Without this
  table, the Trust screen's "every access is logged" claim and the
  responder page's "Access logged & family notified" banner are not
  literally true, which Principle VI forbids shipping.
- **Rejected:** Logging only at the Fastlane layer (`scan_events` alone) —
  doesn't cover Core API reads/writes of profiles, records, meds, and
  emergency data, which is most of the app's PHI surface.

## R11. Fastlane public-page hardening (Principle XIII, Track A clause)
- **Decision:** Responder page is served HTTPS-only; PHI appears only in the
  rendered HTML body, never in the query string (`ctr`/`cmac` only); every
  response sends `X-Robots-Tag: noindex`; access/error logs record `uid`,
  `ctr`, `ip`, timestamp — never blood group, allergies, conditions, or
  contact details.
- **Rationale:** This is the one part of Principle XIII that binds Track A
  directly (not just "don't foreclose Track B") — the responder page serves
  real PHI over a public, unauthenticated URL, so these controls are live on
  demo day, not deferred.
- **Rejected:** Relying on obscurity (unguessable UID) alone — the CMAC/replay
  design already exists for exactly this reason; noindex and log hygiene are
  the cheap remaining pieces.

## Resolved
- App display name — **"TDC Health"** (decided 15 Jul 2026, Soham).
- Stack — **Flutter**, replacing Expo React Native (decided 15 Aug 2026, Adi;
  see `CLAUDE.md` changelog). This research.md reflects that decision.
