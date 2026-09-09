# API Contract: Appointment Booking

**Phase 1 for [`plan.md`](./plan.md).** Base `/api/v1/`. Every endpoint carries
an explicit `IsAuthenticated` permission class; none is public.

**[PHI]** marks endpoints that MUST write an `access_logs` row in the same
transaction as the access (`001` T006, Principle XI). A failed log write fails
the request.

**Client-agnostic by rule** (`011` FR-022, retained): no response field may
vary by which app asked. TDC Doctor will call the same directory endpoints TDC
Health calls. A different projection is a new endpoint, not a branch inside an
existing one. `source` and `external_ref` are internal and **never serialized**.

---

## Directory — intentionally unscoped, non-PHI

No `access_logs` rows. Browsing the directory reads no PHI, and logging it
would dilute the table Principle XI exists to keep meaningful.

| Method | Path | Notes |
|---|---|---|
| `GET` | `/clinics?q=&specialty=` | Search + filter, distance ascending. `q` matches clinic, doctor and specialty **names only** — never symptoms (FR-003) |
| `GET` | `/clinics/{id}` | B2 payload: clinic, lead doctor, facilities, FAQs, fee, `accepts_pay_at_clinic` |
| `GET` | `/clinics/{id}/nearby` | 3–4 rows, seeded-distance order. No See All |
| `GET` | `/specialties` | Chip source |
| `GET` | `/clinics/{id}/doctors` | B3 sheet; next-available derived, never stored |
| `GET` | `/doctors/{id}/sessions?from=&days=5` | **Replaces `/slots`.** Returns every period session in the window with `capacity` and `booked_count`, **full ones included and flagged** |

**Why full sessions are returned rather than filtered.** A day that renders
empty reads as *"this clinic isn't open"* — a false impression the app created
by omission, which Principle VI covers as squarely as a false sentence. It also
stops the grid reflowing under the user's thumb as others book.

**Serializer rule.** Explicit `fields`, never `__all__`. This matters more here
than usual: `__all__` on `clinics` ships internal seeding fields and `source`;
on `doctors` it ships the raw `hid_masked` source.

**Standing regression guard** (carried from `011`): a test asserting these
serializers expose **no profile-derived field**. The failure mode is not
today's code — it is a later *"show which clinics this family has visited"*
convenience turning a public list into a PHI leak.

---

## Appointments — profile-scoped

Every endpoint derives its queryset from the caller's own profiles ∪ unrevoked
`caregiver_grants` (`001` T005). `get_object()` resolves from that queryset —
never `Model.objects.get(pk=…)` plus a permission check.

### `POST /appointments` **[PHI]**

```json
{ "profile_id": "uuid", "slot_id": "uuid", "pay_method": "now|at_clinic" }
```

The server resolves clinic, doctor, `service_date`, `period` and
`period_starts_at` **from the slot**. The client never supplies them, so a slot
from one clinic cannot be cross-wired to another's route.

`profile_id` is **validated against the derived set, never trusted from the
body** (FR-035). This is the single most likely thing to get wrong at
implementation time, because the request body *looks* like it carries the
authorization.

Capacity is claimed by the atomic increment in `research.md` R6; the returned
`booked_count` is the token. Writes `provider_snapshot` at creation.

| Response | Meaning |
|---|---|
| `201` | Booked. Status is **`booked`**, not `requested` — published capacity is the acceptance (FR-016a) |
| `409` | Session genuinely full. B4 refreshes and says *"That session is full — pick another"* |
| `404` | Slot or profile outside the caller's scope — **indistinguishable from nonexistent** |

### `GET /profiles/{id}/appointments?status=` **[PHI]**

B6, B8, the alerts strip and the day-of Home entry.

### `POST /appointments/{id}/cancel` **[PHI]**

Records `cancelled_by = patient` with the acting user. Releases the slot,
**propagates the cancellation to the clinic** (a slot released only in our
cache is one the clinic still thinks is taken), cancels both local reminders,
clears the alerts-strip and day-of entries, and triggers any refund.

Allowed **after** the cutoff — the copy changes, the ability does not. Refusing
a late cancellation produces a no-show, which is worse for the clinic than a
late warning.

### `POST /appointments/{id}/reschedule` **[PHI]**

```json
{ "slot_id": "uuid" }
```

Atomic: marks the old row `rescheduled`, creates the new row, links both
directions, releases the old capacity, claims the new. **Same doctor only** — a
different doctor is a different appointment. Assigns a **fresh token**; a token
is a position in one session's queue and means nothing in another.

Capped at 2 per chain (FR-018). Enforce on the **chain**, not the row —
otherwise each new row starts at zero and the cap does nothing.

### `GET /appointments/{id}/queue`

Read-through from the clinic; polled every 15s while B6 is foregrounded
(`research.md` R2). Returns `now_serving`, `tokens_before_you`, the session
state, and the wait estimate.

`tokens_before_you` is the count of live tokens **strictly between**
`now_serving` and `token_no`, excluding cancelled and no-show tokens. It is
**not** `token_no − now_serving − 1`, which drifts on any cancellation and
drifts toward telling a patient to arrive late (FR-023a).

**When the clinic feed is unreachable**, return the token and session and mark
the position unavailable. Do **not** return a stale number as current and do
**not** estimate — a wrong queue position is worse than none, because a patient
acts on it.

Not `[PHI]`-logged: it re-reads an appointment the caller already opened, and a
15-second poll would flood `access_logs` and drown the reads that matter.

---

## Inbound webhooks — the new attack surface

Both are unauthenticated by JWT and authenticated by signature. They are the
genuinely new exposure this feature adds, and both get the adversarial pass
before merge.

### `POST /hooks/razorpay/`

Payment lifecycle. **Signature verification is mandatory and is the whole
control** — an unverified payment webhook is a free-order vulnerability.

- Verify the HMAC against the endpoint secret **before parsing anything**.
- Idempotent on `gateway_payment_id`; gateways retry.
- Moves `payments.status`. **Never** moves it from a client callback.
- Carries no PHI, and must not be made to: no profile id, no patient name.

### `POST /hooks/clinic/`

TDC Clinic events: cancellation, `completed`, `no_show`, session withdrawal.

- Signed and replay-protected (timestamp + nonce, or mTLS).
- Idempotent on `(appointment_id, event, occurred_at)`.
- **A cancellation here triggers the one FCM push in this feature** (FR-016f) —
  who cancelled, when the appointment was, the reason where given, and a rebook
  action. Everything else in `012` is a device-local schedule; this one cannot
  be, because the trigger is server-side and unpredictable and the alternative
  is a patient arriving at a closed clinic.
- Refunds in full **including the platform fee** on a clinic cancellation: the
  patient did nothing wrong, and it was our flow that failed.
- **`completed` and `no_show` arrive only here.** No other path may write them.

---

## Payments

`POST /appointments/{id}/pay` creates a Razorpay order server-side and returns
what the client needs to open the hosted flow — `razorpay_flutter` on Android,
Checkout JS on web (`research.md` R7).

**TDC renders no card field on either platform.** No PAN, no CVV, no expiry
crosses our code. That is what keeps the app out of PCI-DSS scope, and it is
why the reference design's in-app CVV input was rejected (FR-031a).

Refund rules, restated because they are easy to get subtly wrong:

| Trigger | Refund |
|---|---|
| Patient cancels before the cutoff | Full |
| Patient cancels after the cutoff | **The clinic's call.** Surfaced as *"the clinic will confirm any refund"* — TDC does not adjudicate between a patient and a clinic and must not imply it does |
| Clinic or doctor cancels | **Full, platform fee included** |
| `declined` / `no_response` (UHI) | **Full, platform fee included** |

---

## Rate limiting

- `POST /appointments` — per-user cap. Not a disclosure risk; a **real-clinic**
  risk, since booking spam now reaches a real worklist. Open GAP in
  `checklists/security.md`, owner Adi.
- `GET /clinics?q=` — authenticated, non-PHI, read-only, so the exposure is
  cost. Tag `# DEMO-MODE` naming the real throttle rather than leaving it
  unremarked.
- Both webhooks — cap by source, and fail closed on signature mismatch.
