# Feature Specification: Medications — Bundled Dose Checklist, Reminders & Stock (Track A)

**Feature Branch:** `006-medications-reminders-adherence`
**Created:** 2026-07-16
**Status:** Draft
**Owner:** Adi (dev)
**Depends on:** [`001-tdc-phr-patient-app`](../001-tdc-phr-patient-app/spec.md) —
implements/expands FR-008 (medications, reminders, stock) and the Home
alerts-strip "low stock / missed dose" signals. Covers screens 7–8 (Meds
list, Med add/edit).
**Ripples into:** [`004-seed-data-and-summary`](../004-seed-data-and-summary/spec.md)
(seed meds) and [`../../CLAUDE.md`](../../CLAUDE.md) (data model) — both
updated same-day per the model-change rule.

## What this is / is NOT

**IS:** The medication model built around a **dose occasion** (a time slot
like "after dinner") that bundles every medicine due at that time into one
checklist. Marking a med taken depletes its stock by one; not marking it
leaves stock untouched. Covers reminders, the taken/skipped checklist,
refills with exact stock, and what happens when a dose changes.

**IS NOT:** Clinical adherence analytics, drug-interaction checks, dose
calculators, or any inference about whether a med *should* be taken. Stock
"days left" is an honest estimate, never presented as medical fact. No
integration with pharmacies or e-prescriptions (Track B, if ever).

---

## Core model: the dose occasion

The mental model comes straight from how caregivers actually run a
pill routine:

> "Grandma has 6 meds after breakfast and 5 after dinner. Sometimes I miss
> one. The reminder says *take your medicines*; the list says *oh, I forgot
> this one*."

So the unit the user interacts with is **not a single drug** — it's a
**time slot** that groups all meds due then into one checklist:

- **Bundled by time:** every med whose schedule includes `20:00` appears in
  the single "Evening / after dinner" checklist. One reminder for the slot,
  not one per pill.
- **Ticked individually:** each med in the slot has its own checkbox — that's
  what makes "I forgot *this* one" visible.
- **Confirmation drives stock:** tick a med → its `stock_count` drops by one.
  Leave it unticked → no change. Untick (undo a mistake) → stock goes back up.
- **Honest estimate:** because depletion depends on the user actually
  ticking, stock is an estimate. If they track, it's accurate; if they
  forget to tick, our count drifts high (we "forget" too). "~N days left" is
  framed as an estimate, never a guarantee.

---

## Goal

A caregiver opens a reminder at dinner time, sees one checklist of the five
meds due, ticks off what grandma took, and the app quietly keeps an estimate
of how many days of each medicine remain — nudging a refill before anything
runs out.

### Primary User Story
As a caregiver, at 8pm I get one "Asha's evening medicines" reminder. Tapping
it shows a checklist of every med due at 8pm. I tick the ones she took; each
tick reduces that med's remaining count. When a med drops below a few days'
supply, Home warns me so I can refill before she runs out.

### Acceptance Scenarios
1. **Given** a profile with meds scheduled at the same time, **When** that
   time's reminder fires, **Then** one notification opens **one checklist**
   listing all meds due at that slot — not one notification per med.
2. **Given** a dose checklist, **When** I tick a med as taken, **Then** that
   med's `stock_count` decreases by exactly one and the tick is recorded for
   today's date + slot.
3. **Given** a med I ticked by mistake, **When** I untick it, **Then** its
   `stock_count` increases by one and today's slot returns to un-acted.
4. **Given** a med left unticked when a slot's time has passed, **When** I
   view Home, **Then** stock is unchanged and a "missed dose" signal may
   appear (see Alerts below) — the app never silently assumes it was taken
   (in P0).
5. **Given** a med whose estimated days-left falls to or below its threshold,
   **When** I view Home, **Then** a low-stock alert appears for that med.
6. **Given** a low or empty stock, **When** I tap **Refill**, **Then** I
   enter the exact new quantity on hand and `stock_count` is set/increased to
   that number.
7. **Given** an active med, **When** I change its **dose** (e.g. 500mg →
   1000mg), **Then** the old medication is **stopped** (marked ended, kept for
   history) and a **new** active medication is created with the new dose —
   the same is true for changing the drug itself.
8. **Given** an active med, **When** I edit only its schedule, stock, or
   threshold (not the dose or drug), **Then** the existing row is updated
   in place — no stop-and-replace.

### Edge Cases
- Two meds share a slot; user ticks one, backs out → the ticked one is
  depleted and recorded, the untouched one is simply un-acted (no partial
  batch write; each checkbox is its own commit).
- Stock hits 0 → med still shows on the checklist (the pills may exist
  off-count); ticking it floors `stock_count` at 0 rather than going
  negative, and Home shows "out of stock."
- Same med ticked, then app reopened → today's tick is remembered (idempotent
  per med + date + slot); re-ticking does not double-deplete.
- A med with no schedule (`times` empty) → appears in the med inventory with a
  stock bar but generates no reminders and no checklist rows (a PRN /
  as-needed med). Allowed; just never nags.

---

## Reminders

- **One local notification per dose slot per day** (Expo local
  notifications), repeating daily. A profile with breakfast + dinner meds
  gets two daily notifications, each opening that slot's bundled checklist —
  not a single all-day digest, and not one-per-drug.
- Tapping a reminder deep-links straight to that slot's checklist (satisfies
  `001` NFR-006: notification → action in <10s, ≤1 intermediate screen).
- Reminders are per profile; the caregiver's device receives them for every
  profile they manage.

## Alerts (feeding Home's alerts strip, `001` screen 2)

- **Low stock:** `days_left = floor(stock_count / doses_per_day) ≤ threshold`
  (threshold in days, default 5 per `001`). Because `stock_count` is
  confirmation-driven, `days_left` is explicitly an **estimate**.
- **Missed dose:** a past slot today whose meds were never ticked or skipped,
  evaluated when the app comes to the foreground (demo-grade — not a separate
  server push). Kept deliberately gentle: it prompts ("You have unchecked
  medicines from this morning"), it does not scold, and it never
  auto-marks anything.

---

## Data model changes

Extends `001`'s `medications` and adds one table. **This is a model change**
— `seed_demo.py`, `004`, and `CLAUDE.md`'s data-model line are updated the
same day (per the standing rule).

### `medications` (extended)
| Field | Change | Notes |
|---|---|---|
| name, dose, times[], stock_count, threshold | unchanged | as `001` |
| `status` | **new** — `active` \| `stopped` | dose/drug change stops the old row |
| `started_on` | **new** — date | when this regimen began |
| `ended_on` | **new** — date, nullable | set when stopped |

`stock_count` is an **exact unit count** (individual pills/units the user
reports having, e.g. "30 tablets"), decremented one per confirmed dose.

### `dose_events` (new — the checklist log, parallels `scan_events`)
| Field | Notes |
|---|---|
| id | PK |
| medication FK | which med |
| profile FK | denormalized for fast "today's checklist" queries |
| dose_date | the calendar day |
| slot_time | the schedule slot this belongs to (e.g. `20:00`) |
| status | `taken` \| `skipped` |
| marked_ts | when the user acted |

Unique on (`medication`, `dose_date`, `slot_time`) → guarantees a med can't
be double-counted for the same occasion. Absence of a row for a past slot =
un-acted (the raw material for the "missed dose" signal). A `taken` row is
what triggers the `stock_count − 1`; untick deletes the row and restores the
unit.

### Derived, not stored
- Today's checklist = active meds for the profile, grouped by the distinct
  `slot_time`s in their `times[]`, joined against today's `dose_events`.
- `days_left`, low-stock, and missed-dose are all computed, never persisted.

---

## Requirements

### Functional Requirements
- **FR-001**: Meds due at the same scheduled time MUST be presented as one
  bundled checklist under a single reminder — one notification per slot, not
  per med.
- **FR-002**: Each med in a checklist MUST be independently markable as taken
  or skipped.
- **FR-003**: Marking a med taken MUST decrement its `stock_count` by one and
  write a `dose_events(taken)` row; unmarking MUST restore the unit and remove
  the row. Operations MUST be idempotent per (med, date, slot) — no double
  depletion.
- **FR-004**: Not marking a med MUST leave `stock_count` unchanged. The system
  MUST NOT auto-decrement on schedule in P0. (An optional "assume taken on
  schedule" estimation mode is explicitly deferred — see Open Decisions.)
- **FR-005**: `stock_count` MUST be an exact user-reported unit count and MUST
  floor at 0 (never negative).
- **FR-006**: A **Refill** action MUST let the user set the exact quantity now
  on hand, updating `stock_count` accordingly.
- **FR-007**: Changing a med's **dose or drug** MUST stop the existing
  medication (`status=stopped`, `ended_on=today`, retained) and create a new
  active medication — preserving a history trail.
- **FR-008**: Changing only schedule, stock, or threshold MUST update the
  existing row in place (no stop-and-replace).
- **FR-009**: The system MUST schedule one repeating daily local notification
  per dose slot per profile, deep-linking to that slot's checklist.
- **FR-010**: `days_left` MUST be computed as
  `floor(stock_count / doses_per_day)` and surfaced as an **estimate**;
  low-stock alert fires when `days_left ≤ threshold`.
- **FR-011**: A "missed dose" signal MAY be surfaced for past-due unticked
  slots, evaluated on app foreground; it MUST be advisory only and MUST NOT
  auto-mark or scold.
- **FR-012**: A med with an empty schedule MUST be allowed (PRN/as-needed):
  tracked for stock, but generating no reminders and no checklist rows.

### Non-Functional Requirements
- **NFR-001 (Honesty):** any surfaced "days left" or stock figure MUST read as
  an estimate, consistent with `CLAUDE.md` copy discipline — never implied as
  a verified clinical count.
- **NFR-002 (Demo-grade):** missed-dose evaluation on foreground is acceptable;
  no background job / server scheduler required for Track A.

---

## Open Decisions
| Decision | Owner | Lean |
|---|---|---|
| Optional "assume taken on schedule" estimation mode (auto-deplete even without a tick) | Adi | **Defer past P0.** The user raised it ("we could assume") but it muddies the honest-estimate model; add later as an explicit per-med toggle if wanted. |
| ~~Slot labels~~ | — | **RESOLVED (16 Jul 2026): named occasions** (Morning / After breakfast / After dinner / Bedtime) presented in UI, mapped to and stored as clock times underneath. |
| Strip-based stock entry helper ("2 strips × 15 = 30") | Adi | Nice-to-have; `stock_count` stays an exact unit count underneath. Defer unless quick. |
| ~~Missed-dose grace window / tone~~ | — | **RESOLVED (16 Jul 2026): advisory-only nudge, ~2h grace after slot, never scolding** ("You have unchecked medicines from this morning"). |

---

## Review & Acceptance Checklist

### Content Quality
- [x] Built on the caregiver's real mental model (dose occasion, not per-drug).
- [x] Stock honesty (estimate, confirmation-driven) made explicit, per copy discipline.
- [x] Model change flagged with same-day ripple to seed / `004` / `CLAUDE.md`.

### Requirement Completeness
- [x] Requirements testable (one notification per slot, idempotent depletion, stop-and-new on dose change).
- [x] Deferred scope (auto-assume estimation) named, not silently dropped.
- [x] Edge cases cover double-tick, zero-floor, PRN meds, undo.

## Execution Status
- [x] Core model (dose occasion) defined from review answers
- [x] Data-model additions specified (`dose_events`, medication lifecycle)
- [x] Requirements generated
- [x] Ripple to `004` / `CLAUDE.md` identified (applied same commit)
- [x] Review checklist passed
