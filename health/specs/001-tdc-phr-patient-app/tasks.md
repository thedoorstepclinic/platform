# Tasks: TDC PHR Patient App — Prototype (Track A)

**Input:** `spec.md`, `plan.md`, `data-model.md`, `contracts/`, `quickstart.md`
**Ordering:** follows the spec build order (D1–D12). `[P]` = parallelisable
(different files, no dependency). **The emergency flow is never reordered below
P1 polish** (Constitution Principle I).

**Regenerated 2026-08-15** against the replanned `001` (Flutter stack,
`health/apps/health/` + `health/services/{core-api,fastlane}/` layout, full
Constitution Check gate). Replaces the Expo-era task list. New in this
revision: authorization-scoping (T005) and access-logging (T006) foundation
tasks closing Principles X/XI, an `X-Robots-Tag: noindex` requirement on the
responder page (T024, Principle XIII), a copy-lint scope clarification
(T032), and an NFR-006 daily-loop-latency verification task (T035) — all
tracing to gaps a `/speckit-analyze` pass found open in the prior version.

## Phase A — Setup & Data (D1–2)
- [ ] T001 [Setup] Scaffold Flutter app (`health/apps/health/`): `pubspec.yaml`,
  app shell, `ThemeData` tokens with site-blue `#4B83F2` palette, large
  type/tap targets (Principle VII).
- [ ] T002 [Setup] Scaffold Core API (`health/services/core-api/`) Django/DRF +
  Postgres + SimpleJWT; `DEMO_MODE` flag.
- [ ] T003 [Setup] Scaffold Fastlane (`health/services/fastlane/`) FastAPI
  service + templates.
- [ ] T004 [P] [Data] Implement models per `data-model.md`: users, profiles,
  caregiver_grants, records, medications, emergency_profiles,
  emergency_contacts, cards, scan_events, emergency_payload, **access_logs**
  (Principle XI).
- [ ] T005 [Data] Authorization scoping base (`core-api/phr/`): shared
  `get_queryset()` mixin/base viewset deriving scope from the caller's own
  profiles ∪ profiles reachable via an unrevoked `caregiver_grants` row;
  `get_object()` resolves out of the scoped queryset, never an unscoped
  `.get(pk=...)`. Every profile-scoped viewset in later phases MUST use it
  (Principle X, `contracts/core-api.md` §Authorization Scoping, `research.md`
  R9).
- [ ] T006 [Data] Access-log writer (`core-api/audit/`): shared
  decorator/middleware writing one `access_logs` row in the same transaction
  as each PHI read/write; a failed log write fails the request. Every
  endpoint tagged **[PHI]** in `contracts/core-api.md` MUST call it, and
  Fastlane's scan handler (T025) writes its own row directly (Principle XI,
  `research.md` R10).
- [ ] T007 [Data] `emergency_payload` snapshot builder + save signals on
  emergency/med/contact change (FR-010).
- [ ] T008 [Auth] `POST /auth/otp/request|verify` with mock `000000` (FR-001);
  tag `# DEMO-MODE` with what's unsafe + the Track B replacement on the
  same/next line (Principle XIV).
- [ ] T009 [Feature] Profiles CRUD + assisted-add writing `caregiver_grants`
  (FR-002/003 — the consent moment); scoped via T005, logged via T006.
- [ ] T010 [Seed] `seed_demo.py` v0: Rohan/Asha/Prakash + relations + grants
  (FR-017, Principle V).

## Phase B — Records (D3–4)
- [ ] T011 [Feature] Record upload (camera/gallery via `image_picker` +
  `image_cropper`, crop) + type tag + title + date; `POST
  /profiles/{id}/records/` (FR-005). No OCR. Scoped via T005, logged via T006.
- [ ] T012 [Feature] Timeline screen newest-first + type filter chips (FR-006).
- [ ] T013 [P] [Feature] Record detail (full-screen image/PDF, share, delete).
- [ ] T014 [Seed] Extend seed: Asha 8–10 records / Prakash 3 records.

## Phase C — Health Summary (D5)
- [ ] T015 [Feature] `GET /profiles/{id}/summary.pdf` template render (FR-007).
  Scoped via T005, logged via T006.
- [ ] T016 [Feature] Health Summary screen (`pdf` package render) + share-as-PDF
  via `share_plus` (WhatsApp target) — renders from the same field mapping as
  the PDF, no second hand-maintained template (Principle IX).
- [ ] T017 [Config] Set app display name to **"TDC Health"** in Flutter app
  config / APK label (decision resolved 15 Jul 2026).

## Phase D — Meds & reminders (D6–7)
- [ ] T018 [Feature] Medications CRUD (`times[]`, stock, threshold) (FR-008).
  Scoped via T005, logged via T006.
- [ ] T019 [Feature] `flutter_local_notifications` reminder scheduling from
  `times[]` (8pm demo beat) (`research.md` R4).
- [ ] T020 [Feature] Stock countdown + low-stock badge (days-left ≤ 5).
- [ ] T021 [Seed] Asha Metformin (4 days left) + Prakash 1 med.

## Phase E — Emergency profile & card (D8–10) — CROWN JEWEL, do not cut
- [ ] T022 [Feature] Emergency profile editor + "what responders will see"
  preview (FR-009) — renders from the same `emergency_payload` snapshot
  Fastlane serves (Principle IX). Scoped via T005, logged via T006.
- [ ] T023 [Feature] Fastlane `GET /e/{uid}`: CMAC verify + replay kill
  (`ctr ≤ last_ctr`) (FR-012/013).
- [ ] T024 [Feature] Responder HTML in mandated order, zero-JS, <2s, every
  response sends `X-Robots-Tag: noindex` (FR-012, NFR-001, Principle XIII);
  neutral page for invalid/revoked (FR-015).
- [ ] T025 [Feature] `scan_event` logging (card mechanics, incl. `is_test`)
  **and**, in the same transaction, one `access_logs` row (`actor=null`,
  `source_service=fastlane`) on every valid scan only — neutral-page
  responses log neither (Principle XI) — plus card counter increment on
  real scans only (test scans leave `last_ctr` untouched, `007`) (FR-014).
- [ ] T026 [Feature] FCM family blast — direct via `firebase-admin`, no push
  intermediary (coarse geo optional, degrade gracefully) (FR-014,
  `research.md` R6).
- [ ] T027 [Feature] Card manager: link, activate/revoke toggle, test-scan,
  scan log (FR-011). Scoped via T005, logged via T006.
- [ ] T028 [Feature] Family & consent screen: grants with timestamps + revoke.
- [ ] T029 [Hardware] Write SDM URL to 2 physical cards + spare; test-scan on
  3+ Android phones (D10 risk mitigation).
- [ ] T030 [Seed] Link Asha's card + emergency payload snapshot.

## Phase F — Trust & polish (D11)
- [ ] T031 [Feature] Trust/security screen — approved copy only (FR-016);
  "every access is logged" is now literally true given T006/T025
  (Principle VI).
- [ ] T032 [Guardrail] Copy module + lint check for banned terms / no
  card-tap-as-consent, covering **both** the Flutter app and the
  Fastlane-rendered responder templates (FR-018, Principle III).
- [ ] T033 [Build] Full seed run + Android APK build.
- [ ] T034 [Polish] Accessibility pass (arm's-length readability), alerts strip.
- [ ] T035 [Polish] Daily-loop latency verification: reminder-notification tap
  → resolved action in under 10 seconds, no more than one intermediate screen
  (NFR-006, spec §4.1 Loop 2).

## Phase G — P1 (only if P0 done)
**Note (added 2026-08-22):** `spec.md`'s P1 scope also lists consult booking
(`010`) and care discovery (`011`) — both own full specs and, per `011`,
`011` must build immediately before `010` (find-care is step 1 of the
booking flow). Neither is enumerated as a task here: each gets its own
`tasks.md` once its own `/speckit-plan` + `/speckit-tasks` run (neither has a
`plan.md` yet). This phase only covers the P1 items `001` owns directly.
- [ ] T036 [P1] HMS→timeline slice: Rx from CARE fork lands in timeline
  (shared DB / fake webhook) (FR-019).
- [ ] T037 [P1] ABHA M1 create-via-Aadhaar — sandbox creds arrived
  (2026-08-17), condition resolved; build if P0 time allows, else field
  stays manual.
- [ ] T038 [P1] Marathi (EN/MR) toggle on responder page (FR-020).
- [ ] T039 [P1] Test-scan celebration / shareable moment screen.

## Phase H — Rehearsal (D12)
- [ ] T040 [Demo] Golden-path rehearsal ×3.
- [ ] T041 [Demo] Record backup screen-recording of full golden path.
- [ ] T042 [Demo] Airplane-mode / Core-down test (verify Fastlane still
  renders) (NFR-002).

## Dependency notes
- T004 blocks T005–T009, T011, T018, T022.
- T005 (authorization scoping) and T006 (access-log writer) block every
  **[PHI]**-tagged endpoint task: T009, T011, T015, T018, T022, T027.
- T007 (snapshot) blocks T023–T026 (Fastlane reads snapshot only).
- T023 blocks T024–T026; T027 depends on T023.
- Cut rule: if a day slips, drop from Phase G bottom-up, then Phase F polish —
  **never** Phase E.

## Parallel example
T004, and once done T013 / T019 / T028 touch different files and can run `[P]`
alongside their siblings within a phase.
