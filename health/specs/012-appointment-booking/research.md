# Research & Decision Log: Appointment Booking

**Phase 0 for [`plan.md`](./plan.md).** Eight decisions. Two are marked
**NEEDS OWNER** and neither blocks Phase 1 design — they change a dependency
and a policy, not a shape.

---

## R1 — TDC Clinic ↔ Core: events push, state pulls

**Decision.** Two directions, two mechanisms.

- **Clinic → Core, webhook.** Cancellations, `completed`, `no_show`, and
  session withdrawal are **events**: rare, unpredictable, and meaningless if
  late. `POST /api/v1/hooks/clinic/` on Core, signed and replay-protected.
- **Core → Clinic, read-through pull.** Session capacity and queue position
  are **state**: read constantly, changing continuously, and stale-tolerant for
  seconds. Core pulls on read behind a short cache and never stores them as
  truth.

**Rationale.** The split falls out of what each thing *is*. An event that
arrives late is a patient standing outside a closed clinic; state that is
seconds old is a queue number that ticks a beat behind. Pushing state would
mean a stream we must keep alive; polling events would mean discovering a
cancellation on our schedule rather than the clinic's.

`CLAUDE.md`'s architecture diagram has said `←—(P1 webhook)— CARE fork HMS`
since July. This is that line, made concrete and pointed in the direction it
was always drawn.

**Alternatives considered.**

- **Shared database.** CARE is Django and Postgres, so it is physically
  possible and would be the fastest thing to build. Rejected: it couples two
  independently-deployed services at the schema, which is exactly what
  `docs/repo-structure.md` forbids between Core and Fastlane and for the same
  reason. A CARE migration would become a Core outage.
- **Core polls everything.** Simpler — one mechanism. Rejected because a
  cancellation would surface on our poll interval, and there is no interval
  short enough to be right and long enough to be cheap.
- **Message queue between them.** Correct at scale and wrong at this one: it is
  infrastructure to run, monitor and reason about, for two endpoints and one
  developer.

**Consequence not to skip.** This is unbuilt work in `tdc-care`, a repo with no
spec. We own it, so there is no external gate — but "no gate" is not "no
effort", and it is the largest real dependency in this plan.

---

## R2 — Queue feed: poll while the screen is open, push only the two moments

**Decision.** B6 polls `GET /appointments/{id}/queue` every **15 seconds while
foregrounded**, and stops on background. The two opt-in notifications
(`3 tokens away`, `Next`) go over **FCM**, server-triggered.

**Rationale.** The spec already says queue state updates while the screen is
open, so polling matches the specified behaviour rather than exceeding it. 15s
is under the pace of a real queue — tokens do not advance faster than a
consultation — so a poll is never the bottleneck. Backgrounded polling would
drain a battery to update a screen nobody is looking at.

Push is reserved for the two moments where the user is *not* looking and the
information is worth interrupting for. That asymmetry is the whole design: pull
what you are watching, push what you would miss.

**Alternatives considered.**

- **WebSocket / SSE.** The obvious "live" answer. Rejected: it means Channels
  or an ASGI stack and a connection to keep alive, for a number that changes
  every few minutes. Revisit if a clinic ever runs a queue that moves in
  seconds.
- **Push every change.** A notification per token advance is spam, and it would
  fire for someone sitting in the waiting room already watching the screen.
- **Poll in the background too.** Battery, and no user benefit that the two
  push moments do not already cover.

**Honest-failure rule.** When the clinic feed is unreachable, B6 shows the
token and the session and says the queue position is unavailable. It does
**not** fall back to an estimate, and it does not show a stale number as
current — a wrong queue position is worse than no queue position, because a
patient acts on it.

---

## R3 — Payments live inside `012`, as a reusable module

**Decision.** `payments` is built in `012`, not spun out as its own feature —
but as a **`booking/payments.py` module with no appointment-specific logic in
its gateway layer**, so a second payment surface reuses it rather than forking
it.

**Rationale.** Workflow rule 7 says every feature owns a complete spec-kit set,
which raised the question. But a payment with nothing to pay for is not a
feature — it has no user story, no screen and no acceptance criterion of its
own. Splitting it would produce a spec that exists only to be depended on.

**Alternatives considered.**

- **Own spec (`013-payments`).** Cleaner ownership on paper. Rejected today,
  and the trigger for revisiting is concrete: **the moment a second payment
  surface appears** — the 🔒 Family Plan subscription is the obvious one — it
  graduates, because then it genuinely has a life independent of booking.
- **Gateway logic inline in the booking views.** Fastest, and guarantees the
  fork later.

**Scope boundary recorded:** refunds follow the cancellation window and the
clinic-decline rule (FR-016c, FR-031c). TDC does **not** adjudicate a refund
dispute between a patient and a clinic, and no code path may imply it does.

---

## R4 — Maps: recommend dropping the interactive map · **NEEDS OWNER**

**Recommendation, not a decision.** Replace the interactive map on B2/B6 with a
**static map image plus the OS Directions hand-off**.

**Why this is not mine to decide.** `011` decision 8 was an explicit **owner
override** — a concern was raised during specification, and the owner chose to
keep the interactive map with a written mitigation (FR-024: coordinates must
not land on a real healthcare facility). Reversing an override silently, on
implementation grounds, is exactly the drift the override record exists to
prevent.

**The case for reversing it.** An interactive map costs a maps SDK, an API key
in a shipped APK, a quota, and a standing security finding (`011`'s checklist
lists the key as an open GAP: restrict by package ID and signing fingerprint
before any build ships). It buys a pin the user does not manipulate — every
real interaction is *"take me there"*, which is the Directions hand-off, and
that works with no SDK at all. `011`'s own trim order already ranks the map
third to cut.

**The case for keeping it.** A clinic page with a live map reads as a real
product; a static image can read as a placeholder. That was the owner's call
and it is a reasonable one.

**Either way, FR-024 and FR-034 hold unchanged:** no pin on a real healthcare
facility, and Directions must not request the user's location and must fail
quietly to address text when no maps app exists.

---

## R5 — No-show consequence: build none · **NEEDS OWNER**

**Decision for now.** Detection ships; consequence does not. A `no_show` is
recorded when the clinic writes it, and **nothing happens** — no fee, no
booking restriction, no strike count.

**Rationale.** The spec flagged this as an owner decision and it stays one,
because the choice is not technical. A patient stuck in Pune traffic and a
patient who never intended to come are **identical in the data**, and any
penalty fires on evidence that cannot tell them apart. Building the mechanism
before the policy would mean choosing the policy by accident.

The safe default is also the reversible one: adding a consequence later is a
new rule over data we are already collecting. Removing one after it has charged
someone is not.

**Alternatives considered.** A no-show fee (needs a funding and refund policy
and a dispute path), a booking restriction after N (needs an appeal route),
a soft warning (does nothing, but looks like it does).

---

## R6 — Capacity: a database constraint, not a lock

**Decision.** `booked_count` is incremented **atomically in the database**, and
`CHECK (booked_count <= capacity)` enforces the invariant.

```sql
UPDATE slots SET booked_count = booked_count + 1
WHERE id = %s AND booked_count < capacity
RETURNING booked_count;
```

Zero rows affected means the session filled; the API returns `409` and B4
refreshes. The returned value is the token number.

**Rationale.** The period model already removed the double-booking race —
capacity is a counter, not a unique row two people claim (spec §Data model).
What remains is overselling, and a constraint is the right tool because it
**cannot be bypassed by a code path that forgets**. A `SELECT … FOR UPDATE`
works too, but it is a discipline; the constraint is a guarantee, and it also
issues the token in the same statement.

**Alternatives considered.**

- **`SELECT … FOR UPDATE` then write.** Correct, and one forgotten transaction
  away from wrong.
- **Read, check, write.** The thing `checklists/security.md` explicitly
  prohibits, and the shape auditors look for.
- **Optimistic retry on a version column.** More machinery than a `WHERE`
  clause that already does the job.

**Token consequence, restated because it follows from this:** tokens are the
counter's value at issue and **never renumber**. A cancellation leaves a gap and
does not decrement — decrementing would reissue a number someone already holds.

---

## R7 — Payments must be solved twice, because we ship two platforms

**Decision.** `razorpay_flutter` on Android, **Razorpay Checkout JS** on web.
Both are hosted flows; TDC renders no card field on either.

**Rationale.** This is the finding most likely to be discovered late and hurt.
The target is Android **and web** from one Flutter codebase, and Razorpay's
Flutter plugin does not cover web — a web build would either silently fail at
checkout or, worse, tempt someone into a hand-rolled card form, which is the
PCI-DSS scope violation FR-031a exists to prevent.

Both paths produce the same server-side result: an order created by Core, a
signature-verified webhook, and a `payments` row Core owns. The client is a
renderer, and which renderer it is stays a platform detail.

**Alternatives considered.**

- **Android only for payments; web is pay-at-clinic.** Defensible, and worth
  keeping as the fallback if Checkout JS in a Flutter web view proves painful.
- **Server-side redirect flow for both.** Uniform, but hands the user out of
  the app and back, which is a worse mobile experience for the majority path.

**Never trust the client's word that payment succeeded.** The `payments` row
moves on the **verified webhook**, not on the SDK callback. A client can be
lied to; a signed webhook cannot.

---

## R8 — Reminders extend `006`'s scheduler; ids must be namespaced

**Decision.** Appointment reminders use `flutter_local_notifications` through
the **same helper** `006` uses for dose reminders. One scheduler, one
stop-and-new path.

**Rationale.** `006` already solved scheduling, cancellation and rescheduling
for repeating local notifications. A second system would duplicate that and
then drift from it.

**The risk this creates, and the mitigation.** Both features share one
scheduler namespace. Dose reminders key off `(profile, slot_time)`; appointment
reminders key off `(appointment_id, offset)`. **A collision silently cancels
someone's 8pm dose reminder when an appointment moves** — a medication-safety
bug reached through a booking feature, which is the kind of failure nobody
looks for because the two features seem unrelated. Ids must be namespaced by
domain, and a test must assert that rescheduling an appointment leaves dose
reminders intact.

**Clinic cancellation is the exception** and cannot use this path at all: the
trigger is server-side and unpredictable, so it goes over FCM (R2, FR-016f).
That is the only server-push in this feature.
