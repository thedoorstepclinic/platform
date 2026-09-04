# Implementation Plan: TDC PHR Patient App — Prototype (Track A)

**Feature:** `001-tdc-phr-patient-app` (directory-scoped; trunk-based on `main`
— no per-feature git branch exists post-monorepo-migration, see note below)
· **Date:** 2026-07-15 · **Replanned:** 2026-08-15
**Spec:** [`spec.md`](./spec.md) · **Constitution:** [`../../.specify/memory/constitution.md`](../../.specify/memory/constitution.md) (v2.0.1)

## Summary
Build a seeded, demo-grade PHR family app for the investor pitch (16 Aug 2026)
and the meetup loop. One **Flutter** codebase (Android + web) talks to a
Django/DRF Core API over SimpleJWT. A separate FastAPI **Fastlane** service
serves the emergency responder page from a denormalized snapshot so it
survives Core API downtime. The golden path — home → timeline →
summary/PDF → meds/reminder → NFC card scan + family blast → trust screen —
must run reliably in 5 minutes and reset in one command.

**Replan note:** This plan was regenerated on 2026-08-15 to (1) replace the
Expo React Native / NativeWind stack with Flutter per `CLAUDE.md`'s
2026-08-15 stack decision, (2) move the project layout under the monorepo's
`health/apps/`, `health/services/` structure per `docs/repo-structure.md`
(the services half of that move was superseded on 2026-08-20 — they now sit at
`platform/services/`, see that file),
and (3) close two Constitution Check gaps — Principle X (authorization
scoping) and Principle XI (access logging) — that a `/speckit-analyze` pass
found undesigned. `research.md`, `data-model.md`, `contracts/`, and
`quickstart.md` were regenerated alongside this file. `tasks.md` was
regenerated on 2026-08-15 via `/speckit-tasks` to match. `spec.md`'s three
stale references (the "shadcn-style tokens via NativeWind" line, the "Expo
skeleton" Build Order label, and the `**Feature Branch:**` field asserting a
git branch that doesn't exist) have since been patched directly (2026-08-15)
— all artifacts in this feature are now stack- and branch-consistent.

**Branch note:** this repo has no `001-tdc-phr-patient-app` git branch,
locally or on `origin` — only `main` exists. `health/` was merged directly
into the `platform` monorepo on `main` (`CLAUDE.md` changelog, 2026-08-15);
feature isolation here is by directory (`specs/NNN-slug/`), not by branch.
The Spec Kit scripts already tolerate this (`CURRENT_BRANCH` falls back to
the feature-directory basename when no real git branch is set), so nothing
upstream of this plan broke — but `spec.md`'s `**Feature Branch:**` field
still names a branch that was never created, which is worth a fix whenever
`spec.md` is next touched.

## Technical Context
- **Language/Runtime:** Dart (Flutter, Android + web); Python 3.11 (Django/DRF, FastAPI).
- **Primary dependencies:** Flutter SDK; `flutter_local_notifications` (med
  reminders); `image_picker` + `image_cropper` (camera/gallery upload + crop);
  `pdf` + `printing`/`share_plus` (Health Summary PDF render + WhatsApp
  share); `firebase_messaging` + `firebase_core` (direct FCM — no Expo push
  intermediary); `dio` (JWT-bearing API client) + `flutter_secure_storage`
  (token storage); Django REST Framework, SimpleJWT, FastAPI, Postgres.
- **Storage:** Postgres (Core API). Fastlane reads a denormalized
  `emergency_payload` snapshot (own table/materialized copy). New:
  `access_logs` table (Principle XI, see Constitution Check).
- **Target platforms:** Android + web only. **No iOS.**
- **Constraints:** Responder page <2s on 4G, zero-JS-readable; Fastlane
  independent uptime budget; elderly-first accessibility; Fastlane responder
  page is HTTPS-only, carries no PHI in query strings/logs/referrer, and is
  non-indexable (Principle XIII, Track A clause).
- **App display name:** **TDC Health** (resolved 15 Jul 2026) — used for APK
  label and app config.

## Constitution Check
*Gate: passed for Phase 0. Re-checked after Phase 1 design (2026-08-15) against
constitution v2.0.1 — the previous version of this plan only checked
Principles I–VII.*

**Core principles — bind unconditionally.**
| Principle | Status | How the plan satisfies it |
|-----------|--------|---------------------------|
| I — Emergency Flow Is Sacred | ✅ | Fastlane is a separate service reading a snapshot; task order front-loads it; cut rule encoded in `tasks.md`. |
| II — Consent Is a Standing Grant | ✅ | `caregiver_grants` write on assisted-add is the consent moment; no card-tap copy anywhere; Fastlane logs "access", never "consent". |
| III — Copy Discipline | ✅ | Approved/banned lists enforced in a shared copy module + a lint check task. |
| IV — Demo-Grade, Stated Honestly | ✅ | Mocks (OTP `000000`, manual ABHA, fake HMS webhook) tagged `# DEMO-MODE` with a replacement note (Principle XIV). |
| V — Reset-in-One-Command | ✅ | `seed_demo.py` is a D1–2 deliverable and a demo-day checklist item. |
| VI — Data Sovereignty Framing | ✅ | Trust screen copy limited to approved claims; hosting in Bangalore droplet; `access_logs` (below) makes "every access is logged" literally true. |
| VII — Elderly-First Accessibility | ✅ | Large type/tap-target tokens baked into a shared Flutter `ThemeData`. |
| VIII — Home Stays Quiet | ✅ | Home = family cards + alerts strip only (spec §4.1); no feed, no "did you know" surface introduced by this plan. |
| IX — One Snapshot, Never a Parallel Template | ✅ | Emergency-profile preview and Health Summary both render from the same structured source Fastlane/PDF use — no second hand-maintained template. |

**Compliance principles — binding class checked.**
| Principle | Status | How the plan satisfies it |
|-----------|--------|---------------------------|
| X — Authorization Is Derived, Never Accepted `[A]` | ✅ (was GAP) | FR-022. Every DRF viewset touching `profiles`, `records`, `medications`, `emergency_profiles`, `cards`, `scan_events` overrides `get_queryset()` to the caller's own profiles ∪ profiles reachable via an unrevoked `caregiver_grants` row. `get_object()` resolves out of that scoped queryset — never `Model.objects.get(pk=...)` + a permission check. See `contracts/core-api.md` §Authorization Scoping. |
| XI — Every PHI Access Leaves a Log `[A]` | ✅ (was GAP) | FR-021. New `access_logs` table (data-model.md), written in the same transaction as the access; a failed log write fails the request. Core API writes on every PHI read/write; Fastlane writes one row per valid scan (actor=anonymous, source_service=fastlane), distinct from `scan_events` (card mechanics). This is what makes the Trust screen's "every access is logged" claim (Principle VI) true rather than aspirational. |
| XII — FHIR Is Pinned `[A]` when FHIR appears | N/A | Track A emits no FHIR (ABHA stays a free-text field). Re-check if any FHIR fixture is ever added. |
| XIII — PHI Does Not Cross a Boundary in Plaintext `[B→A]` | ✅ (Track A clause) | Fastlane is HTTPS-only; PHI is rendered only in the HTML body, never in the query string (only `ctr`/`cmac` are); responses send `X-Robots-Tag: noindex`; no PHI in access/error logs. See `contracts/fastlane-api.md`. |
| XIV — Demo Shortcuts Are the Audit Remediation List `[A]` | ✅ | Every `# DEMO-MODE` tag carries what's unsafe and its Track B replacement on the same or next line (format enforced by the copy-lint task). |
| XV — Consent Artifacts, Retention, and Erasure `[B→A]` | ✅ | `records.source` carries provenance; records remain individually deletable (`DELETE /api/v1/records/{id}/`); `emergency_payload` is regenerable from source tables, not a second copy of record content. |

**Compliance checklists** (Development Workflow §6)
- Touches PHI, auth, grants, and Fastlane → **Yes.**
  `specs/001-tdc-phr-patient-app/checklists/security.md` generated and
  reviewed against this plan 2026-08-17. Result: 2 `[A]`-blocking GAPs
  (Authorization — negative-auth test, expected until T005/T009 ship; Audit
  — file/PDF reads need an explicit logged-read path before T011/T015), plus
  12 non-blocking GAPs (mostly config/deploy discipline not yet written into
  this plan). One drift the review caught and fixed directly:
  `scan_events.is_test` existed in `CLAUDE.md`/`007` but was missing from
  `001/data-model.md` — added.
- ABDM surface (ABHA, FHIR, consent artifacts, HIP/HIU)? → **No.** The ABHA
  field is manual free text with no FHIR, consent-artifact, or ABDM-network
  call in this feature (that's `003-abdm-sync-subscription`, Track B). No
  `checklists/abdm.md` required for `001`.

No unjustified violations → Complexity Tracking empty.

## Project Structure
```
health/
├── apps/
│   └── health/                    # Flutter app (Android + web) — in.thedoorstepclinic.health
│       └── lib/
│           ├── screens/           # login, home, profile, upload, record, summary,
│           │                      #   meds, emergency, card, family, trust
│           ├── widgets/           # shared UI, warm/calm theme components
│           ├── services/          # api client (JWT via dio), notifications, pdf/share, fcm
│           └── theme/             # ThemeData tokens: palette (#4B83F2 family), type scale
├── services/
│   ├── core-api/                  # Django/DRF
│   │   ├── phr/                   # models, serializers, viewsets (scoped get_queryset — Principle X)
│   │   ├── payload/               # emergency_payload snapshot builder (signal on save)
│   │   ├── audit/                 # access_logs writer, same-transaction as the access (Principle XI)
│   │   └── seed_demo.py           # one-command demo world rebuild
│   └── fastlane/                  # FastAPI
│       ├── main.py                # GET /e/{uid}: CMAC verify, replay kill, render, blast, access_logs write
│       ├── templates/             # server-rendered responder HTML (zero-JS, noindex)
│       └── push.py                # FCM blast (direct)
└── specs/001-tdc-phr-patient-app/
```

## Phase 0 — Research
See [`research.md`](./research.md). Key resolved decisions: two-service split
for uptime isolation; snapshot table over live joins; SDM CMAC verification
approach; Flutter local notifications for reminders; `pdf`/`share_plus` for
PDF share; direct FCM (no Expo push intermediary); `get_queryset()` scoping
pattern for Principle X; same-transaction `access_logs` write for Principle XI.

## Phase 1 — Design
- [`data-model.md`](./data-model.md) — full entity/field/relationship detail,
  the `emergency_payload` snapshot shape + refresh trigger, and the new
  `access_logs` table (Principle XI, already named in `CLAUDE.md`'s data
  model — this plan is what wires it into `001`).
- [`contracts/core-api.md`](./contracts/core-api.md) — DRF `/api/v1/`
  endpoints, now with an Authorization Scoping section (Principle X) and an
  access-log-on-every-PHI-endpoint note (Principle XI).
- [`contracts/fastlane-api.md`](./contracts/fastlane-api.md) — `GET
  /e/{uid}`, now with the HTTPS-only/noindex/no-PHI-in-URL clause (Principle
  XIII) and the `access_logs` write on valid scans.
- [`quickstart.md`](./quickstart.md) — stand-up + golden-path walkthrough,
  updated to Flutter commands and, since 2026-08-20, the `health/apps/` +
  `platform/services/` paths.

**Constitution re-check after design:** ✅ — snapshot isolation preserved, no
card-tap-as-consent surface introduced, and the two principles that were
previously undesigned (X, XI) now have a concrete mechanism. `checklists/
security.md` was generated and reviewed against this plan 2026-08-17 (see
Compliance checklists above); 2 `[A]`-blocking GAPs remain open there, tracked
for Phase A/B/C.

## Phase 2 — Task Planning Approach
`tasks.md` is derived from the spec's build order (D1–D12), regenerated
against this plan's Flutter stack, `health/apps/`+`platform/services/` layout,
and the new authorization-scoping and access-logging design. Tasks are
grouped Setup → Data/Contracts → P0 features (in golden-path order) →
Fastlane/card → P1 → polish. Anything on the emergency flow is never
reordered below P1 polish. Independent files are marked `[P]`. Done — see
`tasks.md` (regenerated 2026-08-15).

## Complexity Tracking
*(empty — no constitution violations; the two former gaps were closed by
design rather than justified as exceptions)*

## Progress Tracking
- [x] Phase 0 complete
- [x] Phase 1 complete
- [x] Constitution re-check passed
- [x] Compliance checklist(s) reviewed against this plan — `checklists/security.md`
      generated 2026-08-17; 2 `[A]`-blocking GAPs remain open (tracked above
      and in the checklist itself), non-blocking GAPs tracked for Phase A/B/C
- [x] Ready for `/tasks` — done, `tasks.md` regenerated 2026-08-15 against this replan
