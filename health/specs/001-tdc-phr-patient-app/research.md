# Phase 0 Research: TDC PHR Prototype

Decisions and rationale for the unknowns surfaced in `plan.md`. Format:
Decision → Rationale → Alternatives rejected.

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

## R4. Reminders — Expo local notifications
- **Decision:** Schedule local notifications on-device from med `times[]`;
  low-stock badge computed as `stock_count / doses-per-day ≤ 5`.
- **Rationale:** No server push infra needed for reminders; works offline on the
  demo phone; the 8pm reminder is scriptable for the golden path.
- **Rejected:** Server-scheduled FCM reminders — over-engineered for a demo and
  fragile on stage.

## R5. Health Summary PDF
- **Decision:** Template render (expo-print HTML → PDF) from structured
  emergency/profile fields + record list; share via expo-sharing (WhatsApp
  target). Not AI-generated.
- **Rationale:** Deterministic, clinically reviewable (Sharvari sign-off D6),
  fast, offline.
- **Rejected:** LLM-generated summary — non-deterministic, un-reviewable, and
  out of scope (no AI extraction).

## R6. FCM family blast
- **Decision:** On valid scan, Fastlane sends FCM (via Expo push service) to all
  device tokens linked to the profile's family; include coarse geo when the
  scanner shares it, omit gracefully otherwise.
- **Rationale:** The "money shot" — family phones buzz live. Expo push service
  avoids raw FCM credential handling for the demo.
- **Rejected:** SMS blast (slower, costs, less magical); requiring geo (breaks
  when the responder denies location).

## R7. Auth — mock OTP in demo mode
- **Decision:** Phone-OTP flow with mock code `000000` behind a `DEMO_MODE`
  flag; real OTP infra is Track B.
- **Rationale:** Keeps the onboarding beat in the story without provisioning an
  SMS gateway. Clearly tagged so Track B replaces it.
- **Rejected:** Real SMS OTP (cost, deliverability risk on stage, out of scope).

## R8. ABHA number
- **Decision:** Free-text field in the emergency profile. ABHA M1
  create-via-Aadhaar is P1 and **only** if sandbox creds arrive.
- **Rationale:** De-risks the demo against credential delays (top risk). Copy
  says "built on India's ABDM framework (integration in development)".
- **Rejected:** Blocking the emergency profile on live ABHA integration.

## Resolved
- App display name — **"TDC Health"** (decided 15 Jul 2026, Soham).
