# Feature Specification: Seed Data & Health Summary Contract

**Feature Branch:** `004-seed-data-and-summary`
**Created:** 2026-07-16
**Status:** Draft
**Owner:** Adi (dev) / Sharvari (clinical review, PDF layout)
**Depends on:** [`001-tdc-phr-patient-app`](../001-tdc-phr-patient-app/spec.md) —
implements that spec's FR-017 (`seed_demo.py`) and FR-007 (Health Summary PDF)
at field level.
**Source of truth for locked values:** [`../../CLAUDE.md`](../../CLAUDE.md).
Where this spec adds detail beyond `CLAUDE.md`, it is marked **illustrative**
and may be adjusted; where it restates a `CLAUDE.md` value, that value is
**locked**.

## What this is / is NOT

**IS:** The exact contract for what `seed_demo.py` creates — every profile,
record, medication, and emergency-profile field, down to values that the
golden-path script (`001` §Demo Goal) depends on literally matching (e.g.
"4 days left" on screen). Also defines the field-level mapping for the
Health Summary PDF (`001` FR-007), since "structured fields + record list"
needs a concrete order to be implementable and clinically reviewable.

**IS NOT:** The PDF's visual design (typography, layout, branding) — that
stays Sharvari's clinical-review call per `001`'s open decisions. This spec
fixes *which fields appear, in what order, sourced from what data* — not
how they're styled.

---

## Guardrail: all clinics are fictional

Every facility name appearing in seed data, timeline entries, or example
copy anywhere in the app (push notifications, empty states, docs) **MUST be
fictional**. No real, identifiable hospital, clinic, or lab name — this
applies even to placeholder/example text in specs and code comments, not
just runtime data. (Locked, `CLAUDE.md` — "All clinics fictional.")

**Standard fictional facilities for this seed set** (reuse these; don't
invent new ones per-record):
- **Sunrise Poly Clinic** — Asha's prescribing clinic.
- **Ashirwad Diagnostics** — Asha's lab (HbA1c panels). Also the standard
  example name used across specs `001`/`003` for "new lab report" copy.
- **Prabhat Multispecialty Hospital** — the Nov-2025 hypoglycemia discharge.
- **Kavya Family Clinic** — Prakash's prescribing clinic.

### Care-discovery directory (`011`, added 20 Aug 2026; owned by `012` from 6 Sep 2026)

> **Ownership note (6 Sep 2026).** `011` and `010` are superseded by
> [`012-appointment-booking`](../012-appointment-booking/spec.md), which owns the
> whole booking flow. This section's rules are unchanged; the spec that consumes
> them is now `012`. Additions from `012` are in *Booking lifecycle seed* below.

`012` needs a browsable directory, so the four names above are
no longer the whole set — they are the **record-bearing** facilities and stay
locked as such. The directory extends the set with roughly six more bookable
clinics, under the same rule and two extensions of it.

- **Bookable clinics reuse three of the four:** Sunrise Poly Clinic, Prabhat
  Multispecialty Hospital, Kavya Family Clinic. **Ashirwad Diagnostics is a
  lab and MUST NOT appear as a bookable consult clinic** — a lab offering
  consultations is a data-model lie the demo doesn't need.
- **~7 new fictional clinic names** are required to reach ~10. **Approved by
  owner decision, 6 Sep 2026** — the candidate list lands as-is and this section
  is no longer a blocker: Gulmohar Health Centre · Shantiniketan Family Clinic ·
  Nisarg Multispecialty · Anandvan Child Care · Chandrakala Heart Care · Tulip
  Poly Clinic · Riverside Family Clinic. The **avoid list stays binding** for
  any name added later — Ruby Hall, Sahyadri, Jehangir, Deenanath Mangeshkar,
  Noble, Poona Hospital, Sancheti, Inamdar, Aditya Birla. *Residual risk,
  recorded not resolved:* these seven were accepted on judgement rather than a
  register check, so a collision with a small real Pune practice is possible;
  the mitigation is that a collision is corrected by editing one seed row, and
  imagery (Extension 1) and coordinates (Extension 2) remain the two harms that
  cannot be undone by a rename.
- **Extension 1 — imagery.** The rule now covers pictures, not just names: no
  clinic or doctor image may depict an identifiable real facility or a real
  practising clinician (`012` FR-025, was `011` FR-010). A stock photo of an actual hospital
  is the same leak by another route.
- **Extension 2 — coordinates.** `012` seeds real lat/lng per clinic for its
  map. A seeded pin MUST NOT land on a real healthcare facility (`012`
  FR-024, was `011` FR-011) — pinning a fictional clinic onto a real hospital asserts an
  address as well as a name, which is worse than the Ruby Hall leak, not
  better.

Also seeded by `012`: ~10 specialties (4 primary chips + the *More* set,
including **Diabetology** so Asha's follow-up is a natural demo path), 2–4
doctors per clinic with one `is_lead`, seeded ratings in a 4.2–4.9 band,
facility tags, 3–5 FAQs per clinic, and slots for the next 3 days.
**Slots MUST be generated relative to run date**, never fixed timestamps —
a reseed on demo morning must not produce yesterday's availability.
Dr. Kavya must exist as a real seeded doctor row (she is already named in
the alerts-strip example copy).

### Booking lifecycle seed (`012`, added 6 Sep 2026)

Four additions on top of the directory above.

- **`clinics.consult_fee_inr`** — seeded in a believable **₹300–₹800** band,
  varying by clinic and specialty. Ten identical fees read as placeholder data.
- **`clinics.cancellation_window_min`** — **120** for every seeded clinic
  except **one seeded at 240**, so the per-clinic path is actually exercised
  rather than merely present.
- **~30% of generated slots pre-marked `is_booked`.** `012` B4 renders taken
  slots struck through rather than hiding them; with an empty seed that state
  is invisible until someone stages it by hand mid-demo.
- **`appointments` still seeds zero rows** (`012` FR-041, unchanged from `010`
  FR-008). Booking live is the meetup beat, and reset returns everything to
  unbooked. The directory is inventory; appointments are user data.

---

## Seeded profiles

### Rohan S. — account owner (age 34)
| Field | Value |
|---|---|
| Records | **1** (locked count) — e.g. a routine annual blood-panel PDF, Sunrise Poly Clinic, dated ~3 months before demo date. |
| Medications | None. |
| Emergency profile | Intentionally incomplete — no blood group / allergies / conditions filled in. |
| Card | Not linked. |

Rohan's profile is deliberately the least "finished" one. This is not an
oversight to fix — it's realistic (a healthy 34-year-old has little reason
to have filled his own emergency profile yet) and it's a useful, honest
demonstration of the setup-checklist chips from `001` §4.1: even the account
owner's own card can show open chips.

### Asha K. — "Aai" (age 61) — the demo-rich profile

**Emergency profile** (locked anchors from `CLAUDE.md` in **bold**):
| Field | Value |
|---|---|
| Blood group | O+ *(illustrative — any value is fine, just needs to be set)* |
| Allergies | **Penicillin** |
| Conditions | **Type 2 Diabetes Mellitus** |
| ABHA no. | blank (manual field, per `001` — the seed asserts no ABHA; real linking is `002`/`003`) |
| Emergency contacts | Rohan S. (son) — priority 1; Prakash P. (husband) — priority 2 |

**Records — 10 total across Jan 2025–Jul 2026 (18 months, locked):**

| # | Date | Type | Title | Facility |
|---|------|------|-------|----------|
| 1 | Jan 2025 | Lab | HbA1c — **8.1%** *(locked start value)* | Ashirwad Diagnostics |
| 2 | Feb 2025 | Rx | Metformin 500mg started, once daily | Sunrise Poly Clinic |
| 3 | Apr 2025 | Lab | HbA1c — 7.8% | Ashirwad Diagnostics |
| 4 | Jun 2025 | Rx | Metformin — routine renewal | Sunrise Poly Clinic |
| 5 | Jul 2025 | Lab | HbA1c — 7.5% | Ashirwad Diagnostics |
| 6 | Sep 2025 | Rx | Routine follow-up, dose unchanged | Sunrise Poly Clinic |
| 7 | **Nov 2025** | **Discharge** | **Hypoglycemia episode, overnight observation** *(locked event)* | Prabhat Multispecialty Hospital |
| 8 | Jan 2026 | Lab | HbA1c — 7.2% | Ashirwad Diagnostics |
| 9 | Apr 2026 | Lab | HbA1c — 7.0% | Ashirwad Diagnostics |
| 10 | Jul 2026 | Lab | HbA1c — **6.9%** *(locked end value)* | Ashirwad Diagnostics |

Rows 2–6 and 8–9 are **illustrative** — exact dates/titles may shift, but
the record count (10), record-type mix (3 Rx + 6 Lab + 1 Discharge, matching
`001`'s "Rx, HbA1c lab PDFs, one discharge summary"), the HbA1c trend
(**8.1% → 6.9%**, locked start/end), and the November 2025 hypoglycemia
discharge (locked) MUST all hold. Next HbA1c due ~Sep 2026 — this is what
powers the "HbA1c due Sep" alerts-strip example in `001` §4.1.

**Medications — 2 active** (locked count, per `001` §2 "2 active meds";
both seeded at the 20:00 slot so the golden-path 8pm reminder opens a
**bundled two-item checklist**, demonstrating `006`'s dose-occasion model):

| Field | Metformin | Sitagliptin |
|---|---|---|
| Dose | 500mg | 50mg *(illustrative)* |
| `times[]` | **`["20:00"]`** | `["20:00"]` |
| `stock_count` | **4** | 28 *(illustrative — healthy)* |
| `threshold` | 5 | 5 |
| `status` | `active` | `active` |
| `started_on` | Feb 2025 *(matches Rx record #2)* | ~mid-2025 *(illustrative add-on)* |
| `ended_on` | — | — |

Sitagliptin is a realistic second-line add-on for a T2 diabetic already on
Metformin, so it fits Asha's clinical arc without inventing anything. Its
stock is deliberately healthy so only **Metformin** trips the low-stock
alert — one alert, not two competing ones (same discipline as Prakash's
profile below).

**Reconciliation note (locked by the golden-path script itself):** `001`'s
demo script states Metformin's counter must read *"4 days left."* Per
`001`'s `days_left = floor(stock_count / doses_per_day)` formula, that
requires **exactly one dose per day** — hence `times = ["20:00"]`, not
twice-daily. This also produces the "8pm reminder notification" for free,
since the single daily dose *is* the 8pm dose. Do not seed Metformin as
BID; it will silently break both the "4 days left" text and the "8pm
notification" beat.

**Seed the dose log empty:** no `dose_events` rows are seeded — the demo
starts with today's doses un-ticked, so `stock_count` sits at its exact
seeded values (Metformin 4) and "4 days left" holds on demo open. The live
demo action of ticking Metformin taken then visibly drops it to 3.

**Card:** linked, `active = true` — this is the card used in the golden
path's "money shot" (`001` §6).

**Scan log:** seed exactly **one** `scan_events` row on Asha's card —
`is_test = true`, dated her card-link day (the natural "we tested it when
we set it up" entry). This makes the Card manager's scan log demo lived-in
instead of empty, without inventing a mystery real-world scan that would
raise "who scanned Aai's card?" questions on stage.

### Prakash P. — "Baba" (age 66) — the lighter profile

**Emergency profile:**
| Field | Value |
|---|---|
| Blood group | B+ *(illustrative)* |
| Conditions | **Hypertension** (locked) |
| Allergies | None known |
| Emergency contacts | Rohan S. (son) — priority 1; Asha K. (wife) — priority 2 |

**Records — 3 total** *(locked count)*, illustrative content: one Rx
(Amlodipine start), one follow-up Rx, one routine BP-check lab note — all at
Kavya Family Clinic.

**Medication — 1 active:**
| Field | Value |
|---|---|
| Name | **Amlodipine** (locked) |
| Dose | 5mg *(illustrative)* |
| `times[]` | `["08:00"]` |
| `stock_count` | 20 |
| `threshold` | 5 |
| `status` | `active` |
| `started_on` | *(matches his Amlodipine-start Rx record)* |
| `ended_on` | — |

Deliberately healthy stock — Prakash's profile exists to prove multi-profile
support without generating a second competing alert on demo day (`001` §2:
"proves multi-profile without demo clutter").

**Card:** not required for the golden path. Seed it **unlinked** so the
Card manager screen (`001` screen 10) has a second, different card state to
show beyond Asha's active one.

---

## Health Summary PDF — field mapping (FR-007)

Template-rendered from structured fields + the record list — **not**
AI-generated, per `001` FR-007. Field order reuses the same
clinical-priority ordering as the Fastlane responder page (`001` §6), so a
responder and a doctor see the same "what matters most" sequence across both
surfaces:

1. **Header** — profile name, age/DOB, blood group.
2. **Allergies** (flagged/high-contrast, same visual priority as the
   responder page).
3. **Conditions.**
4. **Current medications** — name, dose, schedule.
5. **Recent records** — last 5 by date, newest first: type, date, title.
   (Not the full timeline — a doctor in a waiting room needs a snapshot, not
   18 months of history; full history stays in-app.)
6. **Emergency contacts.**
7. **Footer** — generation timestamp, "This is a summary, not a complete
   medical record," TDC Health branding line.

Layout/typography sign-off is Sharvari's open decision from `001` (due D6);
this field order is what she's reviewing, not a placeholder.

---

## Requirements

### Functional Requirements
- **FR-001**: `seed_demo.py` MUST be idempotent — safe to re-run any number
  of times without creating duplicate rows (delete-and-recreate or upsert
  semantics), per `001` FR-017.
- **FR-002**: All facility names in seed data MUST be fictional (see
  Guardrail above); this is enforced by using only the approved fictional
  facility set — the four record-bearing names plus `012`'s approved
  directory additions — never ad hoc names. Extended 20 Aug 2026 to cover
  clinic/doctor **imagery** (`012` FR-025) and seeded **coordinates**
  (`012` FR-024), not just names.
- **FR-003**: Asha MUST have exactly 2 active meds, both at the 20:00 slot
  (Metformin `stock_count = 4`, threshold 5; Sitagliptin healthy stock), so
  the 8pm reminder opens a bundled two-item checklist (`006`), only Metformin
  trips low-stock, and the counter reads exactly "4 days left." No
  `dose_events` are seeded (doses start un-ticked). These are literal
  golden-path requirements, not approximations.
- **FR-004**: Asha's record set MUST total exactly 10 records spanning Jan
  2025–Jul 2026, include exactly one Discharge record dated Nov 2025, and
  show HbA1c trending from 8.1% to 6.9% across the Lab records in date
  order.
- **FR-005**: Any change to the `records`, `medications`, `dose_events`,
  `emergency_profiles`, `scan_events`, `clinics`, `doctors`, `specialties`,
  or `slots` models MUST be reflected in `seed_demo.py` the same day (per `CLAUDE.md` working conventions) — a
  stale seed script is treated as a broken build, not a follow-up task.
- **FR-007**: Asha's card MUST be seeded with exactly one test scan event
  (`is_test = true`, dated the card-link day) and no non-test scan events.
- **FR-008**: `appointments` (`010`) MUST be seeded **empty** — booking live
  is the meetup demo beat — and re-running `seed_demo.py` MUST clear any
  appointments booked during a demo. The `012` directory tables are the
  opposite: they MUST be fully seeded, because they are inventory rather than
  user data.
- **FR-009**: `012`'s `slots` MUST be generated relative to the seed run date,
  never as fixed timestamps, so a reseed on demo morning yields future
  availability (this is NFR-001's determinism applied to a moving reference
  point — same relative shape every run, not the same absolute instants).
- **FR-006**: The Health Summary PDF MUST render the seven sections above in
  the specified order, sourced only from structured fields and the record
  list (no free-text AI summarization).

### Non-Functional Requirements
- **NFR-001 (Determinism):** re-running `seed_demo.py` must produce
  byte-identical seed state (same IDs where the app depends on stable
  references, e.g. the demo card UID) so rehearsal runs behave identically
  to demo day.

---

## Open Decisions
| Decision | Owner | Notes |
|---|---|---|
| Health Summary PDF visual layout/typography | Sharvari | Field order above is what's under review, not the visual design (`001` open decision, due D6). |
| Exact illustrative record titles/dates for rows 2–6, 8–9 (Asha) and all 3 of Prakash's records | Adi | Free to adjust; locked anchors (counts, HbA1c values, Nov-2025 discharge, Amlodipine) must hold. |
| Demo card UID / spare card UID values | Adi | Assigned at card-personalization time (`001` D10), not fixed in this spec. |

---

## Review & Acceptance Checklist

### Content Quality
- [x] Every locked value traced to `CLAUDE.md` or the original golden-path
      script, not invented.
- [x] Illustrative vs. locked values explicitly distinguished throughout.
- [x] All mandatory sections completed.

### Requirement Completeness
- [x] Requirements testable (record counts, exact stock/dosing values, PDF
      section order).
- [x] The "4 days left" / 8pm-reminder reconciliation is made explicit and
      traceable to a concrete dosing decision, not left ambiguous.
- [ ] PDF visual layout sign-off — **[NEEDS CLARIFICATION: Open Decision,
      owner Sharvari, due D6 per `001`]**.

## Execution Status
- [x] `CLAUDE.md` seed-data section parsed and expanded to field level
- [x] Golden-path script cross-checked for literal-value dependencies
      (caught and resolved the dosing/stock-count reconciliation)
- [x] Fictional-facility guardrail defined and retrofitted into `001`/`003`
- [ ] Review checklist fully passed (blocked only on PDF layout sign-off)
