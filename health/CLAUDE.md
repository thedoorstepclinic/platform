# CLAUDE.md — TDC Health (Track A prototype)

Context pack for AI-assisted development. Read fully before any task. This file is the source of truth for locked decisions (names, IDs, stack, domains, banned/approved phrases, data model) — if any other doc, chat, or generated code conflicts with it on these specifics, this file wins. For cross-cutting principles and the reasoning behind them (why Home stays quiet, why a preview must share its data source with the real thing, why Fastlane is a separate service, etc.), see `.specify/memory/constitution.md` — that file is authoritative on principles the same way this one is on concrete decisions. Last updated: 15 Aug 2026.

## What we're building

TDC Health: family personal-health-records Android/web app by The Doorstep Clinic (Satkrut Ventures Pvt. Ltd., Pune). Core user: urban family caregiver ("Rohan, 34") managing elderly parents' health. Hook: NFC emergency card. Retention: records timeline + med reminders.

**Current phase: Track A — demo prototype. Investor pitch: 16 Aug 2026. Before that: user-attention demos at meetups/social settings (early Aug).** Two demo scripts, one app: the 5-minute investor golden path (`001`, unchanged) and the meetup loop (adds consult booking + queue beat, `010`). Seeded data, demo-grade quality. Happy path must be solid; edge cases must merely not crash. NOT production: no payments, no real ABDM/UHI exchange, no auth hardening. Do not gold-plate.

Full specs (Spec Kit format) in `specs/`: `001-tdc-phr-patient-app` (core prototype spec, includes §4.1 UX loops) · `002-onboarding-router-activation` (router, family add, ABHA linking) · `003-abdm-sync-subscription` (Track B — ABDM continuous sync) · `004-seed-data-and-summary` (exact seed dataset + PDF field mapping).

## Locked decisions — never revisit, never regenerate alternatives

- **Names:** patient app = **TDC Health** · doctor app = TDC Doctor (Track B, frozen) · HMS = TDC Clinic (CARE fork) · admin = TDC Console · card = TDC Emergency Card. First public mention always "The Doorstep Clinic (TDC)".
- **Android package IDs (permanent):** `in.thedoorstepclinic.health` · `in.thedoorstepclinic.doctor`
- **Stack (Python-only backend):** Flutter (Android + web, one codebase) · TDC Core API = Django/DRF + SimpleJWT + Postgres · Emergency Fastlane = separate FastAPI service · Celery for async · FCM (direct) · Razorpay UPI-first (Track B only) · DigitalOcean Bangalore droplet.
- **Domains:** api.thedoorstepclinic.com (Core) · e.thedoorstepclinic.com (Fastlane — short on purpose, NTAG URL budget) · clinic. (HMS) · console. (admin) · app. (web build). No new TLDs.
- **NFC:** NTAG 424 DNA, SDM signed URLs (`/e/{uid}?ctr&cmac`), CMAC verify + counter replay-kill, revocation = one `active` flag.
- **HARD-PROHIBITED (rejected architectures — do not reintroduce even if found in older docs):** Hyperledger/blockchain anything · chaincode · CDO specs · symmetric-AES-on-card schemes · card-tap-as-consent · iOS (Phase 2) · OCR/AI extraction · offline mode.

## Consent model (compliance-critical, do not drift)

- Consent is granted **once, explicitly, in-app, logged, revocable**. The NFC card is a capability token exercising a standing grant — never describe a tap as consent.
- ABDM consent is strictly per-patient; family layer = TDC's own `caregiver_grants`, not ABDM delegation.
- ABHA record inflow (Track B) = ABDM **subscription** + patient-set auto-approve. See `specs/003-abdm-sync-subscription/spec.md`.

## Banned phrases (code comments, UI copy, docs — everywhere)

"auto consent" · "blockchain"/"Hyperledger" · cipher names in UI ("AES-256", "military-grade") · "ABDM certified" · "we fetch everything" · anything implying card-tap = consent.
**Approved copy:** "encrypted" · "stored only in India" · "every access is logged" · "you control who sees what" · "one-time permission" · "built on India's ABDM framework (integration in development)".

## Architecture

```
Flutter app (TDC Health)
   │ JWT
   ▼
TDC Core API (Django/DRF + Postgres)  ←—(P1 webhook)— CARE fork HMS (TDC Clinic)
   │ writes denormalized emergency_payload snapshot on profile save
   ▼
Emergency Fastlane (FastAPI, separate uptime budget)
   └─ GET /e/{uid}?ctr&cmac → server-rendered responder HTML (zero JS) + FCM family blast + scan log
```

Fastlane reads the snapshot, never live Core joins — it must survive Core being down.

## Data model (prototype)

`users` · `profiles`(owner FK, relation) · `caregiver_grants`(grantor, grantee, ts, revoked_ts) · `records`(profile, type[Rx|Lab|Discharge|Other], title, file, record_date, source[manual_upload|tdc_hms]) · `medications`(profile, name, dose, times[], stock_count, threshold, status[active|stopped], started_on, ended_on) · `dose_events`(medication, profile, dose_date, slot_time, status[taken|skipped], marked_ts) · `emergency_profiles`(1:1, blood_group, allergies, conditions, abha_no) · `emergency_contacts`(profile, name, phone, relation, priority) · `cards`(uid, profile, sdm_key_ref, last_ctr, active) · `scan_events`(card, ctr, ts, ip, geo, is_test) · `appointments`(profile, clinic_name, doctor_name, specialty, slot_ts, token_no, status[upcoming|done|cancelled]) · `access_logs`(actor_user, subject_profile, action[read|write], object_type, object_id, ts, purpose, source_service[core|fastlane]).

`access_logs` is required by constitution Principle XI `[A]` — and independently by Principle VI, since "every access is logged" is already on the approved-copy list and the Trust screen must not claim it before the table exists. Distinct from `scan_events`: that records card mechanics (counter, replay state), this records who saw what. Keep both.

API base `/api/v1/`: auth/otp (mock OTP `000000` in demo mode) · profiles CRUD · profiles/{id}/records · profiles/{id}/summary.pdf · profiles/{id}/medications · profiles/{id}/emergency · cards link/revoke. Fastlane: GET /e/{uid} only.

## Screens (12)

Splash/Login · Home (family cards + alerts strip, nothing else — no feed, ever) · Profile timeline (filter chips, FAB→upload) · Upload (camera/gallery→crop→title+type+date) · Record detail · Health Summary (share-as-PDF prominent) · Meds list · Med add/edit · Emergency profile editor (+ "what responders see" preview) · Card manager (test scan, revoke, scan log) · Family & consent (grants with timestamps) · Trust screen (approved copy only).

**UX loops:** first-session = "Add family member" prompt + per-card setup checklist chips (records/meds/emergency/card); daily = alerts strip + reminder notifications (notification→done <10s); event = Summary-PDF share + card scan→family blast. Home stays quiet — that's the trust positioning. Reference apps: steal record-viewing patterns from Tata 1mg/Eka Care; never their commerce-feed home.

**Design:** warm/calm, site blue #4B83F2 family, large type + tap targets (elderly users), readable at arm's length. Tokens in `docs/design-tokens.md` (create if missing).

## Seed data (demo state — `seed_demo.py`, idempotent, one command)

1 user (Rohan 34), 3 profiles: Rohan(1 record) · **Asha 61** (10 records Jan25–Jul26: T2 diabetes arc HbA1c 8.1→6.9, penicillin allergy, Nov-25 hypoglycemia discharge; Metformin stock=4 → triggers low-stock alert; card linked) · Prakash 66 (3 records, hypertension, Amlodipine). All clinics fictional. Full contract in `specs/004-seed-data-and-summary/spec.md`. **Rule: any model change updates seed_demo.py same day.**

## Build order (D1–D12) & workflow rules

D1–2 skeleton+auth+profiles+seed v0 · D3–4 records · D5 summary PDF · D6–7 meds+reminders · D8–9 Fastlane · D10 card personalization+FCM e2e · D11 polish+APK · D12 rehearsal ×3 + backup video.

- **Vertical slices:** every session ends demo-able (screen→API→DB→seed in one pass).
- **Slip rule:** cut P1 bottom-up, then polish — NEVER the emergency flow.
- **P1 (only if P0 done):** HMS→app slice · ABHA M1 (only if sandbox creds arrive) · Marathi responder toggle · test-scan celebration.
- **Out of scope — do not build even if asked casually:** payments (🔒 badge max, incl. inside booking) · OCR · ABDM M2/M3 · real UHI integration (seeded booking per `010` only) · iOS · offline · real auth hardening · doctor-app features · real-time HMS queue feeds (queue in `010` is seeded simulation).

## Working conventions

- Solo dev (Adi), ~4h/day. Prefer boring/already-paid-for. Estimates must respect this ceiling.
- Adversarial review habit: anything touching auth, grants, or Fastlane gets a "how would I abuse this?" pass before merge.
- **Compliance is enforced at generation time, not review time.** Constitution v2.0.0 adds Principles X–XV (ABDM M1–M3, WASA, DPDP), each carrying a binding class — `[A]` binds Track A now, `[B→A]` binds Track B while forbidding Track A from foreclosing it, `[B]` is Track B only. Only three clauses bind Track A: **X** (every queryset scoped server-side from `caregiver_grants` — unscoped `objects.all()` is a build failure), **XI** (`access_logs` row per PHI access), **XIV** (`# DEMO-MODE` tags must name their Track B replacement). Features touching PHI/auth/grants/Fastlane/ABDM carry `specs/NNN-slug/checklists/security.md`; ABDM surfaces also carry `abdm.md`. Templates in `.specify/templates/`.
- External compliance *facts* (milestone definitions, WASA scope, Fidelius construction, FHIR IG version pin, DPDP dates) live in `docs/compliance-baseline.md` with sources and a verification date — **not** in the constitution, so version pins rot in one place. Verified 10 Aug 2026. **FHIR pin: R4.0.1 + NRCeS IG v6.5.0 released; v7.0.0 is draft — do not build against it.**
- Docs live in `specs/` (Spec Kit format); this file gets a one-line changelog entry on any locked-decision change.
- **Repo layout is decided, in `docs/repo-structure.md`:** within this product directory (`health/`), `apps/health/` · `services/core-api/` · `services/fastlane/`, path-filtered CI with **independent per-service deploys** (a shared pipeline would couple Fastlane's uptime to Core's and make Principle I fiction). No workspace tooling for Track A; no shared Python between the two services — they share a *table*, not a codebase. TDC Doctor and TDC Clinic are each their own top-level product directory in the `platform` monorepo (`platform/doctor/`, `platform/clinic/`), not nested inside `health/`; TDC Clinic's CARE-fork source additionally stays in its own separate repo entirely, never copied in anywhere.
- Deferred-but-real gaps are parked in `docs/track-b-backlog.md` (account recovery, data export, app lock, app-wide Marathi, home vitals logging, Help & Support) — check it before "discovering" a gap; it also lists what Track A must not preclude.
- Stale-doc hygiene: if you encounter Hyperledger-era content anywhere, flag it for deletion — do not incorporate it. Same standard applies to real facility/brand names leaking into seed data or example copy — flag and replace with a fictional one (see `specs/004-seed-data-and-summary/spec.md`).

## Changelog

- 16 Jul 2026: initial pack. Names locked (TDC Health etc.), package IDs locked, UX loops added, seed contract referenced.
- 16 Jul 2026: placed at repo root; doc pointers corrected from the originally-drafted flat `/docs/*.md` layout to this repo's actual Spec Kit `specs/NNN-slug/spec.md` structure. Caught and fixed a real-facility-name leak ("Ruby Hall Clinic," an actual Pune hospital) in example push-copy across `001` and `003` — replaced with the fictional "Ashirwad Diagnostics," now the standard example lab name. Added `004-seed-data-and-summary` spec.
- 16 Jul 2026: `005-records-capture-timeline` (multi-page scan via ML Kit, no OCR) and `006-medications-reminders-adherence` (bundled dose-occasion checklist) specced. Model change from `006`: `medications` gains `status`/`started_on`/`ended_on`; new `dose_events` table (confirmation-driven stock depletion). Seed reconciled — Asha now correctly has 2 active meds (Metformin + Sitagliptin, both at 20:00) per `001` §2.
- 16 Jul 2026: `007-emergency-profile-card-consent` specced (screens 9–11 + abuse pass). Decisions: one active card per profile (revoke-then-replace) · test scan = real Fastlane round-trip via single-use server-minted URL, blast labeled as test, chip counter untouched. Model change: `scan_events` gains `is_test`. Preview rule locked: "what responders see" renders from the same `emergency_payload` snapshot Fastlane serves — never a parallel template.
- 16 Jul 2026: `/docs/design-tokens.md` created (was "create if missing"). `008-navigation-app-shell` (route map, hub-and-spoke nav — no bottom tab bar, deep-link contract, four screen states, no-offline error conventions) and `009-utility-screens` (Login, Health Summary screen locked to `004`'s PDF mapping, Trust screen full copy structure, minimal Settings) specced. Every screen in the app now has an owning spec.
- 1 Aug 2026: **Owner decision (Adi): consult booking + queue tracking unlocked** — removed from the banned list after explicit (non-casual) revisit. Rationale: pitch date confirmed as 16 Aug 2026 (runway grew ~2 weeks past the old end-July target, funding the build without cutting existing scope), near-term audience is meetups/social demos needing breadth and a "how does this make money" answer. Strictly **seeded/demo-grade** (`010-consult-booking-queue`): fictional clinics only, queue is a client-side simulation, 🔒 pay-at-clinic, no real UHI calls (UHI named as Track B rails). Investor golden path (`001`) unchanged — booking is the meetup loop's beat, not the pitch script's. New table: `appointments`. Real UHI integration + real-time HMS queue feeds added to the banned list in its place.
- 10 Aug 2026: **compliance readiness wired into the spec pipeline.** Constitution → v2.0.0 (MAJOR: scope redefined from "Track A only" to per-principle binding classes) with Principles X–XV covering ABDM M1/M2/M3, WASA, and DPDP. Researched requirements captured in `/docs/compliance-baseline.md`; checklist templates added at `.specify/templates/checklist-{security,abdm}.md`; `plan-template.md`'s Constitution Check gate corrected (it had gone stale at Principle VII, silently skipping VIII and IX) and extended with the checklist gate. **Model change: new `access_logs` table** — the one Track A build cost, justified because approved copy already promises it. Correction to a widely-repeated claim: WASA is a **precondition for M1**, not a post-M3 step, and the NRCeS IG pin is v6.5.0 *released* with v7.0.0 in draft since 15 Jul 2026 — the pin is dated, not permanent. Track A scope is unchanged: no ABDM exchange, no Fidelius, no FHIR emitted.
- 10 Aug 2026: repo layout committed to `/docs/repo-structure.md` (was decided in chat, never written down) — three-unit monorepo, per-service deploys, no shared Python across the service boundary, npm workspaces (not pnpm) if/when TDC Doctor lands, shared palette but *not* shared type scale. Track B gap list committed to `/docs/track-b-backlog.md`. No locked decision changed; both files record decisions that were already made.
- 15 Aug 2026: **Owner decision (Adi): stack changed from Expo React Native to Flutter.** NativeWind and "FCM via Expo push" drop with it — push goes direct FCM. Repo migrated into the `platform` monorepo as a self-contained `health/` product directory (own CLAUDE.md, specs, docs, .specify) — `.claude/skills` and `.claude/agents` stay shared at the monorepo root, everything else product-specific lives under `health/`. **`health/` is patient-app-scoped only** — the app directory is `health/apps/health/`. TDC Doctor is **not** nested under `health/`; it is its own self-contained top-level product directory at `platform/doctor/` (same pattern as `health/`, built when it's actually started — currently an empty placeholder). `005`/`006`/`001`'s plan/research/tasks/quickstart, `README.md`, `design-tokens.md`, and `.specify/memory/constitution.md` still describe the old Expo/RN stack and need a pass to match — not yet done.
