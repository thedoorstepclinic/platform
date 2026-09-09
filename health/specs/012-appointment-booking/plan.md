# Implementation Plan: Appointment Booking

**Branch:** `012-appointment-booking` (merged to `main` 9 Sep 2026; feature
isolation here is by directory, not by branch — see `001/plan.md`) ·
**Date:** 2026-09-09 · **Spec:** [`spec.md`](./spec.md)
**Constitution:** [`../../.specify/memory/constitution.md`](../../.specify/memory/constitution.md) (**v3.0.0**)
**Input:** `spec.md` — 1,261 lines, 130 FRs, 8 screens (B1–B8), four supplied
reference designs folded in.

## Summary

Build the whole appointment loop for TDC Health: find a clinic, choose a
doctor, choose a **period session**, pay, get reminded, watch the queue,
attend — and reschedule or cancel if plans change.

**The architectural centre of gravity is that availability is not ours.** The
clinic publishes sessions with capacity; our `slots` rows are a **cache** of
that. Booking into published capacity auto-confirms (owner decision, 9 Sep) —
the publication *is* the acceptance — and the loop closes when the clinic or
doctor cancels and we push that to the patient. Everything else in this plan
follows from that: a read-through cache for capacity and queue, a webhook for
clinic-originated events, and exactly one server-push.

Flutter client → Django/DRF Core API over SimpleJWT. A new `directory` app
holds reference data (non-PHI, intentionally unscoped) and a new `booking` app
holds `appointments` and `payments` (PHI-adjacent, scoped from
`caregiver_grants` like everything else). Razorpay handles money and renders
every card field, so no PAN or CVV touches TDC.

## Technical Context

- **Language/Runtime:** Dart (Flutter, Android + web); Python 3.11 (Django/DRF).
- **Primary dependencies:**
  - Client: `go_router` (routes, `008`) · `dio` + `flutter_secure_storage`
    (JWT) · `flutter_local_notifications` (T−24h/T−2h reminders — **reuse
    `006`'s scheduler, do not add a second**) · `firebase_messaging` (the one
    server-push) · `url_launcher` (`tel:` and OS maps hand-off).
  - Payments: **`razorpay_flutter` (Android) + Razorpay Checkout JS (web)** —
    both hosted flows. See `research.md` R7; this is the only way FR-031a
    holds on both targets.
  - Server: Django REST Framework · SimpleJWT · Postgres · `firebase-admin`.
- **Storage:** Postgres. Four directory tables (reference data), `appointments`,
  `payments`. **No queue table** — queue position is read-through from the
  clinic, never persisted.
- **Target platforms:** Android + web. No iOS.
- **Constraints:**
  - B4 is the densest screen in the app and must pass the arm's-length check on
    a real device (Principle VII, FR-013).
  - `booked_count` must never exceed `capacity` — enforced in the database, not
    in a view (R6).
  - No card data through TDC, on either platform (FR-031a).
  - Fastlane is untouched by this feature and must stay that way (Principle I).
- **NEEDS CLARIFICATION:** two, both carried into `research.md` and neither
  blocking Phase 1 — the **maps provider** (R4: reversing an owner override, so
  it needs the owner) and the **no-show consequence policy** (R5: builds as
  "no consequence" until decided).

## Constitution Check

*Gate: passed for Phase 0. Re-checked after Phase 1 design — see Progress
Tracking.*

**All fifteen principles bind unconditionally** (v3.0.0).

**Product principles.**

| Principle | Status | How this plan satisfies it |
|---|---|---|
| I — Emergency Flow Is Sacred | ✅ | This feature does not touch Fastlane, the `emergency_payload` snapshot, or the card path. It adds read and write load to **Core only**, which is precisely why the two services are split. No task in this feature may take a dependency in the Fastlane direction. |
| II — Consent Is a Standing Grant | ✅ | B4's "Booking for" switcher lists profiles the caller already holds grants over and **cannot create one** (FR-015b) — profile creation writes a `caregiver_grants` row, and that consent moment must not sit inside a payment flow. |
| III — Copy Discipline | ✅ | The checkout rejections are FRs, not review notes: no *verified* / *partner* (FR-031d), no cipher strengths, no borrowed certification. Enforced by extending `001` T032's copy lint to this feature's strings. |
| IV — Production-Grade, Demo Switchable | ✅ | See the Demo-path inventory below. The seeded directory, seeded capacity and seeded queue are `DEMO_MODE` **data** substitutions; none of them skips authorization, logging or validation, and removing the switch leaves the feature working against TDC Clinic. |
| V — Reset-in-One-Command | ✅ | `seed_demo.py` seeds the directory and **zero appointments** (FR-041); reset must also clear `payments` rows created during a demo — new, and easy to forget. |
| VI — Data Sovereignty Framing | ✅ | The load-bearing principle here. "Confirmed" and "live" are permitted **per row, following the data** — true where the clinic published it, banned where a fixture stands in. Pricing copy is covered too (FR-031): the platform fee must say what it is for. |
| VII — Elderly-First Accessibility | ✅ | The period model *helps*: three full-width rows beat a grid of small time chips. 48dp floor on every date cell and period row (FR-013). |
| VIII — Home Stays Quiet | ✅ | No tab bar, no Clinics tab (FR-030b — rejected twice from two reference sets). The day-of Home entry (FR-030a) is argued in the spec as a resolving alert given one day's weight, counting **against** the two-row cap rather than beside it. |
| IX — One Snapshot, Never a Parallel Template | ✅ | The fee shown on B2 and again on B4 renders from one source (`clinics.consult_fee_inr`); the payment summary total and the pay-button amount are the same computed value (FR-031d). |

**Compliance principles.**

| Principle | Status | How this plan satisfies it |
|---|---|---|
| X — Authorization Is Derived | ✅ | `appointments` **and `payments`** derive their queryset from the caller's own profiles ∪ unrevoked `caregiver_grants`, reusing `001` T005's base viewset. `profile_id` in a request body is validated against that set, never trusted (FR-035). The directory stays intentionally unscoped and non-PHI. |
| XI — Every PHI Access Leaves a Log | ⚠️ **Blocked** | FR-036 requires an `access_logs` row on create, cancel and reschedule, in the same transaction. **The writer is `001` T006 and is not built.** This feature cannot ship its writes before `001` does — carried as a blocking dependency, not a gap of this feature's own making. |
| XII — FHIR Is Pinned | ✅ N/A | No FHIR appears in this feature, in any mock or fixture. Re-check if a clinic integration ever emits one. |
| XIII — PHI Does Not Cross a Boundary in Plaintext | ✅ | No PHI in any booking URL (`profileId`, `apptId` are opaque ids). **New surfaces to hold to this:** the Razorpay webhook (signature-verified, carries no PHI) and the TDC Clinic event webhook (mTLS or signed, carries appointment ids). |
| XIV — Every Demo Path Is Inventoried | ✅ | Inventory below; every tag names a real path and, where unbuilt, its owner. |
| XV — Consent Artifacts, Retention, Erasure | ✅ | No ABDM consent artifact here. **Payments raise a retention question this feature is the first to face:** a `payments` row is a financial record with a statutory retention life of its own, so it must survive a profile deletion in anonymised form rather than cascading. Recorded in `data-model.md`; the erasure design is `001`'s to own. |

**Compliance checklists** (Development Workflow §6)

- Touches PHI, auth and grants → **Yes.** `checklists/security.md` exists
  (created 6 Sep, 333 lines). **It is stale in one area and says so:** three
  findings are marked `[RESCOPE]` because payments went from *"N/A, nothing
  integrated"* to a live surface. **Re-review against this plan is a Phase 1
  exit condition**, not a follow-up.
- ABDM surface → **No.** No ABHA, FHIR, consent artifact, care context, HIP or
  HIU appears in this feature. No `checklists/abdm.md` required.

**Demo-path inventory** (Principles IV / XIV)

- [x] Every demo path is behind the single `DEMO_MODE` switch, off by default.
- [x] Each substitutes **data**, never behaviour.
- [x] Removing `DEMO_MODE` leaves the feature working — against TDC Clinic for
      capacity/queue/events, and against Razorpay for money.
- [x] Each tag names its real path and, where unbuilt, its owner.

| `# DEMO-MODE` path | Substitutes | Real path | Owner |
|---|---|---|---|
| Seeded directory (`source = seed`) | Clinics, doctors, specialties | TDC Clinic facilities (`tdc_clinic`); UHI for external | Adi |
| Seeded session capacity | `capacity` / `booked_count` | Clinic-published sessions | Adi |
| Seeded queue feed | `now_serving` on a deterministic timer | TDC Clinic queue endpoint | Adi |
| Seeded consult fees | `consult_fee_inr` | Clinic-published fee | Adi |
| Seeded ratings | `rating` | Real post-visit feedback (product decision pending) | Adi |
| Seeded HID chip | `hid_masked` | HPR registry lookup | Adi — **gated on ABDM** |
| Derived `lapsed` | An unreported outcome | Clinic-written `completed` / `no_show` | Adi |

**External gates**

| Gate | Blocks | Owner | Ships without it? |
|---|---|---|---|
| **TDC Clinic endpoints** (capacity, queue, event webhook, outcomes) | Real auto-book, real queue, `completed`/`no_show`, clinic cancellation | Adi | **Yes** — on `DEMO_MODE` fixtures. **Not an external gate: we own the CARE fork.** It is unbuilt work in another repo (`tdc-care`) with no spec, and that is the largest real dependency in this plan. |
| **Razorpay account + keys** | `Pay now` | Adi | **Yes** — clinics with `accepts_pay_at_clinic` book without it |
| **UHI onboarding** | `source = uhi` directory rows | Adi | Yes — TDC Clinic + seed cover the directory |
| **Maps provider + key** | B2/B6 interactive map | Adi | Yes — degrades to address text + Directions (`011` trim order) |
| ABDM certification | Nothing in this feature | — | N/A |

**Blocking, and not a Complexity Tracking row:** Principle XI above. `001`
T006 (the access-log writer) and T010b (the logged-read path) must land before
this feature's writes. There is no non-blocking class of compliance gap under
v3.0.0.

## Project Structure

```
platform/services/core-api/
├── directory/                  # reference data — intentionally unscoped, non-PHI
│   ├── models.py               # specialties, clinics, doctors, slots (period sessions)
│   ├── serializers.py          # explicit fields only; never __all__
│   ├── views.py                # search, clinic detail, nearby, doctors, sessions
│   └── clinic_sync.py          # read-through cache of clinic capacity + queue
├── booking/                    # appointments + payments — profile-scoped
│   ├── models.py               # appointments, payments
│   ├── views.py                # create / cancel / reschedule / list — via 001 T005 base
│   ├── payments.py             # Razorpay order creation + webhook verification
│   └── webhooks.py             # TDC Clinic inbound: cancellation, completed, no_show
└── seed_demo.py                # directory fixtures; zero appointments (FR-041)

health/apps/health/lib/
├── screens/booking/            # B1 find care · B2 clinic · B3 doctor sheet
│                               # B4 confirm · B6 queue status · B7 reschedule · B8 visits
└── services/
    ├── booking_api.dart
    ├── payments.dart           # razorpay_flutter (Android) / Checkout JS (web)
    └── notifications.dart      # EXTENDS 006's scheduler — not a second one
```

## Phase 0 — Research

See [`research.md`](./research.md). Eight decisions: clinic↔Core transport
(R1), queue-feed transport (R2), payments scope boundary (R3), maps provider
(R4 — **needs the owner**), no-show policy (R5 — builds as none), capacity
concurrency (R6), cross-platform payments (R7), and reminder-scheduler reuse
(R8).

## Phase 1 — Design

- [`data-model.md`](./data-model.md) — six tables, the session/token model, the
  full status enum with transition owners, and the payments retention note.
- [`contracts/booking-api.md`](./contracts/booking-api.md) — directory reads,
  appointment writes, and the **two inbound webhooks** (Razorpay, TDC Clinic),
  which are this feature's genuinely new attack surface.
- [`quickstart.md`](./quickstart.md) — stand-up and the end-to-end booking
  path, plus the negative paths that matter (full session, clinic cancellation,
  payment failure).

## Phase 2 — Task Planning Approach

`tasks.md` will be **dependency-ordered**, not tiered — the P0/P1 ladder and
the slip rule are repealed (v3.0.0 workflow rule 3). Grouping:

1. **Blocked-on-`001`** — declared first so the dependency is visible, not
   discovered: T005 scoping base, T006 access-log writer, T010b binary path.
2. **Directory** — models, serializers, endpoints, seed fixtures. Independent
   of everything else and safe to build first.
3. **Booking core** — `appointments`, the atomic capacity increment, create /
   cancel / reschedule, `access_logs` writes.
4. **Client B1–B4** — the booking spine up to confirm, pay-at-clinic path only.
5. **Payments** — Razorpay both platforms, webhook verification, refunds.
6. **Queue + lifecycle** — B6, the read-through feed, clinic webhook, the one
   FCM push, B7, B8.
7. **Reminders** — extend `006`'s scheduler; namespaced ids.
8. **Verification** — extend `001` T010a with this feature's negative-auth and
   logging cases; copy lint; arm's-length pass on B4.

Anything touching money or the clinic webhook carries the adversarial pass
(`CLAUDE.md` working conventions) before merge.

## Complexity Tracking

| Violation | Why needed | Simpler alternative rejected because |
|---|---|---|
| *(none)* | — | — |

No constitution violations. The one blocking item (Principle XI) is a
**dependency on `001`**, not a violation of this plan — and under v3.0.0 it
blocks implementation rather than earning a justification row.

## Progress Tracking

- [x] Phase 0 complete — `research.md`
- [x] Phase 1 complete — `data-model.md`, `contracts/`, `quickstart.md`
- [x] Constitution re-check passed after design (no new surface introduced by
      Phase 1; the two webhooks were already implied by the spec and are now
      contracted explicitly)
- [ ] **Compliance checklist re-reviewed against this plan** — `security.md`
      carries three `[RESCOPE]` findings from the payments scope change and one
      blocking GAP inherited from `001`. **This is the Phase 1 exit condition
      that is not yet met.**
- [ ] Ready for `/speckit-tasks` — after the checklist re-review above
