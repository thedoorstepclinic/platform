# Implementation Plan: TDC PHR Patient App — Prototype (Track A)

**Branch:** `001-tdc-phr-patient-app` · **Date:** 2026-07-15
**Spec:** [`spec.md`](./spec.md) · **Constitution:** [`../../.specify/memory/constitution.md`](../../.specify/memory/constitution.md)

## Summary
Build a seeded, demo-grade PHR family app for the end-of-July investor demo. One
Expo React Native codebase (Android + web) talks to a Django/DRF Core API over
SimpleJWT. A separate FastAPI **Fastlane** service serves the emergency
responder page from a denormalized snapshot so it survives Core API downtime.
The golden path — home → timeline → summary/PDF → meds/reminder → NFC card scan
+ family blast → trust screen — must run reliably in 5 minutes and reset in one
command.

## Technical Context
- **Language/Runtime:** TypeScript (Expo RN); Python 3.11 (Django/DRF, FastAPI).
- **Primary dependencies:** Expo, NativeWind (shadcn-style tokens),
  expo-notifications (local reminders), expo-camera/expo-image-picker,
  expo-print / expo-sharing (PDF share), Django REST Framework, SimpleJWT,
  FastAPI, Postgres, FCM (via Expo push service).
- **Storage:** Postgres (Core API). Fastlane reads a denormalized
  `emergency_payload` snapshot (own table/materialized copy).
- **Target platforms:** Android + web only. **No iOS.**
- **Constraints:** Responder page <2s on 4G, zero-JS-readable; Fastlane
  independent uptime budget; elderly-first accessibility.
- **App display name:** **TDC Health** (resolved 15 Jul 2026) — used for APK
  label and app config.

## Constitution Check
*Gate: passed for Phase 0. Re-check after Phase 1 design.*

| Principle | Status | How the plan satisfies it |
|-----------|--------|---------------------------|
| I — Emergency Flow Is Sacred | ✅ | Fastlane is a separate service reading a snapshot; task order front-loads it at D8–9; cut rule encoded in `tasks.md`. |
| II — Consent Is a Standing Grant | ✅ | `caregiver_grants` write on assisted-add is the consent moment; no card-tap copy anywhere; Fastlane logs "access", never "consent". |
| III — Copy Discipline | ✅ | Approved/banned lists enforced in a shared copy module + a lint check task. |
| IV — Demo-Grade, Stated Honestly | ✅ | Mocks (OTP `000000`, manual ABHA, fake HMS webhook) tagged `# DEMO-MODE`. |
| V — Reset-in-One-Command | ✅ | `seed_demo.py` is a D1–2 deliverable and a demo-day checklist item. |
| VI — Data Sovereignty Framing | ✅ | Trust screen copy limited to approved claims; hosting in Bangalore droplet. |
| VII — Elderly-First Accessibility | ✅ | Large type/tap-target tokens baked into the NativeWind theme. |

No violations → Complexity Tracking empty.

## Project Structure
```
repo/
├── app/                      # Expo RN app (Android + web)
│   ├── app/                  # screens (expo-router): login, home, profile,
│   │                         #   upload, record, summary, meds, emergency,
│   │                         #   card, family, trust
│   ├── components/           # shared UI (shadcn-style via NativeWind)
│   ├── lib/                  # api client (JWT), copy/, notifications, pdf
│   └── theme/                # tokens: palette (#4B83F2 family), type scale
├── core-api/                 # Django/DRF
│   ├── phr/                  # models, serializers, viewsets (see contracts)
│   ├── payload/              # emergency_payload snapshot builder (signal on save)
│   └── seed_demo.py          # one-command demo world rebuild
├── fastlane/                 # FastAPI
│   ├── main.py               # GET /e/{uid}: CMAC verify, replay kill, render, blast
│   ├── templates/            # server-rendered responder HTML (zero-JS)
│   └── push.py               # FCM blast
└── specs/001-tdc-phr-patient-app/
```

## Phase 0 — Research
See [`research.md`](./research.md). Key resolved decisions: two-service split for
uptime isolation; snapshot table over live joins; SDM CMAC verification approach;
Expo local notifications for reminders; PDF via expo-print template.

## Phase 1 — Design
- [`data-model.md`](./data-model.md) — full entity/field/relationship detail and
  the `emergency_payload` snapshot shape + refresh trigger.
- [`contracts/core-api.md`](./contracts/core-api.md) — DRF `/api/v1/` endpoints.
- [`contracts/fastlane-api.md`](./contracts/fastlane-api.md) — `GET /e/{uid}`.
- [`quickstart.md`](./quickstart.md) — stand-up + golden-path walkthrough.

**Constitution re-check after design:** still ✅ (snapshot isolation preserved,
no card-tap-as-consent surface introduced).

## Phase 2 — Task Planning Approach
`tasks.md` is derived from the spec's build order (D1–D12). Tasks are grouped
Setup → Data/Contracts → P0 features (in golden-path order) → Fastlane/card →
P1 → polish. Anything on the emergency flow is never reordered below P1 polish.
Independent files are marked `[P]`.

## Complexity Tracking
*(empty — no constitution violations)*

## Progress Tracking
- [x] Phase 0 complete
- [x] Phase 1 complete
- [x] Constitution re-check passed
- [x] Ready for `/tasks`
