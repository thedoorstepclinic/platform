# Data Model: Appointment Booking

**Phase 1 for [`plan.md`](./plan.md).** Six tables in TDC Core API — four
directory (reference data, non-PHI) and two booking (profile-scoped). No queue
table: queue position is read through from the clinic and never persisted.

Ripples into `CLAUDE.md`'s global data model were applied 9 Sep 2026;
`seed_demo.py` follows same-day per Workflow rule 5 when it exists.

---

## Ownership and scoping, in one line each

| Table | Class | Scoping |
|---|---|---|
| `specialties` · `clinics` · `doctors` · `slots` | Reference data, non-PHI | **Intentionally unscoped** — readable by any authenticated user, recorded in `checklists/security.md` |
| `appointments` · `payments` | Profile-linked | Derived from the caller's own profiles ∪ unrevoked `caregiver_grants` (Principle X, via `001` T005's base viewset) |

`appointments` is PHI. `payments` is **not** PHI but **is** profile-linked, so
it takes the same scoping and is excluded from the directory's unscoped
exemption. That distinction matters at serializer-review time: a payments row
leaking tells you who saw a doctor and when, which is the same disclosure by a
different route.

---

## Directory tables

### `specialties`

| Field | Type | Notes |
|---|---|---|
| `id` | uuid PK | |
| `name` | string | |
| `icon_ref` | string | Complete set required — no missing-glyph fallback (`012` B1) |
| `sort_order` | int | |
| `is_primary` | bool | The 4 shown before *More* |

### `clinics`

| Field | Type | Notes |
|---|---|---|
| `id` | uuid PK | |
| `source` | enum | `seed` \| `tdc_clinic` \| `uhi`. **Never serialized to clients** |
| `external_ref` | string, null | HFR facility id in the UHI path. **Never serialized** |
| `name` · `address_line` · `area` · `city` | string | Fictional for `seed` rows (`004`) |
| `lat` / `lng` | decimal | FR-024: MUST NOT land on a real healthcare facility |
| `distance_km` | decimal | Seeded static; no location permission (FR-033) |
| `hero_image_ref` | string | FR-025: no identifiable real facility |
| `rating` | decimal, null | Rating only — no count, no corpus (FR-027) |
| `typical_wait_min` · `queue_count` | int | Display values; approximate phrasing (FR-026) |
| `avg_consult_min` | int, null | Input to the wait estimate. **Null means omit the estimate** (FR-023b) — never default it |
| `consult_fee_inr` | int | Clinic's published fee; seeded value is a `DEMO_MODE` fixture |
| `platform_fee_applies` | bool | Whether the ₹50 rides on this clinic's bookings — see `payments` |
| `cancellation_window_min` | int, default 120 | Cutoff for cancel and reschedule |
| `response_window_min` | int, default 30 | **UHI path only.** Unused on the TDC Clinic path, where booking auto-confirms |
| `accepts_pay_at_clinic` | bool | False → B4 is prepay-only (the reference layout) |
| `phone` | string, null | `tel:` target on B6 |
| `facility_tags` | string[] | parking, wheelchair, pharmacy, wifi, lift, lab |
| `faqs` | jsonb | Q/A pairs |
| `active` | bool | |

### `doctors`

| Field | Type | Notes |
|---|---|---|
| `id` | uuid PK | |
| `source` · `external_ref` | | As `clinics`. `external_ref` is the HPR id in the UHI path. **Never serialized** |
| `clinic` | FK | |
| `name` · `qualification` | string | Fictional for `seed` rows |
| `specialty` | FK | |
| `experience_years` | int | |
| `hid_masked` | string | Seeded string; **no verification copy, no tick, no tap** (FR-028) |
| `is_lead` | bool | Surfaced on B2's identity card |
| `photo_ref` | string | FR-025: no real practising clinician |
| `active` | bool | |

### `slots` — period sessions

Replaced the exact-time model on 9 Sep 2026. **One row is one session**, not
one instant.

| Field | Type | Notes |
|---|---|---|
| `id` | uuid PK | |
| `doctor` | FK | |
| `service_date` | date | Not a timestamp |
| `period` | enum | `morning` \| `afternoon` \| `evening` |
| `starts_at` / `ends_at` | time | The band rendered on B4 (`09:00`–`12:00`) |
| `capacity` | int | The clinic's number. Ours is a **cache** of it |
| `booked_count` | int, default 0 | Tokens issued |
| `source` | enum | `tdc_clinic` where published; `seed` is a `DEMO_MODE` fixture |

**Constraints:**

```sql
CHECK (booked_count >= 0 AND booked_count <= capacity)
UNIQUE (doctor_id, service_date, period)
```

The `CHECK` is the invariant, not a convention — see `research.md` R6. It
cannot be bypassed by a code path that forgets, which a `SELECT … FOR UPDATE`
discipline can.

**Availability is still not ours.** Booking against published capacity
auto-confirms because the clinic published it, but a stale cache can still
oversell. Refresh on read; clinic-initiated cancellation is the correction
mechanism, not a bug.

---

## Booking tables

### `appointments`

| Field | Type | Notes |
|---|---|---|
| `id` | uuid PK | |
| `profile` | FK | Every read and write scoped from this |
| `clinic` · `doctor` · `slot` | FK | |
| `service_date` · `period` · `period_starts_at` | date / enum / timestamptz | Denormalized from the slot. **Every cutoff computes from `period_starts_at`** — cancel, reschedule, reminders, `lapsed` |
| `token_no` | int | `booked_count` at issue. **Never renumbered, never carried across a reschedule** |
| `status` | enum | See below |
| `decline_reason` | string, null | Includes `no_response` (UHI path only) |
| `cancelled_at` | timestamptz, null | |
| `cancelled_by` | enum + FK, null | `patient` (with acting user) \| `clinic` \| `doctor`. Attribution is required: *"who cancelled Aai's appointment?"* is a question the family will ask |
| `cancellation_reason` | string, null | Clinic's reason, carried verbatim where given |
| `reschedule_count` | int, default 0 | Max 2 (FR-018) — enforced on the **chain**, not the row |
| `rescheduled_from` / `rescheduled_to` | self-FK, null | The chain is the record |
| `provider_snapshot` | jsonb | Clinic name, doctor name, address, fee **as at booking**. Required: a directory row can legitimately change after a visit, and an appointment history that silently rewrites itself is wrong |
| `created_at` | timestamptz | |

**Status enum and who may write each:**

| Status | Written by |
|---|---|
| `booked` | **Patient**, directly, on booking into published capacity |
| `requested` → `booked` / `declined` | Provider — **UHI path only**, never TDC Clinic |
| `cancelled` | Patient, **or clinic, or doctor** |
| `rescheduled` | Patient (terminal; a new row is created) |
| `completed` · `no_show` | **Clinic only.** This app never writes either |
| `lapsed` | **Nobody — derived at read time** |

`lapsed` is **never stored**. It is computed from `period_starts_at` + grace
where no clinic outcome has arrived. No scheduled job, no sweep: a background
writer mutating profile-owned rows outside a request context is an unauditable
actor in a table whose entire purpose is naming the actor (Principle XI). A row
aging in `lapsed` is a clinic-side reporting gap and should be visible as one.

### `payments`

Deliberately **not** folded into `appointments.status`: a gateway failure must
not be able to corrupt an appointment's lifecycle, and an appointment must stay
readable with its payment row in any state at all.

| Field | Type | Notes |
|---|---|---|
| `id` | uuid PK | |
| `appointment` | FK | |
| `gateway` | enum | `razorpay` |
| `gateway_order_id` · `gateway_payment_id` | string, null | |
| `amount_inr` | int | Consultation fee at booking |
| `platform_fee_inr` | int | TDC's line. Itemised separately, never bundled (FR-031) |
| `status` | enum | `pending` \| `paid` \| `failed` \| `refund_pending` \| `refunded` |
| `refunded_at` | timestamptz, null | |
| `refund_reason` | enum, null | `patient_cancel` \| `clinic_cancel` \| `declined` |
| `created_at` | timestamptz | |

**`status` moves on the verified webhook, never on the client's word**
(`research.md` R7). A client can be lied to; a signed webhook cannot.

**No card data, ever.** No PAN, no CVV, no expiry — Razorpay's hosted flow
renders every card field on both platforms (FR-031a). There is nothing in this
table a PCI auditor would scope.

**Retention — this feature is the first to hit it.** A `payments` row is a
financial record with a statutory life of its own, so it **must not cascade on
profile deletion**. On erasure it is anonymised (profile link severed, amounts
and gateway refs retained) rather than dropped. Flagged under Principle XV; the
erasure design belongs to `001`, and this is the first table that forces the
question.

---

## Relationships

```
specialties 1───* doctors
clinics     1───* doctors
doctors     1───* slots            (one row per date+period)
profiles    1───* appointments
slots       1───* appointments
appointments 1───1 payments        (nullable — pay-at-clinic has none)
appointments 0───1 appointments    (rescheduled_from / rescheduled_to)
```

## What is deliberately absent

- **No queue table.** Position is `now_serving` from the clinic compared to
  `token_no`. Persisting it would create a second truth that goes stale.
- **No `tokens_before_you` column.** Derived from the session's live token
  list, excluding cancelled and no-show tokens — **not** `your_token −
  current_token − 1`, which drifts on any cancellation and drifts toward
  telling a patient to arrive late (FR-023a).
- **No reason-for-visit field.** Re-opened by the clinic-side surface existing,
  but not built until specced: it is optional free-text PHI and needs scoping,
  logging, and exclusion from notification bodies.
- **No wait-estimate column.** Computed from `tokens_before_you ×
  avg_consult_min`, and **omitted entirely** when there is no average — never
  defaulted to a constant.
