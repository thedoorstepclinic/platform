# Tasks: TDC PHR Patient App — Core Application

**Input:** `spec.md`, `plan.md`, `data-model.md`, `contracts/`, `quickstart.md`
**Ordering:** **by dependency** (constitution v3.0.0 workflow rule 3 — the
P0/P1 ladder and the cut-from-the-bottom slip rule are repealed; no task is cut
to fund another). The D1–D12 shape below is retained as a sensible dependency
order, not a priority ranking. `[P]` = parallelisable (different files, no
dependency). **The emergency flow is never destabilised or reordered behind
anything** (Principle I — now a stability rule).

**Regenerated 2026-08-15** against the replanned `001` (Flutter stack,
`health/apps/health/` + `platform/services/{core-api,fastlane}/` layout
(services moved out of `health/` on 2026-08-20 — see `docs/repo-structure.md`), full
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
- [ ] T002 [Setup] Scaffold Core API (`platform/services/core-api/`) Django/DRF +
  Postgres + SimpleJWT; `DEMO_MODE` flag.
- [ ] T003 [Setup] Scaffold Fastlane (`platform/services/fastlane/`) FastAPI
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
  behind `DEMO_MODE` (off by default), tagged `# DEMO-MODE` with what it
  substitutes + its real path on the same/next line (Principle XIV). The real
  OTP path is `002`'s; this fixture must never be the only path.
- [ ] T009 [Feature] Profiles CRUD + assisted-add writing `caregiver_grants`
  (FR-002/003 — the consent moment); scoped via T005, logged via T006.
- [ ] T010 [Seed] `seed_demo.py` v0: Rohan/Asha/Prakash + relations + grants
  (FR-017, Principle V).
- [ ] T010a [Compliance] **Principle X + XI verification** — the one automated
  test in this feature, and it exists because FR-022 calls an unscoped result set
  *a build failure, not a review comment*. pytest + DRF `APIClient`, two seeded
  users with one grant between them:
  - **Scoping (X, FR-022):** user B receives `404` — not `403` — on every
    **[PHI]**-tagged endpoint addressed with user A's ids. Out-of-scope and
    nonexistent MUST be indistinguishable.
  - **Revocation (X, FR-004/FR-022):** revoking a grant denies on the **very
    next request**. No cached scope.
  - **Logging (XI, FR-021):** each `[PHI]` request writes exactly one
    `access_logs` row with the right actor/subject/action, and a forced
    log-write failure **rolls the whole request back** — the guarantee T006
    makes is otherwise untested.
  - **Static check:** no viewset under `core-api/phr/` uses `objects.all()` or
    `Model.objects.get(pk=...)`. Catches the failure at write time rather than
    at test time.

  Runs at the end of Phase A, after T005/T006/T009. T009 is the only `[PHI]`
  endpoint in Phase A, so nothing ships unverified for long, and every later
  `[PHI]` task (T011, T015, T018, T022, T027) extends this test rather than
  re-deriving the convention. Closes both blocking compliance GAPs in
  `checklists/security.md` (Authorization — negative-auth test; Audit — the
  logged-read path, with T010b covering its file/PDF half).
  *(Suffixed rather than renumbered: `spec.md` and `plan.md` both cite task
  numbers, and a renumber would ripple into three files.)*
- [ ] T010b [Compliance] **Logged-read path for binary responses** — closes the
  second blocking compliance GAP in `checklists/security.md`, recorded 2026-08-17
  and never scheduled. T006's decorator fires on serializer render; a record
  file and `summary.pdf` are `FileResponse`/`StreamingHttpResponse`, so the
  **most sensitive reads in the app are the ones most likely to slip past it.**
  - **Every file byte is served by a scoped Django view** that logs before it
    streams. **No direct media URLs, ever** — no `MEDIA_URL` static path, no
    signed-URL hand-off. A signed URL logs the *issuing*, not the *reading*,
    and a copied URL is then an unlogged PHI read; the Trust screen's "every
    access is logged" would be false in exactly the case that matters.
  - **Web build consequence:** files load through an authenticated `dio`
    request, not a bare `<img src>` / `<iframe src>`. Flag this at T013 and
    T016 — it is the one place the web target diverges from Android.
  - **Throughput:** serving PHI through the app server is the wrong shape at
    scale and acceptable at current volume. Tag `# DEMO-MODE` naming the real
    path (an authenticated object-store proxy that still logs). No external
    gate — this one is ours to build when volume warrants it.
  - **Log shape:** a Health Summary read logs
    `object_type='health_summary'`, `object_id=<profile_id>`,
    `purpose='summary_export'` — deliberately **not** a plain profile read.
    Exporting a profile's whole clinical history to a shareable PDF is a
    different event from glancing at the profile, and a caregiver reading the
    log should see which one happened. `object_type` is already a free string
    (`data-model.md` §access_logs), so this is a documented value, not a
    schema change.
  - **Scope boundary — the export is logged, the share is not.** T016's
    `share_plus` hand-off leaves the app; where the file then goes is outside
    TDC's knowledge and MUST NOT be claimed or implied in the log or on the
    Trust screen. Revisit under Principle XV with DPDP export duties.

  Blocks T011 and T015. Verified by T010a.

## Phase B — Records (D3–4)
- [ ] T011 [Feature] Record upload (camera/gallery via `image_picker` +
  `image_cropper`, crop) + type tag + title + date; `POST
  /profiles/{id}/records/` (FR-005). No OCR. Scoped via T005, logged via T006;
  **file reads served through T010b's logged path, never a direct media URL.**
- [ ] T012 [Feature] Timeline screen newest-first + type filter chips (FR-006).
- [ ] T013 [P] [Feature] Record detail (full-screen image/PDF, share, delete).
- [ ] T014 [Seed] Extend seed: Asha 8–10 records / Prakash 3 records.

## Phase C — Health Summary (D5)
- [ ] T015 [Feature] `GET /profiles/{id}/summary.pdf` template render (FR-007).
  Scoped via T005, logged via T006 through **T010b's binary path** —
  `object_type='health_summary'`, `purpose='summary_export'`.
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
- [ ] T042a [Demo] **Second-user check, by hand.** Log in as a second account,
  attempt to open Asha's timeline / summary / card by id, confirm it fails
  closed. T010a asserts this in CI; this is the demoer having personally
  watched it fail correctly, so the answer to *"can you see my parent's
  records?"* at a meetup is first-hand rather than recited. Also written into
  `quickstart.md` as a standing verification step.

## Dependency notes
- T004 blocks T005–T009, T011, T018, T022.
- T005 (authorization scoping) and T006 (access-log writer) block every
  **[PHI]**-tagged endpoint task: T009, T011, T015, T018, T022, T027.
- T010b (logged-read path for binaries) blocks T011 and T015 — the two tasks
  that serve files. Depends on T006.
- T010a depends on T005, T006, T009, T010b. It blocks nothing — but every `[PHI]`
  task after it (T011, T015, T018, T022, T027) **extends** it rather than
  adding a parallel test file. A `[PHI]` endpoint that ships without a case in
  T010a is the regression this task exists to prevent.
- T007 (snapshot) blocks T023–T026 (Fastlane reads snapshot only).
- T023 blocks T024–T026; T027 depends on T023.
- ~~Cut rule: if a day slips, drop from Phase G bottom-up, then Phase F polish.~~
  **Repealed 6 Sep 2026** (v3.0.0). Nothing is cut. If a day slips, the
  sequence takes longer; Phase E still never gets destabilised (Principle I).

## Parallel example
T004, and once done T013 / T019 / T028 touch different files and can run `[P]`
alongside their siblings within a phase.
