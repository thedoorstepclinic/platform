# Feature Specification: Consult Booking & Seeded Queue Tracking (Track A — Meetup Loop)

**Feature Branch:** `010-consult-booking-queue`
**Created:** 2026-08-01
**Status:** Draft
**Owner:** Adi (dev) / Soham (meetup script)
**Unlock record:** consult booking + queue tracking were on the banned list;
unlocked by explicit owner decision 1 Aug 2026 (see `CLAUDE.md` changelog for
the full rationale). This spec exists *because of* that entry — if the entry
is ever reverted, this spec reverts with it.
**Depends on:** [`001`](../001-tdc-phr-patient-app/spec.md) (P1 slot, scope
fences) · [`008`](../008-navigation-app-shell/spec.md) (routes, states) ·
[`004`](../004-seed-data-and-summary/spec.md) (fictional facilities, seed
ripple).
**Amended by:** [`011-care-discovery`](../011-care-discovery/spec.md)
(20 Aug 2026) — discovery became step 1 of this flow. Three changes land here:
the entry sequence gains a find-care screen, a clinic page and a doctor step
(FR-002 rewritten); the `appointments` copied strings become foreign keys now
that a real directory exists; and the clinic list this spec described as
"seeded list of 3 fictional clinics" is superseded by `011`'s ~10-clinic seeded
directory. Everything else in this spec — the fences, the queue simulation, the
🔒 pay-at-clinic badge, the two-script model — is unchanged.

## Why this exists (be honest with ourselves)

Two audiences, two jobs:
- **Meetups / social demos (early Aug):** a stranger holding the phone asks
  "what does it do?" — breadth lands here. Booking a consult and watching a
  queue tick is the most *legible* beat in the app for a general crowd, and
  it answers "how does this make money?" without a slide.
- **Investor pitch (16 Aug):** the golden path (`001`) is the script, and
  booking is **not in it**. If asked "what else," the demoer may show it
  live — from the profile, on purpose, framed as "and here's the revenue
  loop, built on our own clinics."

## What this is / is NOT

**IS:** A seeded, demo-grade booking flow (find a fictional clinic → pick a
doctor → pick a slot → confirm; discovery steps owned by `011`) and a
simulated queue view on the resulting appointment. Everything renders from
local seed data.

**IS NOT — hard fences, restated from the unlock:**
- **No real UHI calls.** UHI is named in copy as the intended rails
  ("integration in development" phrasing pattern), Track B builds it.
- **No payments.** Confirmation shows 🔒 **"Pay at clinic"** — the badge
  pattern, exactly like the Family Plan lock.
- **No real-time HMS queue feed.** The queue is a deterministic client-side
  simulation off seed values. Track B wires TDC Clinic's real queue.
- **No real clinics.** `004`'s fictional-facility rule applies with zero
  exceptions — a real clinic name in a booking flow implies a partnership
  and is worse than the Ruby Hall leak we already caught once.

## Constitution check (VIII especially)

Booking enters **through the person, not through a services surface**: a
"Book consultation" action inside the profile context (like Health
Summary). No services tab, no clinic feed, no promo cards — Home gains
nothing except that an **upcoming appointment appears in the alerts strip**
("Baba — Dr. Kavya, tomorrow 10:30, token 16"), which is a resolving alert
(it disappears after the visit), not an inbox item. Principle VIII holds.
Principle IX note: the queue view *is* the primary surface, not a preview
of another surface — no snapshot-sharing obligation triggered.

---

## User flow

1. **Entry:** Profile context → "Book consultation" (placed with Health
   Summary in the profile's action row — per `008`, one route:
   `/profile/[id]/book`).
2. **Find care & pick clinic:** `011` owns this — search, specialty chips and
   a distance-sorted clinic list at `/profile/[id]/book`, then the clinic
   information page at `/profile/[id]/book/[clinicId]`. *(Superseded: this
   spec originally specified a bare list of 3 clinics.)*
2b. **Pick doctor:** `011` owns this — a sheet at
   `/profile/[id]/book/[clinicId]/doctors`. One screen, one choice.
3. **Pick slot:** seeded next-3-days slot grid at
   `/profile/[id]/book/[clinicId]/[doctorId]/slots`. One screen, one choice.
4. **Confirm:** summary card — who (profile), where, **which doctor**, when,
   token number assigned, 🔒 "Pay at clinic" — single Confirm button. Route:
   `/profile/[id]/book/[clinicId]/[doctorId]/confirm`.
5. **Appointment lives on the profile** (`/profile/[id]/appointment/[apptId]`)
   and in the Home alerts strip while upcoming.
6. **Queue view (the meetup money-shot):** appointment detail shows
   *"Now serving token 12 · your token 16 · ~40 min"* with the served-token
   advancing on a seeded, deterministic timer while the screen is open.
   Framed copy: "Live queue from TDC Clinic" is **not** used — copy says
   "Queue status" plain, because the live feed doesn't exist yet and
   Principle VI (claims must be literally true) applies.
7. **Cancel:** one action + confirm, appointment → `cancelled`, alert
   clears.

### Demo-day behaviors
- The simulation is deterministic per seed so rehearsals behave identically
  (same `004` NFR-001 spirit).
- `seed_demo.py` seeds **zero** appointments — booking live *is* the meetup
  beat, and an empty state costs nothing because the flow is 5 taps
  (3 before `011` added discovery and the doctor step).
- Reset returns everything to unbooked.

---

## Data model

New table `appointments` (added to `CLAUDE.md` data model, seed ripple
applied same-day per Workflow rule 5):

| Field | Notes |
|---|---|
| id | PK |
| profile FK | who the visit is for |
| ~~clinic_name / doctor_name / specialty~~ | **superseded by `011`** — copied strings replaced by `clinic` FK · `doctor` FK · `slot` FK now that a real seeded directory exists. Track B should snapshot provider details at booking time; Track A deliberately does not (see `011` *Data model*). |
| slot_ts | timestamptz — mirrors the chosen `slots` row |
| token_no | int, assigned at booking from seed |
| status | `upcoming` \| `done` \| `cancelled` |

Queue simulation state is **not persisted** — derived client-side from
(`token_no`, `slot_ts`, elapsed time). No queue table, no backend job.

## Requirements

- **FR-001**: Booking MUST be reachable only from a profile context —
  no Home-level or nav-level services entry point.
- **FR-002**: *(Amended by `011` FR-005, 20 Aug 2026.)* The flow MUST be
  exactly: find care → clinic → doctor → slot → confirm (one decision per
  screen), ending in an `appointments` row. The first three steps are `011`'s;
  slot and confirm are this spec's.
- **FR-003**: All clinics/doctors MUST be fictional, drawn from `004`'s
  standard facility set as extended by `011`'s seed ripple.
- **FR-004**: Confirmation MUST show 🔒 "Pay at clinic" and MUST NOT
  collect or simulate payment.
- **FR-005**: An upcoming appointment MUST appear in the Home alerts strip
  and clear on completion/cancellation.
- **FR-006**: The queue view MUST be a deterministic client-side simulation;
  no network calls, no claim of live clinic data in copy.
- **FR-007**: The system MUST NOT call any UHI API. UHI appears only in
  roadmap-pattern copy ("built for India's UHI network — integration in
  development") if referenced at all.
- **FR-008**: `seed_demo.py` seeds no appointments; reset clears any booked
  during a demo (`001` FR-017 extension). Unchanged by `011` — the *directory*
  is seeded, appointments are not.
- **FR-009**: Booking MUST NOT appear in the `001` golden-path script; the
  meetup script owns it (Soham). Extended by `011` FR-020 to cover discovery.

## Build placement

Built **only after P0 is demo-ready**, in the runway created by the 16 Aug
date (~1.5–2 days). Top of P1 (cut last among P1, first before any P0 or
emergency-flow polish is touched — slip rule unchanged). If P0 slips past
~9 Aug, this is the first thing paused.

## Open Decisions
| Decision | Owner | Notes |
|---|---|---|
| Meetup script beat order (card first or booking first?) | Soham | Lean card first — it's the differentiator; booking answers the follow-up. |
| Beat length now that discovery is in front | Soham | The flow went from 3 taps to 5. Still short, but the clinic page invites browsing mid-demo — decide whether the script pauses on it or drives through. |
| Simulated queue pacing (tokens/min) | Adi | Fast enough to visibly tick in a 30-second show-and-tell. |

## Review & Acceptance Checklist
- [x] Unlock provenance recorded and linked both ways (`CLAUDE.md` ⇄ this spec).
- [x] Hard fences restated as FRs (no UHI calls, no payments, no real clinics, no live-feed claims).
- [x] Constitution VIII/VI compliance argued, not assumed.
- [x] Golden path untouched; two-script model explicit.
- [x] Requirements testable; build placement + pause trigger defined.

## Execution Status
- [x] Owner unlock recorded
- [x] Flow, model, requirements defined
- [x] Ripples applied (`CLAUDE.md`, `001`; `004` seeds zero rows by design)
- [x] Review checklist passed
