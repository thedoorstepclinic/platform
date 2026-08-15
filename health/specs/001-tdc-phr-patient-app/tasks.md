# Tasks: TDC PHR Patient App — Prototype (Track A)

**Input:** `spec.md`, `plan.md`, `data-model.md`, `contracts/`, `quickstart.md`
**Ordering:** follows the spec build order (D1–D12). `[P]` = parallelisable
(different files, no dependency). **The emergency flow is never reordered below
P1 polish** (Constitution Principle I).

## Phase A — Setup & Data (D1–2)
- [ ] T001 [Setup] Scaffold Expo RN app (`app/`) with expo-router, NativeWind,
  shadcn-style tokens; theme with site-blue `#4B83F2` palette, large type/tap
  targets (Principle VII).
- [ ] T002 [Setup] Scaffold Core API (`core-api/`) Django/DRF + Postgres +
  SimpleJWT; `DEMO_MODE` flag.
- [ ] T003 [Setup] Scaffold Fastlane (`fastlane/`) FastAPI service + templates.
- [ ] T004 [P] [Data] Implement models per `data-model.md`: users, profiles,
  caregiver_grants, records, medications, emergency_profiles,
  emergency_contacts, cards, scan_events, emergency_payload.
- [ ] T005 [Data] `emergency_payload` snapshot builder + save signals on
  emergency/med/contact change (FR-010).
- [ ] T006 [Auth] `POST /auth/otp/request|verify` with mock `000000` (FR-001,
  tag `# DEMO-MODE`).
- [ ] T007 [Feature] Profiles CRUD + assisted-add writing `caregiver_grants`
  (FR-002/003 — the consent moment).
- [ ] T008 [Seed] `seed_demo.py` v0: Rohan/Asha/Prakash + relations + grants
  (FR-017, Principle V).

## Phase B — Records (D3–4)
- [ ] T009 [Feature] Record upload (camera/gallery, crop) + type tag + title +
  date; `POST /profiles/{id}/records/` (FR-005). No OCR.
- [ ] T010 [Feature] Timeline screen newest-first + type filter chips (FR-006).
- [ ] T011 [P] [Feature] Record detail (full-screen image/PDF, share, delete).
- [ ] T012 [Seed] Extend seed: Asha 8–10 records / Prakash 3 records.

## Phase C — Health Summary (D5)
- [ ] T013 [Feature] `GET /profiles/{id}/summary.pdf` template render (FR-007).
- [ ] T014 [Feature] Health Summary screen + share-as-PDF (WhatsApp target).
- [ ] T015 [Config] Set app display name to **"TDC Health"** in app config /
  APK label (decision resolved 15 Jul 2026).

## Phase D — Meds & reminders (D6–7)
- [ ] T016 [Feature] Medications CRUD (`times[]`, stock, threshold) (FR-008).
- [ ] T017 [Feature] Expo local reminder notifications from `times[]`
  (8pm demo beat).
- [ ] T018 [Feature] Stock countdown + low-stock badge (days-left ≤ 5).
- [ ] T019 [Seed] Asha Metformin (4 days left) + Prakash 1 med.

## Phase E — Emergency profile & card (D8–10) — CROWN JEWEL, do not cut
- [ ] T020 [Feature] Emergency profile editor + "what responders will see"
  preview (FR-009).
- [ ] T021 [Feature] Fastlane `GET /e/{uid}`: CMAC verify + replay kill
  (`ctr ≤ last_ctr`) (FR-012/013).
- [ ] T022 [Feature] Responder HTML in mandated order, zero-JS, <2s (FR-012,
  NFR-001); neutral page for invalid/revoked (FR-015).
- [ ] T023 [Feature] `scan_event` logging + card counter increment (FR-014).
- [ ] T024 [Feature] FCM family blast (coarse geo optional, degrade gracefully)
  (FR-014).
- [ ] T025 [Feature] Card manager: link, activate/revoke toggle, test-scan,
  scan log (FR-011).
- [ ] T026 [Feature] Family & consent screen: grants with timestamps + revoke.
- [ ] T027 [Hardware] Write SDM URL to 2 physical cards + spare; test-scan on
  3+ Android phones (D10 risk mitigation).
- [ ] T028 [Seed] Link Asha's card + emergency payload snapshot.

## Phase F — Trust & polish (D11)
- [ ] T029 [Feature] Trust/security screen — approved copy only (FR-016).
- [ ] T030 [Guardrail] Copy module + lint check for banned terms / no
  card-tap-as-consent (FR-018, Principle III).
- [ ] T031 [Build] Full seed run + Android APK build.
- [ ] T032 [Polish] Accessibility pass (arm's-length readability), alerts strip.

## Phase G — P1 (only if P0 done)
- [ ] T033 [P1] HMS→timeline slice: Rx from CARE fork lands in timeline
  (shared DB / fake webhook) (FR-019).
- [ ] T034 [P1] ABHA M1 create-via-Aadhaar (only if sandbox creds arrive; else
  field stays manual).
- [ ] T035 [P1] Marathi (EN/MR) toggle on responder page (FR-020).
- [ ] T036 [P1] Test-scan celebration / shareable moment screen.

## Phase H — Rehearsal (D12)
- [ ] T037 [Demo] Golden-path rehearsal ×3.
- [ ] T038 [Demo] Record backup screen-recording of full golden path.
- [ ] T039 [Demo] Airplane-mode / Core-down test (verify Fastlane still renders).

## Dependency notes
- T004 blocks T005–T007, T009, T016, T020.
- T005 (snapshot) blocks T021–T024 (Fastlane reads snapshot only).
- T021 blocks T022–T024; T025 depends on T021.
- Cut rule: if a day slips, drop from Phase G bottom-up, then Phase F polish —
  **never** Phase E.

## Parallel example
T004, and once done T011 / T017 / T026 touch different files and can run `[P]`
alongside their siblings within a phase.
