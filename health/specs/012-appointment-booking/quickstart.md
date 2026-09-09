# Quickstart: Appointment Booking

**Phase 1 for [`plan.md`](./plan.md).** How to stand this feature up and prove
it works end to end. Validation guide — implementation lives in `tasks.md`.

> **Nothing here runs yet.** The repo is spec-only: `platform/services/core-api/`
> and `health/apps/health/` are scaffolded by `001` tasks T001–T003, which have
> not run. This document is the target, written now so the tasks have something
> to be checked against.

---

## Prerequisites

| Needed | From | Blocking? |
|---|---|---|
| Core API + Flutter app scaffolded | `001` T001–T003 | **Yes** |
| Authorization scoping base | `001` **T005** | **Yes** — every appointment endpoint uses it |
| Access-log writer | `001` **T006** | **Yes** — Principle XI; `012` cannot ship writes before this exists |
| Seeded profiles (Rohan / Asha / Prakash) | `001` T010, `004` | Yes |
| Razorpay test keys | Owner (Adi) | No — `accepts_pay_at_clinic` clinics book without them |
| TDC Clinic endpoints | `tdc-care` (unbuilt) | No — `DEMO_MODE` fixtures cover capacity, queue and events |

```bash
# Core API
cd platform/services/core-api
python -m venv .venv && . .venv/bin/activate && pip install -r requirements.txt
export DEMO_MODE=1                 # fixtures ON — never in a shipped build
python manage.py migrate
python seed_demo.py                # directory + sessions; ZERO appointments

# Client
cd health/apps/health
flutter run -d android             # or: flutter run -d chrome
```

`DEMO_MODE=1` substitutes **data only** — the seeded directory, session
capacity, the queue feed and consult fees. It does not skip authorization,
logging or validation, and **removing it must leave the feature working**
against TDC Clinic. If it doesn't, the feature is unfinished (Principle IV).

---

## Golden path — book a consult

Log in as Rohan. Nine steps, five taps once you are in the flow.

1. **Home → Asha's profile → "Book consultation."** There is no other way in:
   no tab bar, no Clinics tab, no route into `/book` without a profile
   (FR-001). Confirm the header reads *"Booking for Aai (Asha K.)"*.
2. **B1 Find care.** Search `Kavya`; the doctor-name hit resolves the clinic
   and opens B3 with her highlighted (FR-005) — never a dead end. Back out and
   tap a specialty chip instead; confirm it filters and un-taps to clear.
3. **B2 Clinic page.** Rating renders with **no review count** (FR-027). HID
   chip has **no tick, no verification copy, no tap** (FR-028). Fee line shows
   the consultation fee. Sticky **Book Appointment** never scrolls away.
4. **B3 Choose doctor.** The sheet appears **even for a single-doctor clinic**
   (FR-007) — predictability beats smoothness.
5. **B4 Confirm Booking.** Date strip of 5 days; period rows Morning /
   Afternoon / Evening. Confirm a **full session renders disabled and labelled,
   not hidden** (FR-012). Payment summary itemises **consultation + platform
   fee with an explicit total** (FR-031); the pay button amount equals it.
6. **Switch "Booking for" to Prakash and back.** It lists only profiles you
   hold grants over, and offers **no "+ Add New"** (FR-015b).
7. **Confirm.** Appointment is created **`booked`**, not `requested` — the
   clinic published the capacity, so it is confirmed (FR-016a). A token is
   issued. Two local reminders are scheduled.
8. **Home.** An alerts-strip row appears (within 48h), and on the day it is
   promoted to the day-of entry (FR-030a).
9. **B6 Queue Status.** Session banner, your token, current token, tokens
   before you, wait estimate **with its disclaimer** (FR-023c). Directions and
   Call both hand off correctly.

**Expected end state:** one `appointments` row, `status = booked`, `token_no =
booked_count` at issue, `provider_snapshot` populated, one `access_logs` row.

---

## Negative paths — these are the ones that matter

### Session full

Book into a session seeded at capacity. Expect **`409`**, copy *"That session
is full — pick another,"* and B4 refreshed with live counts. **No overselling:**
the `CHECK (booked_count <= capacity)` constraint holds even if a view forgets.

### Two bookings at once

Fire two `POST /appointments` at one session with room. **Both must succeed**
with distinct tokens (16 and 17) — capacity is a counter, not a unique row two
people claim. This is the case the old exact-slot model got wrong.

### Clinic cancels

Post a cancellation to `/hooks/clinic/`. Expect: status `cancelled`,
`cancelled_by = clinic`, reason carried verbatim, slot released, **both
reminders cancelled**, alerts-strip and day-of entries cleared, full refund
**including the platform fee**, and an **FCM push** naming who cancelled, when
the appointment was, and a rebook action. This is the only server-push in the
feature (FR-016f).

### Payment fails

Fail the Razorpay test payment. **The booking must survive.** Where the clinic
accepts pay-at-clinic, fall back to it with the appointment intact; where it
does not, hold in place with a retry. **The token is not released on a gateway
timeout** (FR-031b) — losing a slot to a payment glitch is the worst outcome
available here.

### Wrong patient

As a second user, `GET`, `cancel` and `reschedule` user A's appointment by id.
All must return **`404`, not `403`** — out-of-scope and nonexistent are
indistinguishable. Then revoke a grant and repeat: access must stop on the
**very next request**, with no cached scope.

Cancel and reschedule are the higher risk of the three: creating a spurious
appointment is noise; cancelling someone else's is the appointment silently not
happening.

### Queue feed down

Stop the feed mid-session. B6 shows the token and session and says the position
is **unavailable**. It must not show a stale number as current, and must not
substitute an estimate.

### Reminder collision

Reschedule an appointment for a profile that also has 8pm dose reminders
(`006`). **Assert the dose reminders survive.** Shared scheduler namespace: a
colliding id silently cancels a medication reminder through a booking action —
a safety bug nobody looks for because the features seem unrelated
(`research.md` R8).

---

## Copy checks — read these on a real screen

| Must read | Must never read |
|---|---|
| "Confirmed" **where the clinic published the capacity** | "Confirmed" on a `DEMO_MODE` fixture — "Booking placed" there |
| "Queue status", "~10 min wait", "about 40 min" | "live" / "real-time" / "now serving" without a real feed |
| "Pay at clinic" (no lock — it is a real option) | 🔒 on a real option |
| "Payments handled by Razorpay" | "PCI-DSS Level 1 Certified", "256-bit SSL" |
| Platform fee, labelled with what it is for | "Verified", "partner", any cashback offer |

The word **follows the data, per row, not per screen.** The same B6 says "live"
for a `tdc_clinic` session and does not for a seeded one.

---

## Reset

```bash
python seed_demo.py --reset
```

Rebuilds the directory, returns every session to its seeded `booked_count`, and
clears appointments **and `payments`** created during a demo. The payments half
is new with this feature and is the easy thing to forget (Principle V).

---

## Before calling it done

- [ ] `DEMO_MODE=0` and the golden path still works against TDC Clinic — or the
      gate is named with its owner and everything on our side of it is built.
- [ ] B4 checked **at arm's length on a real device**, not a simulator. It is
      the densest screen in the app (Principle VII, FR-013).
- [ ] `checklists/security.md` re-reviewed — three `[RESCOPE]` findings from the
      payments scope change, plus the blocking `access_logs` GAP inherited from
      `001`.
- [ ] Adversarial pass on both webhooks and the payment path
      (`CLAUDE.md` working conventions).
