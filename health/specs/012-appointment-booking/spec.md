# Feature Specification: Appointment Booking — Discovery, Slots, Lifecycle & Reminders

**Feature:** `012-appointment-booking`
**Created:** 2026-09-06
**Status:** Draft — **rescoped to production 6 Sep 2026** (see §Rescope below)
**Owner:** Adi (dev) / Soham (meetup script + copy)
**Supersedes:** [`010-consult-booking-queue`](../010-consult-booking-queue/spec.md)
and [`011-care-discovery`](../011-care-discovery/spec.md). Both are retained as
the decision record for their unlocks and overrides; neither is the
implementation reference any more. All of `010`'s fences and all eight of
`011`'s 20 Aug decisions carry forward unchanged — see §Carried forward.
**Depends on:** [`001`](../001-tdc-phr-patient-app/spec.md) (P1 slot, profile
context, alerts strip) · [`004`](../004-seed-data-and-summary/spec.md)
(fictional-facility rule, seed ripple) · [`006`](../006-medications-reminders-adherence/spec.md)
(local-notification primitive, reused here) · [`008`](../008-navigation-app-shell/spec.md)
(routes, sheets, four screen states).

> ## Rescope notice — 6 Sep 2026
>
> **This spec was written one day earlier against a Track A demo constraint
> that no longer exists.** Owner decision: the Track A / Track B split is
> abolished; one project, built to production (constitution v3.0.0).
>
> Four of this spec's defining decisions were **consequences of that
> constraint, not conclusions on the merits**, and they invert:
>
> | Was (demo) | Is now | Gate? |
> |---|---|---|
> | No clinic accept/decline; booking is one-sided | **Booking is a request TDC Clinic can accept or decline** | **None** — we own the CARE fork |
> | `slots.is_booked` local flag = "availability" | **Availability is the clinic's**; the flag is a cache | **None** |
> | Queue is a deterministic client-side simulation | **Real queue feed from TDC Clinic** | **None** |
> | Fee displayed, 🔒 pay-at-clinic, nothing collected | **Payments in scope** — Razorpay UPI-first, pay-at-clinic retained as a choice | **None** |
> | Seeded fictional directory | **TDC Clinic facilities are real**; external providers gated | **UHI onboarding** (owner: Adi) for `source = uhi` only |
> | `lapsed` — "we don't know what happened" | **`completed` / `no_show`, written by the clinic** | **None** |
>
> **The good news is structural: the seams were already built.** `source` /
> `external_ref` on `clinics`/`doctors`, the `requested → accepted | declined`
> states named in the enum, `slots.is_booked` tagged as not-authoritative, and
> the *"Booking placed, never Confirmed"* copy rule all exist precisely so this
> transition would not be a rewrite. It isn't one. But it **is** a real re-spec
> of §The confirmation problem, §Appointment lifecycle, §Payments and the queue
> block — done below — and this spec now needs its own `plan.md` and `tasks.md`
> before implementation (workflow rule 7).
>
> **What does not change:** every honesty fence. No fabricated reviews, no
> symptom search, no live-data claims without a live feed, no PHI in URLs, no
> pin on a real hospital. Those were never track-dependent.
>
> **Sections below still using demo framing** — the seed ripple, and the
> fictional-clinic rules — remain **valid and in force** for `DEMO_MODE`
> fixtures, which are still a first-class dev/test asset (Principle IV/V). They
> describe the fixture, not the product.

> **Why one spec instead of two.** `010` unlocked booking and `011` bolted
> discovery onto the front of it, which left the flow owned by two specs with a
> seam running through the middle — `011` owned three screens, `010` owned four,
> and the appointment's own life after confirmation was owned by neither.
> Reschedule did not exist. Cancellation had a verb and no policy. Nothing
> scheduled a reminder. This spec takes the whole flow, front to back, and the
> ownership split is deleted rather than maintained.

---

## Clarifications

### Session 2026-09-06

- Q: Track A demo booking, or Track B production? → A: ~~**Track A.**~~ → **SUPERSEDED same week: production.** The track split was abolished 6 Sep 2026 (constitution v3.0.0). See §Rescope notice.
- Q: Where does this live relative to `010` and `011`? → A: **New `012`, supersedes both.** They keep their decision records; this owns the flow. *(Unaffected by the rescope.)*
- Q: Does the clinic accept or decline a booking? → A: ~~**No.**~~ → **YES, as of the rescope.** TDC Clinic is the CARE fork we own, so there is no external gate on building the clinic-side surface. Booking is a request in `requested` until the clinic answers.
- Q: Which lifecycle events are in scope? → A: **Reschedule · cancellation window + no-show · reminders and notifications.** *(Still true, and `no_show` is now writable — by the clinic, not by this app.)*
- Q: Payments? → A: ~~**Specced, built later.**~~ → **In scope.** Razorpay UPI-first is a locked stack decision and payments were only out of scope because the demo said so. Pay-at-clinic survives as a **user choice**, not as a placeholder.

### Rescope, 2026-09-06 (later the same day)

- Q: Track A or production? → A: **Production. The split is abolished entirely** — there is no track to be on.
- Q: Does the seeded directory survive? → A: **As a `DEMO_MODE` fixture, yes** — off by default, never shipped as content. TDC Clinic facilities are real rows (`source = tdc_clinic`); external providers are gated on UHI onboarding.
- Q: What blocks what? → A: **Nothing on the TDC Clinic side** — accept/decline, real availability and the real queue feed are all ours to build. **UHI onboarding** gates `source = uhi` only; **ABDM certification** gates nothing in this spec.

### Session 2026-09-09 — reference design for B4/B5

Owner supplied a "Confirm Booking" reference. Four decisions taken:

- Q: Period bands or exact times? → A: **Period + token.** Morning / Afternoon / Evening with a capacity and a token number, not discrete `slot_ts` rows. A data-model change, not a layout one — see §B4 *Why a period, not a time*.
- Q: One screen or two? → A: **One.** B4 and B5 merge into a single "Confirm Booking" scroll; B5's number is retired, not reused.
- Q: The "Booking for" switcher inside checkout? → A: **Switcher yes, "+ Add New" no.** Profile creation writes a `caregiver_grants` row — the consent moment — and must not sit inside a payment flow.
- Q: Payments? → A: **Platform fee (₹50) confirmed** as a real, separately-labelled line. Pay-at-clinic was *not* settled by that answer; specced as `clinics.accepts_pay_at_clinic` pending confirmation (Open Decisions).

### Session 2026-09-09 (clarify) — who confirms a booking

- Q: When a patient confirms, who accepts or declines, and what does `012` do before a clinic-side worklist exists? → A: **Auto-book against published capacity.** If the TDC Clinic system says a slot exists, the booking is **immediately confirmed** — the clinic's published availability *is* the acceptance. Afterwards, **the clinic or the doctor may cancel**, and **the patient is notified**. That closes the loop without a pending state and without a worklist to staff.

---

## What this is / is NOT

**IS:** The complete appointment loop for TDC Health — find a clinic, choose a
doctor, choose a time, request it, get an answer from the clinic, get reminded,
attend, and reschedule or cancel if plans change.

**IS NOT — fences, and why each still stands.** Rewritten 6 Sep 2026: the
demo-driven fences are gone, the honesty fences are not. They were never
track-dependent, and several matter *more* now that the data is real.

**Still binding, permanently:**

- **No symptom search.** Symptom → specialty is medical inference. It needs a
  curated mapping, a not-medical-advice disclaimer and an owned compliance
  pass — and it puts health complaints into search logs. Pending an owner
  decision (`CLAUDE.md` §Scope); rejected until then.
- **No written reviews, review counts, or a Reviews tab.** The old objection
  (a fabricated corpus for fictional clinics) dissolves with a real directory,
  so this is now a **product and moderation decision, not a fence** — also
  pending an owner call. Until then: ratings only, as `011` decision 6 set it.
- **No claim without the thing behind it.** *live*, *real-time*, *now serving*,
  *verified*, *certified*, *partner* stay prohibited **unless the named feed or
  relationship actually exists**. With a real TDC Clinic queue feed, "live"
  becomes accurate for `source = tdc_clinic` rows and may be used there — and
  remains banned for seeded or UHI rows that have no such feed. **The word
  follows the data, per row, not the screen.**
- **No location permission** without a specced purpose; the map centres on the
  clinic, never on the user.
- **No `completed` / `no_show` written by *this* app.** Unchanged, and for the
  original reason: this app cannot observe attendance. What changed is that the
  **clinic can now write them** — see §Appointment lifecycle.
- **No PHI in URLs, no pin on a real hospital, no imagery of a real facility or
  a real clinician.** FR-024/FR-025.

**Repealed by the rescope** (each was demo-only): ~~no real clinics~~ ·
~~no UHI calls~~ · ~~no payment collection~~ · ~~no live availability~~ ·
~~no teleconsult~~ (unspecced, not prohibited). Fictional-facility rules
continue to govern **`DEMO_MODE` fixtures**, which are still first-class.

---

## The confirmation problem — now solved properly

`011` named this and left it as a warning. Yesterday's version of this spec
turned it into a copy rule. **Today it gets its real fix.**

Availability is owned by the clinic. Our copy of it — `capacity` and
`booked_count` since 9 Sep, `is_booked` before that — is a **cache of someone
else's truth**, and a system that treats it as authoritative will cheerfully
confirm a session the clinic closed an hour ago.

**Resolution (owner decision, 9 Sep 2026): published capacity is the
acceptance.**

1. Availability comes from the clinic's own system. If TDC Clinic publishes a
   session with room in it, that publication **is** the clinic saying yes.
2. Booking into it therefore resolves **immediately to `booked`**. There is no
   pending state, no worklist for someone to watch, and no request that can
   rot.
3. **The clinic or the doctor may cancel afterwards** — a session gets pulled,
   a doctor is called away. That is a real event, it flows back to us, and
   **the patient is notified**. See §Clinic-initiated cancellation.

**This is what makes "Confirmed" honest.** The earlier draft banned the word
outright because nothing on our side could ever earn it. Now it can: where the
capacity is the clinic's own published data (`source = tdc_clinic`), a booking
against it **is** confirmed and may say so. The ban survives exactly where the
data does not — a `DEMO_MODE` fixture or an un-onboarded provider has published
nothing, so "Booking placed" is the honest wording there. **Same per-row rule
as the word "live": the copy follows the data, not the screen.**

**What this removes.** A pending `requested` state on the TDC Clinic path, the
response-window timeout, and the `no_response` decline — none of which have a
job any more, because nobody is being waited on. They stay in the model for a
provider that genuinely requires confirmation before a booking is real; that is
not TDC Clinic, and it is not the default path.

**What it does not remove.** Our `capacity` / `booked_count` is still a
**cache** of the clinic's number. Booking is confirmed against what the clinic
published, which is a much stronger claim than a local flag flip — but a stale
cache can still oversell a session. Refresh on read, and treat a clinic-side
cancellation as the correction mechanism rather than pretending it cannot
happen.

**For providers reached through UHI** (`source = uhi`), acceptance may
genuinely be asynchronous — UHI's protocol, not ours. That is where the
`requested` / `declined` states earn their place, and that path is **GATED on
UHI onboarding (owner: Adi)**. The states exist in the model from day one so
that adding UHI is a new transport rather than a new lifecycle.

---

## Constitution check

**VIII (Home Stays Quiet) — preserved, not amended.** Booking is reachable only
from a profile's action row, beside Health Summary. No Home entry, no nav slot,
no tab, no clinic feed. Home's entire relationship to this feature is the
alerts strip: an upcoming appointment appears there within 48 hours and
resolves after the visit. It is a **resolving alert**, not an inbox item — the
same category as a low-stock warning, and bounded by FR-030 so that a family of
four cannot fill Home with appointments.

**VI (claims must be literally true).** The load-bearing principle for this
spec. Three surfaces sit closest to the line — the seeded queue, the seeded
rating, the seeded HID chip — and each carries its mitigation as an FR (FR-026,
FR-027, FR-028). §The confirmation problem is a Principle VI decision before it
is a UX one.

**IV (demo-grade, stated honestly).** `lapsed` exists as a terminal state
precisely for the window where the clinic has not yet reported an outcome. Inventing an
outcome we did not observe is the exact failure this principle names.

**VII (elderly-first).** B4 is the densest screen in the app — date strip,
period rows, profile switcher and payment summary in one scroll. The period
model helps here rather than hurting: three full-width rows are far more legible
at arm's length than the grid of small time chips it replaced, which was the
classic accessibility failure. FR-013 holds the 48dp floor on every cell.

**X (authorization derived server-side).** The directory (clinics,
doctors, specialties, slots) is non-PHI reference data readable by any
authenticated user — an intentional unscoped read, recorded in
`checklists/security.md`. Everything touching `appointments` is scoped from the
caller's own profiles ∪ unrevoked `caregiver_grants`, with no exception for
reads.

**XI (access logging).** Browsing the directory logs nothing — no PHI is
read. Creating, rescheduling and cancelling an appointment each write an
`access_logs` row.

**XIV (every demo path is inventoried).** Seven
`# DEMO-MODE` tags required — see FR-029.

---

## Screens

Seven screens, `B1`–`B7`. `B1`–`B3` are `011`'s `D1`–`D3` carried forward
substantially unchanged; `B4`–`B7` are new or newly detailed.

### B1 — Find care (`/profile/:profileId/book`)

Themed to `docs/design-tokens.md` — `primary #4B83F2`, warm neutrals, large
type, arm's-length legibility. Top to bottom:

| Element | Behaviour |
|---|---|
| **Profile context header** | *"Booking for Aai (Asha K.)"* + avatar. Replaces the reference's hamburger / brand / city-dropdown header. Non-negotiable: it keeps entry through the person and prevents the wrong-patient booking error class. |
| **Area label** | A real location control at production. A static seeded label (`Pune`) is the `DEMO_MODE` fixture. |
| **Search field** | Placeholder **"Search clinics or doctors"**. Matches clinic name, doctor name, specialty name. Debounced server query. No filter icon — the chips are the filter. |
| **Specialty chips** | Icon + label from `specialties`; 4 primary visible + **More** to expand. Tap filters, tap again clears. Complete icon set required — the reference ships a visible missing-glyph fallback (`H4` for Orthopedics); ours must not. |
| **Nearby clinics list** | Card per clinic: image, name, `2.5 km away`, `~10 min wait`, `3 waiting`, chevron. Seeded-distance ascending. |

### B2 — Clinic page (`/profile/:profileId/book/:clinicId`)

| Element | Behaviour |
|---|---|
| **App bar** | Back + clinic name. **Share icon: re-open.** It was dropped because a fictional clinic page had nowhere to go; a real clinic page is shareable. Owner: Adi — needs a share target and a no-PHI check first (a clinic page carries none, but the URL must not carry `profileId`). |
| **Hero image** | Seeded clinic image (FR-025 governs sourcing). |
| **Identity card** | Name · seeded star rating, no review count · lead doctor: name, `Senior Cardiologist • 15+ years`, seeded masked `HID: ****8821` chip. |
| **Fee line** | **`Consultation ₹400`**, with `Pay at clinic` shown only where the clinic accepts it. New in `012` — see §Payments. Shown here and again on B5, so the number is never a surprise at the confirm step. |
| **Location** | Interactive map on the clinic's seeded coordinates · fictional Pune address · **Directions** hands off to the OS maps app. |
| **Other clinics nearby** | Horizontal rail, 3–4 cards, seeded-distance order. No See All. Tapping replaces the current clinic page. |
| **Tabs** | **Facilities** (icon tiles: parking, wheelchair access, pharmacy, wi-fi, lift, lab) and **FAQ** (seeded Q&A). No Reviews tab. |
| **Sticky CTA** | **Book Appointment** → B3. Always visible, never scrolls away. |

### B3 — Choose doctor (`/profile/:profileId/book/:clinicId/doctors`) — sheet

Per `008`'s sheet convention. One row per doctor: name, specialty,
qualification, and a next-available label derived from `slots` (never stored).
One decision, one tap. A doctor-name hit in B1's search opens this sheet with
that doctor highlighted and the clinic already resolved.

**Never auto-skipped for single-doctor clinics** (resolves `011`'s open
decision). Predictability beats smoothness mid-demo: a screen that sometimes
appears is a screen the demoer has to think about.

### B4 — Confirm Booking (`/profile/:profileId/book/:clinicId/:doctorId/confirm`)

**Rewritten 9 Sep 2026 against a supplied reference design.** This screen was
previously two — B4 (a grid of exact-time chips) and B5 (a confirm summary).
They are now **one scrolling screen titled "Confirm Booking"**, and the booking
unit changed from an exact time to a **period band + token**. Both are owner
decisions, taken 9 Sep; §Why a period, not a time explains the second because it
is a data-model change, not a layout preference.

Top to bottom:

| Element | Behaviour |
|---|---|
| **App bar** | Back + **"Confirm Booking"**. |
| **Provider card** | Doctor photo, name, specialty, and the clinic name + address beneath a pin icon. Confirms all three earlier choices in one block so nothing has to be remembered. |
| **Select Date** | Horizontal strip of **5 days** (`Mon 12` … `Fri 16`), one selected, plus a **Calendar** affordance top-right for anything beyond the strip. Selected day fills with `primary`. Days with no remaining capacity render disabled, not hidden — same reasoning as the old grid: an absent day reads as "closed", which is a false impression made by omission. |
| **Select Time Period** | **Morning · Afternoon · Evening**, each a full-width row showing its band (`09:00 AM – 12:00 PM`). One selected. A period at capacity is disabled with a plain reason (*"Full"*), never removed. A clinic that does not run a period simply has no row for it. |
| **Booking for** | Avatar row of the caregiver's profiles — `Self`, plus each managed profile — with the current one selected. **No "+ Add New"** (see below). |
| **Payment Summary** | `Consultation Fee ₹800` · `Platform Fee ₹50` · a divider · **`Total ₹850`**. Every line labelled; no rounded-up or bundled figure. |
| **Cancellation line** | *"Free to cancel or reschedule up to 2 hours before."* Stated before the button, as on the old B5 — a fee makes this more important, not less. |
| **Sticky bottom bar** | Payment-method selector (e.g. `PhonePe ⌄`) + primary **`Confirm & Pay`**. Stays visible; the summary above it scrolls. |

**Why a period, not a time.** An exact-time slot promises something an OPD
queue does not deliver — a 10:30 appointment that runs 40 minutes late is a
broken promise the app made on the clinic's behalf. A period band plus a token
describes what actually happens: arrive in the morning, you are number 16. It
also makes the queue view on B6 coherent rather than decorative, and it
**removes the double-booking race entirely** — capacity is a counter, not a
unique row two people can claim (see §Data model, and the concurrency section
of `checklists/security.md`, which this supersedes on that point).

**"Booking for" keeps the switcher and drops "+ Add New."** The switcher earns
its place: wrong-patient booking is the error class this flow exists to
prevent, and the last moment before paying is exactly when a caregiver notices.
**Creating a profile is a different matter** — it writes a `caregiver_grants`
row, which is *the* consent moment (`001` FR-003, constitution Principle II).
A consent decision must not be a step inside a payment flow, where the user's
attention is on a total and a button. Profile creation stays on Home's
persistent Add-family action (`001` FR-023).

**Payment-method copy must not overclaim.** The selector shows the user's UPI
app because Razorpay's UPI intent flow hands off to one. The screen must not
say or imply that TDC integrates with PhonePe, or with any named app — it
integrates with Razorpay, and the app shown is the user's choice (Principle VI;
"partner" is already a banned word on these screens).

**On the reference's theme:** it is rendered in teal. Ours is `primary #4B83F2`
per `docs/design-tokens.md`, same retheme `011` already recorded for the
find-care reference.

### B5 — retired

Merged into B4 on 9 Sep 2026. The number is **retired rather than reused**, so
existing cross-references to B6 and B7 stay correct.

### B6 — Appointment detail / Queue Status (`/profile/:profileId/appointment/:apptId`)

**Rewritten 9 Sep 2026 against a supplied reference design.** The screen is
**queue-first on the day of the appointment** and summary-first before it —
the reference leads with the token because that is the only thing anyone opens
this screen for once they are in the waiting room.

| Element | Behaviour |
|---|---|
| **App bar** | Back + **"Queue Status"** on the day; the appointment's date before it. A bell toggles queue notifications (see below). |
| **Session state banner** | A single-line statement of what the queue is doing: **`Queue Moving` — *"Doctor is currently seeing patients."*** Other states: **`Not started`** (session hasn't opened), **`Paused`** (doctor away — say so, don't leave a stalled number), **`Delayed`** (running behind), **`Closed`**. Semantic colour only, `ok` / `warn` per `docs/design-tokens.md` — never decoration. |
| **Your token** | The largest element on the screen, with an **`In Queue`** state chip. Chip changes with the appointment's own state (`In Queue` · `Next` · `Called` · `Done`). |
| **Current token** | What the clinic is serving now, from the feed. |
| **Tokens before you** | **Derived, never stored** — see the arithmetic note below. |
| **Estimated wait** | *"Approx. 45 mins"* plus the required disclaimer: *"Wait times are estimates and may vary based on consultation duration."* The disclaimer is **not optional** (Principle VI) — an estimate presented as a promise is a claim we cannot keep, and this is the one number on the screen a patient will plan their morning around. |
| **Clinic card** | Photo, name, pin. **Directions** (OS maps hand-off, B2 rules) and **Call** — a `tel:` hand-off to the clinic's published number. New in this reference and worth having: the reason a patient stares at this screen is usually a question only the front desk can answer. |
| **Reschedule** | Visible until the cutoff, then hidden with an explanatory line. → B7. |
| **Cancel** | Available before or after the cutoff. One action + a confirm dialog naming who and when. |

**Before the day**, the queue block is **absent, not empty**, and the screen
leads with the summary block — who, where, with whom, when, token — in the same
order B4 confirmed them, so it is recognisably the thing that was booked. The
status chip reads `Requested` · `Booked` · `Declined` · `Cancelled` ·
`Rescheduled` · `Past`, and **never `Confirmed` before the clinic has
accepted** (§The confirmation problem).

**The arithmetic must be stated, or it will drift.** In the reference: your
token 28, current token 12, tokens before you **15** — not 16. The one being
seen is not waiting. So:

```
tokens_before_you = count of live tokens strictly between current_token and your_token
```

Cancelled and no-show tokens are **not** counted, which is why this is derived
from the session's live token list rather than by subtracting two numbers. A
naive `your_token − current_token − 1` drifts the moment anyone cancels, and it
drifts in the direction that makes a patient arrive late.

**Estimated wait is derived the same way** — `tokens_before_you × the clinic's
observed average consultation time for that session`, never a fixed constant.
Where no average is available yet, show the token counts and **omit the
estimate entirely** rather than inventing one.

**Divergences from the reference (deliberate, 9 Sep 2026).**

| Reference | TDC | Why |
|---|---|---|
| Bottom tab bar: Status · Clinics · History · Profile | **No tab bar.** B6 is a pushed screen with a back arrow | `008`'s hub-and-spoke model: a global tab bar implies app-level sections that do not exist, and elderly-first means fewer persistent controls, not more |
| **Clinics** tab | **Not built** | A browse destination reachable without choosing a patient first — the exact surface Principle VIII (NON-NEGOTIABLE) and `011`/`012` reject. TDC coordinates care *for a person*; it does not browse inventory |
| **History** tab | Already exists as the profile timeline (`005`) | A second history surface would be a parallel view of the same data |
| **Profile** tab | Home + Settings already carry this | — |

The reference is a *booking app's* shell. TDC is a family health record that
also books, and the two want different navigation. The queue screen itself is
adopted almost wholesale; only its chrome is not.

**Getting back here from a waiting room.** Dropping the tab bar costs
re-entry, so B6 is reachable three ways, none of which is a browse destination:
the T−2h reminder deep-links straight to it, the alerts strip carries it, and
on the day of the appointment Home shows a **day-of entry** (below).

**Queue notifications (the bell).** Now honest, and only now: `010` and this
spec's earlier draft both **banned** queue-position push, because firing
*"you're next"* from a client-side simulation fabricates a real-world event.
With a real `tdc_clinic` feed the event is real, so it may be sent — **opt-in,
off by default**, at two moments only (`3 tokens away` and `Next`). It stays
banned for any session without a real feed. Same per-row rule as the "live"
wording: **the notification follows the data, not the screen.**

### B7 — Reschedule (`/profile/:profileId/appointment/:apptId/reschedule`)

**B4's date + period picker again**, with three differences: the context bar
reads *"Moving your Tuesday morning appointment"*; the currently-held session is
marked `Current` rather than disabled; and the primary action reads
`Move appointment`. **The payment block does not reappear** — the booking is
already paid for, and a reschedule within the same clinic and fee is not a new
transaction. Confirming
goes through a short confirmation sheet showing old time → new time — not a
second full confirm screen, because the who / where / with rows have not
changed and re-asking about them would be a decision the user did not make.

**Same doctor only.** A different doctor is a different appointment: cancel and
book again. Reusing a reschedule flow to switch doctors would silently change
who the visit is with while the screen says "reschedule".

### B8 — Visit History (`/profile/:profileId/visits`) — added 9 Sep 2026

**A visit is a completed appointment.** From a supplied "Visit History"
reference. No new entity: `appointments` with `status = completed` already
carries the clinic, the doctor, the date and a provider snapshot, so the
narrative timeline the reference shows is a **view**, not a table.

| Element | Behaviour |
|---|---|
| **Timeline** | Vertical rail, newest first, one card per completed visit. |
| **Card** | Date chip · title (the reason or specialty) · clinic · doctor · a clinical summary paragraph. |
| **Summary text** | **Written by the clinic, never by us.** It arrives with `completed` through the same TDC Clinic surface, and is rendered attributed. Where no summary was written, the card shows the visit without one — it does **not** get a generated line. |
| **Empty** | *"No past visits yet."* Not an error, and not a prompt to book — Principle VIII. |

**Why this is not a second records timeline.** `005`'s timeline holds
**documents the user captured**; this holds **encounters the clinic recorded**.
They answer different questions (*"where is the prescription"* vs *"what
happened at that appointment"*) and have different authors. A visit links to
any records attached to it; it does not absorb them.

**A visit that TDC did not book has no row here.** The reference implies a
complete clinical history; ours only knows about appointments made through the
app. That gap is real and must not be papered over — the screen is *Visit
History*, not *Medical History*, and copy must not imply completeness. Manual
visit entry is a possible future feature and is **not** specced here.

### Screen states

All seven implement `008`'s four states: skeleton cards while loading ·
instructive empty states that name their recovery action · plain-language error
+ Retry · **no offline pretence**. The single exception is already-scheduled
local reminders, which fire without network by nature (`006`).

---

## Appointment lifecycle

**Rewritten 6 Sep 2026.** The previous version had five states and one honest
apology: with no clinic surface, nothing observed attendance, so `lapsed` meant
*"this time passed and we do not know what happened."* **TDC Clinic is ours to
build against, so we can know.**

| From | To | Trigger | Who owns it |
|---|---|---|---|
| — | `booked` | Patient confirms on B4 against published capacity | Patient — the clinic's published availability is the acceptance |
| — | `requested` | Same, for a provider that requires async confirmation (UHI) | Patient; **not the TDC Clinic path** |
| `requested` | `booked` / `declined` | Provider answers, or the response window expires | Provider, or the timeout — **UHI path only** |
| `booked` | `cancelled` | Patient cancels on B6, **or the clinic or doctor cancels** | Either side — see §Clinic-initiated cancellation |
| `booked` | `rescheduled` | Patient moves it on B7 | Patient (re-enters `requested`) |
| `booked` | `completed` | Visit happened | **Clinic** |
| `booked` | `no_show` | Patient did not attend | **Clinic** |
| `booked` | `lapsed` | `period_starts_at` + grace passed with **no clinic outcome** | Nobody — derived |
| `declined` / `cancelled` / `rescheduled` / `completed` / `no_show` / `lapsed` | — | Terminal | — |

**`lapsed` survives, with a smaller and better-defined job.** It is no longer
"we have no clinic"; it is **"the clinic has not told us yet."** Still derived
at read time from `period_starts_at` + grace, never stored, never swept by a job — so
there is no background writer mutating profile-owned rows outside a request
context, which would be an unauditable actor in a table whose purpose is naming
the actor (Principle XI). A row that sits `lapsed` for days is a **clinic-side
reporting gap**, and should be visible as one rather than quietly aging.

**This app still never writes `completed` or `no_show`.** Not a capability
limit any more — a correctness rule. `completed` asserts an event only the
clinic witnessed, and `no_show` is an accusation. Both arrive from TDC Clinic
or not at all. FR-016 is unchanged in text and stronger in meaning.

**No-show consequences remain unspecified, deliberately.** We can now *detect*
one, which is exactly why the policy question is live rather than moot: whether
a no-show costs a fee, a booking restriction, or nothing is a **business
decision with fairness implications** — a patient stuck in Pune traffic and a
patient who never intended to come look identical in the data. Owner: Adi. No
consequence ships until that decision is taken; detection without a policy is
fine, a policy invented here is not.

**The request window.** A `requested` appointment that the clinic has not
answered within `clinics.response_window_min` (default 30) resolves to
`declined` with reason `no_response`, and the slot is released. Surfaced to the
patient plainly, with the alternative slots offered inline. Owner of the
default: Adi.

### Clinic-initiated cancellation (added 9 Sep 2026)

A session gets pulled; a doctor is called away. This is the other half of the
auto-book loop, and without it auto-booking would be a promise we cannot keep.

- **It is a real state change, not a silent one.** Status → `cancelled` with
  the acting side recorded (`cancelled_by` distinguishes patient, clinic and
  doctor) and the clinic's reason carried through verbatim where given.
- **The patient MUST be notified, and it MUST be a push.** Every other
  appointment notification in this spec is a device-local schedule (`006`'s
  primitive). This one is not: the trigger is server-side and unpredictable, so
  it goes over **FCM**, which is already in the locked stack and already used
  for the family blast. This is the only server-push in `012`, and it exists
  because the alternative is a patient arriving at a clinic that is closed.
- **The notification says what happened and what to do next** — who, when, why
  if given, and a direct action to rebook. It must not read as an app error.
- The slot is released, both local reminders are cancelled, the alerts-strip
  and day-of entries clear, an `access_logs` row is written, and **any payment
  is refunded in full including the platform fee** — the patient did nothing
  wrong (FR-016c's reasoning, now reached by a second route).

### Cancellation

- **Cutoff: 2 hours before `period_starts_at`**, seeded per clinic in
  `clinics.cancellation_window_min` (default 120) rather than hard-coded, so
  it varies per provider without a schema change.
- **Before the cutoff:** cancel freely. Copy: *"Cancelled. Nothing else to do."*
- **After the cutoff: still allowed.** The app never traps a patient in an
  appointment they cannot attend — refusing a late cancellation produces a
  no-show, which is worse for the clinic than a late warning. The copy changes,
  not the ability: *"Cancelling this late — the clinic may not be able to fill
  the slot."* Stated once, without scolding.
- One action + a confirm dialog that names who and when, because a cancel
  button on the wrong family member's appointment is a real error class.
- On cancel: status → `cancelled`, `cancelled_at` set, **the cancellation is
  sent to the clinic** (a slot released only in our cache is a slot the clinic
  still thinks is taken), both reminders are cancelled, the alerts-strip row
  clears, and an `access_logs` row is written.
- **Cancelling a `requested` appointment** withdraws the request. Allowed at
  any time — there is nothing to be late for until the clinic has accepted.

### Reschedule

- **Same cutoff** as cancellation, and **same doctor** only.
- **Maximum 2 reschedules per booking** (`reschedule_count`). Not a punishment
  — a bound. Unlimited rescheduling turns one appointment into an unbounded
  chain of rows pointing at each other, which is a data-model problem before it
  is a policy one. After two, the Reschedule action is replaced by an
  explanatory line pointing at Cancel.
- Reschedule **creates a new `appointments` row** and marks the old one
  `rescheduled` with `rescheduled_to` set. It does not mutate the existing row.
  The chain is the record of what happened, and B6 on an old row shows *"Moved
  to Thu 10:30 AM"* with a link forward.
- **The token does not carry over.** A token is a position in a specific
  session's queue; carrying it to another day would be meaningless at best and
  wrong at worst. The new row gets a fresh token, and the reschedule
  confirmation sheet says so.
- Reminders are cancelled and re-scheduled against the new time (`006`'s
  stop-and-new pattern, FR-021).

---

## Reminders & notifications

**Reuses `006`'s primitive exactly** — `flutter_local_notifications`,
device-local. This is not a new notification system,
and building one would be the wrong call for two rows in a seeded demo.

| When | Copy | Deep link |
|---|---|---|
| **T−24h** | *"Tomorrow: Asha's appointment with Dr. Kavya Rane, 10:30 AM, Sunrise Poly Clinic."* | B6 |
| **T−2h** | *"In 2 hours: Asha's appointment, 10:30 AM. Token 16."* | B6 |

- Both are **per profile, delivered to the caregiver's device** — same model as
  `006`'s dose reminders. The caregiver receives them for every profile they
  manage.
- A reminder tap lands on B6 directly, satisfying `001` NFR-006: notification →
  action in under 10 seconds, no more than one intermediate screen.
- A T−24h reminder for a booking made less than 24 hours out is **not
  scheduled**, rather than fired immediately. A notification that arrives while
  the confirm animation is still on screen reads as a bug.
- Cancel or reschedule **cancels both notifications** and, for a reschedule,
  schedules two new ones. Never leave an orphaned reminder for an appointment
  that no longer exists — the failure mode `006` FR-009 already names.
- **No queue-position push.** *"You're next"* requires a live feed that does not
  exist; sending it from a client-side simulation would be a fabricated
  notification about a real-world event. The queue updates only while B6 is
  open, and that is the honest ceiling.

### Home alerts strip

An upcoming appointment appears as one alerts-strip row: *"Baba — Dr. Kavya,
tomorrow 10:30, token 16."* It resolves after the visit and clears immediately
on cancellation.

**Bounded so it cannot become a feed (Principle VIII):** at most one row per
appointment, shown only **within 48 hours** of `period_starts_at`, capped at **two
appointment rows total** across all profiles, and never ranked above a
`low stock` or `missed dose` alert. The alerts strip is a retention engine for
things the user must act on; an appointment four days out is not one of them.

---

## Payments

**Rescoped 6 Sep 2026 — in scope.** The previous version shipped a fee display
with a 🔒 badge and designed the rest for "later". Payments were only out of
scope because the demo said so; Razorpay UPI-first is already a locked stack
decision (`CLAUDE.md`), so there is nothing to choose and nothing gating it.

**Pay-at-clinic becomes a per-clinic capability, not a per-booking toggle**
(9 Sep 2026). The reference design shows a single `Confirm & Pay` and no
pay-at-clinic option, which is right for a clinic that requires prepayment and
wrong for the many that don't — it is still how most of this market pays, and
removing it outright would cost bookings. So it lives on the clinic:
`clinics.accepts_pay_at_clinic`. Where true, B4 offers both; where false, B4 is
exactly the reference. This keeps checkout uncluttered without deleting the
option. *(My call, not an explicit owner decision — the multi-select that
confirmed the platform fee did not settle this. Flagged in Open Decisions.)*
The 🔒 badge comes off either way: a lock icon on a real option is a lie about
the product.

**Two fees, both labelled.** Confirmed 9 Sep 2026 from the reference:

| Line | Source | Notes |
|---|---|---|
| `Consultation Fee` | `clinics.consult_fee_inr` | The clinic's money. |
| `Platform Fee` | `platform_fee_inr`, config not per-clinic | **TDC's money**, and the first time this product charges for itself. |
| `Total` | sum | Shown explicitly; never a single bundled figure. |

**The platform fee needs copy, not just a line.** Principle VI governs pricing
claims as much as trust claims: the line must say what the fee is *for*
(booking and coordination), and must not be presented as a tax, a clinic
charge, or a government levy. A tappable explainer next to it is the minimum.
**Unresolved:** whether it applies to a pay-at-clinic booking, where TDC
collects nothing else. Owner: Adi.

### Checkout surface (reference design, 9 Sep 2026)

Taken from the supplied "Payment Checkout" reference. **Adopted:**

| Element | Behaviour |
|---|---|
| **Booking summary at top** | Provider, consultation type, date + period. Confirms what is being paid for before any method is chosen. |
| **`Total Amount Due` + `View fee breakdown`** | Collapsed total with an expander showing consultation + platform fee. Satisfies FR-031's itemisation without a wall of numbers — the breakdown must be **one tap away, never hidden behind a help page**. |
| **Contact block with `Edit`** | Name + phone the clinic will use. Editable here because a wrong number is discovered exactly now. |
| **UPI first, marked fastest** | Matches the locked Razorpay UPI-first decision. App choices (GPay / PhonePe / Paytm) are **the user's UPI apps**, not TDC integrations — copy must not imply otherwise. |
| **Saved cards, tokenised per RBI directive** | Correct and required — RBI card-on-file tokenisation. The token is Razorpay's; TDC stores no PAN. |
| **Netbanking / Wallets / Pay Later** | Razorpay-provided method families. See the Pay Later decision below. |
| **Sticky bottom bar** | Selected method + `Pay ₹850.00`. The amount is on the button — good, and it must always match the total above it. |

**Rejected, with reasons.** Four of these are locked-copy or compliance
violations, not preferences:

| Reference | Verdict | Why |
|---|---|---|
| **CVV field rendered in-app** next to a saved card | **REJECTED — compliance** | The moment our UI touches a CVV, the app is in **PCI-DSS scope** (SAQ A-EP or worse) and a WASA/PCI finding follows. Card fields MUST be rendered by **Razorpay's SDK/iframe**, never by TDC. FR-031a already says card data must not touch TDC systems; this is what that means concretely. Non-negotiable. |
| **"Verified Healthcare Partner"** badge | **REJECTED — banned copy** | *Verified* and *partner* are **both** prohibited on these screens (FR-026), and neither is true: no verification is performed and no partnership exists. This is the same class as the banned "ABDM certified". |
| **"256-bit SSL Protection"** | **REJECTED — banned copy** | Cipher names in UI are banned outright (`CLAUDE.md`): *"AES-256", "military-grade"*. Approved wording is **"encrypted"**. |
| **"PCI-DSS Level 1 Certified"** on our screen | **REJECTED — misattribution** | That is **Razorpay's** certification, not TDC's. Stating it beside our brand implies we hold it. *"Payments handled by Razorpay"* is true and sufficient. |
| **"Flat ₹50 cashback via CRED UPI & PhonePe"** | **REJECTED as specced** | An offer tied to named payment providers is a promotion **and** sponsored placement — both prohibited (FR-039). It also creates a duty to honour it. Needs an owner decision before any version of it exists. |
| **"CareConnect Guarantee — instant 100% refund if the doctor misses the slot"** | **DECISION NEEDED** | Not banned, but it is a **commercial guarantee** we would have to honour and fund, and "instant" and "100%" are literal claims under Principle VI. It is also a genuinely good trust signal. Owner: Adi. Not specced until decided. |
| **"Live Session" / teleconsult** | **OUT OF SCOPE, unspecced** | The reference books a *telehealth* consultation. This spec books an in-person visit with a token and a queue — a teleconsult has no token, no queue, and no Directions button. It is not prohibited, it is **a different feature needing its own spec**. |

**Refunded in full on a clinic decline — platform fee included.** A `declined`
or `no_response` booking means the patient paid and got nothing, and the
failure was in *our* request flow. Keeping a coordination fee for coordination
that did not happen is indefensible.

| Element | Behaviour |
|---|---|
| **Fee** | `clinics.consult_fee_inr`, shown on B2 and again on B4. For `source = tdc_clinic` this is the clinic's real published fee; a seeded value is a `DEMO_MODE` fixture. |
| **Choice at confirm** | `Pay now` always; `Pay at clinic` only where `clinics.accepts_pay_at_clinic`. Neither is pre-selected in a way that hides the other. |
| **Method selector** | Razorpay UPI intent hands off to the user's UPI app, shown in the sticky bar (`PhonePe ⌄`). Copy MUST NOT imply TDC integrates with any named app — the integration is Razorpay; the app is the user's choice (Principle VI, and "partner" is already banned here). |
| **Gateway** | Razorpay, UPI-first (locked). No second provider is evaluated. |
| **Payment states** | On a **separate `payments` table**, never on `appointments.status`. A gateway failure must not be able to corrupt an appointment's lifecycle, and an appointment must stay readable with its payment row in any state. |
| **Failure** | A failed payment MUST NOT destroy the booking. Where the clinic accepts pay-at-clinic, fall back to it with the appointment intact. Where it does not, the appointment holds in `requested` with a retry — the token is not released on a gateway timeout. Losing a slot to a payment glitch is the worst outcome available here. |
| **Refunds** | Follow the cancellation window, not a separate policy. Cancel before the cutoff → full refund. After → *"the clinic will confirm any refund"* — **TDC does not adjudicate a refund between a patient and a clinic**, and must not imply that it does. |
| **Declined request** | A `declined` or `no_response` booking that was paid for is **refunded automatically and in full, platform fee included**. The patient did nothing wrong and got nothing, and it was TDC's request flow that failed. This is the one refund path TDC decides itself. |

**Compliance note.** Payments introduce a surface this spec did not previously
have: PCI scope is avoided by never touching card data (Razorpay's hosted flow
handles it), and payment identifiers are **not PHI but are profile-linked** —
so `payments` rows are scoped from `caregiver_grants` exactly like
`appointments`, and appear in `checklists/security.md`.

## Data model

Builds on `011`'s four directory tables (`specialties`, `clinics`, `doctors`,
`slots`), all in TDC Core API's `directory` app. Additions and amendments only.

### `clinics` — two new columns

| Field | Notes |
|---|---|
| `consult_fee_inr` | int. The clinic's published fee; a seeded value is a `DEMO_MODE` fixture. Displayed on B2 and B4. |
| `accepts_pay_at_clinic` | bool. When false, B4 is prepay-only (the reference design). When true, B4 offers both. |
| `cancellation_window_min` | int, default `120`. Cutoff for cancel and reschedule; per-clinic. |
| `response_window_min` | int, default `30`. How long a `requested` booking waits for the clinic before resolving to `declined` / `no_response`. |

### `slots` — replaced by period capacity (9 Sep 2026)

The old shape was `slots(doctor, slot_ts, is_booked)` — one row per bookable
instant, claimed by flipping a boolean. The period model replaces it:

| Field | Notes |
|---|---|
| `id` | PK |
| `doctor` FK | |
| `service_date` | date, not a timestamp |
| `period` | `morning` · `afternoon` · `evening` |
| `starts_at` / `ends_at` | local times defining the band shown on B4 (`09:00`–`12:00`) |
| `capacity` | int — how many tokens this session issues |
| `booked_count` | int — issued so far |
| `source` | `tdc_clinic` where the clinic publishes it; `seed` is a `DEMO_MODE` fixture |

**This deletes the double-booking race rather than mitigating it.** The old
model had two people claiming one unique row, which needed a lock or a unique
constraint and produced a `409` for the loser. Capacity is a **counter**: two
simultaneous bookings both succeed and get tokens 16 and 17. The only failure
is a genuinely full session.

**The counter still needs to be correct.** `booked_count` MUST be incremented
under a row lock (`SELECT … FOR UPDATE`) or by an atomic DB-side increment with
a `booked_count < capacity` check — never read-then-write. Overselling a session
by three is a smaller harm than double-booking a slot was, but it is still the
clinic absorbing our race.

**Token = the counter's value at issue time, and never renumbers.** Token 16 is
the 16th booking of that session. A cancellation leaves a gap; tokens are not
compacted, because a token that changes after it was given out is worse than a
missing number.

**Availability is still the clinic's.** `capacity` and `booked_count` are our
copy of what the clinic published, refreshed from it. A booking is still a
**request** (§The confirmation problem) — having a token is not having an
accepted appointment.

### `appointments` — rewritten

Replaces `010`'s table and `011`'s amendment to it.

| Field | Notes |
|---|---|
| `id` | PK |
| `profile` FK | Who the visit is for. Every read and write scoped from this (Principle X). |
| `clinic` FK · `doctor` FK · `slot` FK | Real FKs, not copied strings (`011`'s amendment, retained). **Plus a provider snapshot at booking time** — clinic name, doctor name, address and fee as they were when booked. Previously deferred; now required, because a directory row can legitimately change after a visit is booked and an appointment history that silently rewrites itself is wrong. |
| `service_date` · `period` · `period_starts_at` | denormalized from the chosen `slots` row. The lifecycle reads them constantly and they must survive the slot row changing. Cutoffs (cancel, reschedule, reminders, `lapsed`) are computed from `period_starts_at`, which replaces every former use of `period_starts_at`. |
| `token_no` | int — `booked_count` at issue time. Never renumbered, never carried across a reschedule. |
| `status` | `requested` · `booked` · `declined` · `cancelled` · `rescheduled` · `completed` · `no_show`, plus `lapsed` derived at read time. |
| `decline_reason` | nullable — includes `no_response` for an expired request window. |
| `cancelled_at` | nullable timestamptz |
| `cancelled_by` | nullable FK to the acting user — a caregiver cancelling a dependent's appointment must be attributable |
| `reschedule_count` | int, default 0, max 2 (FR-018) |
| `rescheduled_from` / `rescheduled_to` | nullable self-FKs forming the chain |
| `created_at` | timestamptz |

**`lapsed` is never stored.** It is derived at read time from `period_starts_at` + grace
against a `booked` status with no clinic outcome recorded.

**`payments` is a separate table** (§Payments) — profile-linked and scoped from `caregiver_grants`, deliberately not folded into `appointments.status`.

**No queue table.** Queue position comes from the clinic's feed (`now_serving`
for the session) compared against `token_no`. Nothing about the queue is
persisted on our side, and nothing is simulated where a real feed exists — see
§What this is / is NOT on the per-row rule for the word "live".

---

## API (`/api/v1/`)

Directory reads, carried from `011`:

- `GET /clinics?q=&specialty=` — search + filter, distance-ascending
- `GET /clinics/{id}` — B2's payload: clinic, lead doctor, facilities, FAQs, fee
- `GET /clinics/{id}/nearby` — the rail, 3–4 rows
- `GET /specialties` — chip source
- `GET /clinics/{id}/doctors` — B3's sheet, next-available derived

Amended:

- `GET /doctors/{id}/sessions?from=&days=5` — **returns every period session in
  the window with `capacity` and `booked_count`**, full ones included and
  flagged. Replaces `GET /doctors/{id}/slots`, which returned exact-time rows.
  Full sessions are returned rather than filtered for the same reason taken
  slots used to be (FR-012): omission reads as "closed".

New, appointment-scoped:

- `POST /appointments` — `{profile_id, slot_id}`. Server resolves clinic,
  doctor, `service_date` and `period` from the slot; the client never supplies
  them. Increments `booked_count` under a lock and assigns `token_no`. Returns
  `409` only when the session is **genuinely at capacity** (FR-015).
- `GET /profiles/{id}/appointments?status=` — B6 and the alerts strip
- `POST /appointments/{id}/cancel`
- `POST /appointments/{id}/reschedule` — `{slot_id}`. Atomically marks the old
  row `rescheduled`, creates the new row, and links both directions.

**Scoping (Principle X).** Every `appointments` endpoint derives its
queryset from the caller's own profiles ∪ unrevoked `caregiver_grants`. No
`objects.all()`, and no trusting a `profile_id` in the request body — it is
validated against the derived set, never accepted from it.

**Client-agnostic by rule (`011` FR-022, retained).** No field in any of these
responses may vary by which app asked. TDC Doctor calls the same endpoints TDC
Health calls. A different projection is a new endpoint, not a branch inside an
existing one. `source` and `external_ref` are internal and are never serialized
to clients.

---

## Seed ripple (`004`, same day per Workflow rule 5)

Carried from `011`, plus this spec's additions:

- **~10 bookable clinics.** The three existing consult facilities are reused
  (Sunrise Poly Clinic · Prabhat Multispecialty Hospital · Kavya Family Clinic);
  **Ashirwad Diagnostics is a lab and MUST NOT appear as a bookable clinic.**
  ~7 new fictional clinics still need Adi's real-name check before landing (see
  Open Decisions — this remains the one item that blocks the seed).
- **~10 specialties**, 4 primary chips (General Medicine, Cardiology,
  Pediatrics, Orthopedics) plus the *More* set. Include **Diabetology** —
  booking a diabetes follow-up for Asha is the seed's most natural demo path.
- **2–4 doctors per clinic**, one flagged `is_lead`. Dr. Kavya must exist as a
  real seeded row (she is already named in alerts-strip example copy).
- **Seeded coordinates** on neutral ground per FR-024.
- **Seeded ratings** in a believable band (4.2–4.9), not all identical.
- **Facilities and 3–5 FAQs per clinic**, generic and non-promotional.
- **Consult fees** in a believable band — **₹300–₹800**, varying by clinic and
  specialty. Identical fees across ten clinics read as placeholder data.
- **`cancellation_window_min` = 120** for every seeded clinic, with **one clinic
  seeded at 240** so the per-clinic path is actually exercised rather than
  merely present.
- **Slots generated relative to run date**, never fixed timestamps — a reseed on
  the morning of a demo must not produce yesterday's slots (`004` NFR-001).
  Cover the full 3-day window, with **a realistic proportion already booked**
  (~30%), so B4's struck-through state is visible without staging it.
- **`appointments` still seeds zero rows.** Booking live is the beat, and the
  flow is short enough that an empty state costs nothing. Reset returns
  everything to unbooked.

---

## Requirements

### Functional Requirements

**Entry & discovery**

- **FR-001**: Booking MUST be reachable only from a profile context — no Home
  entry, no nav entry, and no deep link that lands on any booking screen
  without a resolved profile.
- **FR-002**: Every screen from B1 to B6 MUST display the profile the booking
  is for, above the fold, at all times.
- **FR-003**: Search MUST match clinic names, doctor names and specialty names.
  It MUST NOT accept, suggest, or interpret symptoms; the placeholder MUST read
  "Search clinics or doctors".
- **FR-004**: All clinic, doctor and specialty names MUST be fictional, drawn
  from `004`'s facility set as extended by this spec's ripple.
- **FR-005**: A doctor-name search result MUST resolve both clinic and doctor
  and open B3 directly — never a dead end.
- **FR-006**: B2's Book Appointment CTA MUST remain visible regardless of
  scroll position.
- **FR-007**: B3 MUST NOT be skipped when a clinic has exactly one doctor.

**The booking spine**

- **FR-008**: The flow MUST be exactly **find care → clinic → doctor → slot →
  confirm**, one decision per screen, ending in an `appointments` row.
- **FR-009**: B4 MUST display who, where, with whom, when, the fee,
  and the cancellation window before the Confirm action — not after it, and not
  behind a link.
- **FR-010**: The system MUST NOT collect a reason for visit, a symptom, or any
  free-text clinical note **until the reason-for-visit field is specced** (§B5) — it is no longer prohibited, it is unbuilt.
- **FR-011**: Confirmation copy MUST read "Booking placed" and the status chip
  MUST read "Booked". The words "confirmed", "reserved", "held" and
  "guaranteed" MUST NOT appear in relation to a slot on any screen.
- **FR-012**: B4 MUST render every period the clinic runs on the selected day,
  with full periods shown **disabled and labelled** rather than omitted, and
  MUST NOT reflow as capacity changes. The same applies to the date strip: a
  fully-booked day is disabled, never hidden. Omission reads as "closed", which
  is a false impression the app created by leaving something out.
- **FR-013**: Period rows and date cells MUST meet the 48dp minimum touch
  target (`docs/design-tokens.md`; Principle VII). Period rows are full-width
  by design — the reference's three stacked rows are more legible at arm's
  length than the grid of time chips this replaced, which is a second reason
  the period model is the better fit.
- **FR-014**: The slot window MUST be the next 3 days. No further-out booking
  
- **FR-015**: `booked_count` MUST be incremented under a row lock or by an
  atomic DB-side increment guarded by `booked_count < capacity` — **never
  read-then-write**. A `409` is returned only when the session is genuinely
  full, surfaced as *"That session is full — pick another,"* returning to B4
  with fresh data. Two simultaneous bookings on a session with room MUST both
  succeed and receive distinct tokens.
- **FR-015a**: A token MUST be the value of `booked_count` at issue time, MUST
  NOT be renumbered when another booking is cancelled, and MUST NOT be carried
  across a reschedule. A token that changes after it was given out is worse
  than a gap in the sequence.
- **FR-015b**: The "Booking for" switcher on B4 MUST list only profiles the
  caller may act for (own ∪ unrevoked `caregiver_grants`) and MUST NOT offer
  profile creation. Creating a profile writes a `caregiver_grants` row — the
  consent moment (`001` FR-003, Principle II) — which MUST NOT occur inside a
  payment flow. Profile creation stays on Home's Add-family action
  (`001` FR-023).

**Lifecycle**

- **FR-043**: Visit History MUST render from `appointments` with
  `status = completed` — no separate visits entity. A clinical summary MUST be
  displayed only when the clinic wrote it, attributed, and MUST NOT be
  generated, inferred, or summarised by TDC.
- **FR-044**: Visit History copy MUST NOT imply a complete clinical history.
  It shows visits TDC booked; visits arranged elsewhere are absent, and the
  screen is named and worded so that absence is not read as "no visit".
- **FR-016**: An appointment MUST have exactly one stored status from
  `requested` · `booked` · `declined` · `cancelled` · `rescheduled` ·
  `completed` · `no_show`. **This app MUST NOT write `completed` or
  `no_show`** — both assert what only the clinic observed, and `no_show` is an
  accusation. They arrive from TDC Clinic or not at all.
- **FR-016a**: Confirming on B4 against a session the clinic has published with
  available capacity MUST create the appointment **directly in `booked`** — the
  published availability is the acceptance. The app MAY describe such an
  appointment as confirmed. Where capacity is a `DEMO_MODE` fixture or the
  provider has published nothing, the wording MUST remain "Booking placed".
- **FR-016b**: `requested` / `declined` / `response_window_min` apply **only to
  providers that require asynchronous confirmation** (UHI). They MUST NOT be
  used on the TDC Clinic path, where nothing is being waited on. A `requested`
  appointment on such a provider MUST NOT hang indefinitely.
- **FR-016e**: A clinic- or doctor-initiated cancellation MUST set `cancelled`,
  record which side acted, carry the clinic's reason through where given,
  release the slot, cancel both local reminders, clear the alerts-strip and
  day-of entries, write an `access_logs` row, and **refund any payment in full
  including the platform fee**.
- **FR-016f**: A clinic- or doctor-initiated cancellation MUST notify the
  patient **by push (FCM)**, not by a device-local schedule — the trigger is
  server-side and unpredictable. The notification MUST state who cancelled,
  when the appointment was, the reason where given, and offer a direct rebook
  action. This is the only server-push in this feature.
- **FR-016c**: A paid booking that ends `declined` (including `no_response`)
  MUST be refunded automatically and in full.
- **FR-016d**: Cancelling MUST propagate to the clinic. Releasing a slot only
  in the local cache leaves the clinic holding an appointment nobody attends.
- **FR-017**: `lapsed` MUST be derived at read time from `period_starts_at` plus a grace
  period **where no clinic outcome has been recorded**. It MUST NOT be stored,
  and MUST NOT be produced by a scheduled job or background sweep — a
  background writer mutating profile-owned rows outside a request context is an
  unauditable actor in a table whose purpose is naming the actor (Principle XI).
  A row aging in `lapsed` is a clinic-side reporting gap and MUST be visible as
  one.
- **FR-018**: Cancellation and reschedule MUST both be gated on
  `clinics.cancellation_window_min` before `period_starts_at`. Reschedule MUST be capped
  at 2 per booking and MUST keep the same doctor.
- **FR-019**: Cancellation MUST remain available **after** the cutoff. The copy
  changes; the ability does not. The system MUST NOT block, penalise, or
  restrict future booking on a late cancellation.
- **FR-020**: Reschedule MUST create a new `appointments` row, mark the old one
  `rescheduled`, link both directions, release the old slot, and assign a
  **new** token. It MUST NOT mutate the original row's slot.
- **FR-021**: Cancelling or rescheduling MUST cancel both scheduled reminders,
  and rescheduling MUST schedule two new ones against the new time. An orphaned
  reminder for a non-existent appointment is a defect.

**Reminders & Home**

- **FR-022**: Reminders MUST be device-local (`flutter_local_notifications`,
  reusing `006`'s primitive) at T−24h and T−2h, MUST deep-link to B6, and MUST
  NOT be scheduled in the past.
- **FR-023**: Queue-position notifications MUST be **opt-in and off by
  default**, MUST fire at most twice (`3 tokens away`, `Next`), and MUST NOT be
  sent for any session without a real clinic feed. Firing a queue event from a
  fixture fabricates a real-world occurrence and is prohibited (Principle VI).
- **FR-023a**: `tokens_before_you` MUST be derived from the session's live
  token list — the count of live tokens strictly between the current token and
  the patient's own, excluding cancelled and no-show tokens. It MUST NOT be
  computed as `your_token − current_token − 1`, which drifts on any
  cancellation and drifts toward telling a patient to arrive late.
- **FR-023b**: The estimated wait MUST be derived from `tokens_before_you` and
  the clinic's observed average consultation time for that session. Where no
  average exists, the estimate MUST be **omitted**, not defaulted.
- **FR-023c**: Any wait estimate MUST be shown with the disclaimer that it is
  an estimate that varies with consultation duration. The estimate MUST NOT be
  presented as a scheduled or guaranteed time.
- **FR-023d**: B6 MUST state what the queue is doing, not only where it is —
  `Queue Moving` · `Not started` · `Paused` · `Delayed` · `Closed`. A stalled
  number with no explanation is the failure this replaces.
- **FR-030**: An upcoming appointment MUST appear in the Home alerts strip only
  within 48 hours of `period_starts_at`, capped at two appointment rows across
  all profiles, never ranked above a low-stock or missed-dose alert, and MUST
  clear on cancellation or after the visit.
- **FR-030a**: **On the day of an appointment only**, Home MUST show a
  prominent **day-of entry** giving one tap to B6 (Queue Status). It MUST name
  the person and the session (*"Aai — Sunrise Poly Clinic, this morning · token
  28"*), MUST appear only while that appointment is live, and MUST disappear
  the moment it resolves. **Principle VIII check:** this is a resolving alert
  given more weight for one day, not a new content surface — it cannot be
  refreshed for novelty, carries no recommendation, links to exactly one
  destination the user already committed to, and is gone tomorrow. Bounded by
  FR-030's two-row cap, which it counts against rather than sitting beside.
- **FR-030b**: There MUST NOT be a bottom tab bar, a Clinics tab, or any
  booking entry point reachable without a profile in context (`008` navigation
  model; Principle VIII; extends FR-001).

**Honesty & compliance**

- **FR-024**: Seeded clinic coordinates MUST NOT fall on a real healthcare
  facility. Placing a fictional clinic's pin on a real hospital is a worse leak
  than a real name, because it implies an address as well as an identity.
  Neutral ground — a street segment, a commercial block — only.
- **FR-025**: Clinic and doctor imagery MUST NOT depict an identifiable real
  healthcare facility or a real practising clinician, and MUST NOT be sourced
  from photographs of one.
- **FR-026**: Wait, queue and availability values MUST be seeded and rendered
  with approximate phrasing. The words "live", "real-time", "now serving",
  "verified", "certified" and "partner" MUST NOT appear on any screen in this
  spec.
- **FR-027**: Clinic ratings MUST render as a rating alone — no review count,
  no review text, no Reviews tab, no "rated by" attribution, no sort-by-rating
  control (`011` decision 6 mitigation).
- **FR-028**: The doctor HID chip MUST carry no verification, certification,
  ABDM, HPR or registry copy, no tick or shield iconography, and no tap
  affordance (`011` decision 7 mitigation; "ABDM certified" remains a banned
  phrase).
- **FR-029**: Every `DEMO_MODE` fixture path MUST carry a `# DEMO-MODE` tag naming its real
  replacement: the seeded directory (→ UHI provider discovery) · seeded
  wait/queue values (→ TDC Clinic queue feed) · seeded ratings (→ real
  post-visit feedback) · seeded HID chip (→ HPR registry lookup) ·
  seeded session capacity (→ clinic-published `capacity`) · seeded consult
  fees (→ clinic-published fees) · derived `lapsed` (→ clinic-written
  `completed` / `no_show`).
- **FR-031**: The payment summary MUST itemise **consultation fee and platform
  fee separately with an explicit total** — never a single bundled figure. The
  platform fee MUST carry copy stating what it is for, MUST NOT be presented as
  a tax, a clinic charge, or a levy, and MUST NOT be labelled ambiguously
  (Principle VI applies to pricing claims).
- **FR-031a**: Card data MUST NOT touch TDC systems. **All card fields —
  number, expiry, and CVV — MUST be rendered by Razorpay's SDK or iframe, never
  by TDC UI**, keeping the project out of PCI-DSS scope. A CVV input drawn by
  our own code puts the app in SAQ A-EP scope and produces a WASA finding.
  Saved cards MUST be RBI-directive tokens held by Razorpay; TDC MUST NOT store
  or transmit a PAN. Payment-method copy MUST NOT imply an integration with any
  named UPI app — the integration is Razorpay; the app is the user's choice.
- **FR-031d**: Checkout copy MUST NOT display: cipher or protocol strengths
  (*"256-bit SSL"* — banned outright, `CLAUDE.md`; the approved word is
  **"encrypted"**), a certification TDC does not itself hold (*"PCI-DSS Level 1
  Certified"* is Razorpay's — *"Payments handled by Razorpay"* is the true
  form), or the words *verified* / *partner* applied to a clinic or to TDC
  (FR-026). The total shown on the pay button MUST equal the itemised total
  above it.
- **FR-031e**: Checkout MUST NOT carry cashback, discounts, or any offer tied
  to a named payment provider — a promotion and sponsored placement in one
  (FR-039), and a promise TDC would owe. No version ships without an owner
  decision.
- **FR-031b**: A payment failure MUST NOT release the token or destroy the
  appointment.
- **FR-031c**: A `declined` or `no_response` booking that was paid MUST be
  refunded automatically and in full, **platform fee included**.
- **FR-032**: No UHI API call. UHI may appear only in roadmap-pattern copy
  ("built for India's UHI network — integration in development").
- **FR-033**: The app MUST NOT request location permission. Distance MUST come
  from seeded `distance_km`, and the map MUST centre on the clinic.
- **FR-034**: The Directions action MUST hand off to the OS maps app without
  requesting the user's location, and MUST fail quietly to the address text if
  no maps app is available — not to an error dialog.

**Security & data**

- **FR-035**: Directory endpoints MUST expose no profile-linked data and MAY be
  read by any authenticated user. Every `appointments` endpoint — read and
  write — MUST derive its queryset from the caller's profiles ∪ unrevoked
  `caregiver_grants`, and MUST validate any supplied `profile_id` against that
  set rather than trusting it (Principle X).
- **FR-036**: Creating, rescheduling and cancelling an appointment MUST each
  write an `access_logs` row in the same transaction. Browsing the directory
  MUST NOT (Principle XI).
- **FR-037**: The four directory tables and `appointments` MUST live in TDC
  Core API. No client may hold a local directory table, or a cache presented as
  data. Responses MUST NOT vary by calling client, and `source` /
  `external_ref` MUST NOT be serialized.
- **FR-038**: `cancelled_by` MUST record the acting user, so that a caregiver
  cancelling a dependent's appointment is attributable.

**Scope & script**

- **FR-039**: Booking MUST NOT display written reviews, offers, promotions,
  sponsored placement, a See All browse destination, or any surface that
  refreshes to be re-checked.
- **FR-040**: Every screen MUST implement `008`'s four screen states, with
  empty states that name their recovery action.
- **FR-041**: `seed_demo.py` MUST seed the directory and zero appointments, and
  reset MUST clear anything booked during a demo.
- **FR-042**: Booking and discovery MUST NOT appear in the `001` golden-path
  script. The meetup script owns them (Soham).

### Non-Functional Requirements

- **NFR-001 (Tap count):** clinic card → Book Appointment → doctor → time →
  Confirm. Five taps from B1 to a booking, unchanged from `011`. Reschedule is
  three from B6.
- **NFR-002 (Reminder latency):** notification tap → B6 in under 10 seconds,
  with no more than one intermediate screen (`001` NFR-006).
- **NFR-003 (Determinism):** the queue simulation MUST be deterministic per
  seed so that rehearsals behave identically (`004` NFR-001 spirit).
- **NFR-004 (Queue pacing):** the served token MUST visibly advance inside a
  30-second show-and-tell without becoming absurd — one token per 8–12 seconds
  of screen time.
- **NFR-005 (Elderly-first):** `001` NFR-004 applies throughout. B4 is the
  screen most likely to fail it and MUST be checked at arm's length on a real
  device before the demo, not in a simulator.
- **NFR-006 (Build ceiling):** the whole feature must fit a solo-dev ~4h/day
  budget in the P1 window. Trim order in §Build placement is the mechanism.

---

## Copy guardrails

**Banned on every screen in this spec** (in addition to `001` and `CLAUDE.md`):

- "live", "real-time", "now serving" — no live feed exists.
- "confirmed", "reserved", "held", "guaranteed" applied to a slot.
- "verified", "certified", "partner", "ABDM certified" — no relationship exists.
- Any review count, "rated by N", or written review text.
- Any phrasing implying a payment will be, or has been, taken.
- Any no-show accusation, penalty warning, or strike language.

**Approved framing:**

- "Booking placed" · "Booked" · "Queue status" · "about 40 min" ·
  "~10 min wait" · "3 waiting" · "Pay at clinic" (no lock — it is a real
  option, not a locked one) ·
  "built for India's UHI network — integration in development".

---

## Carried forward

Nothing from `010` or `011` is dropped. Explicit mapping so the supersession is
auditable:

| Origin | Carried as |
|---|---|
| `010` FR-001 (profile-only entry) | FR-001 |
| `010` FR-002 / `011` FR-005 (flow order) | FR-008 |
| `010` FR-003 / `011` FR-004 (fictional only) | FR-004 |
| `010` FR-004 (pay at clinic, no simulation) | FR-031 |
| `010` FR-005 (alerts strip) | FR-030, now bounded |
| `010` FR-006 (deterministic queue, no live claim) | FR-026, NFR-003 |
| `010` FR-007 / `011` FR-021 (no UHI) | FR-032 |
| `010` FR-008 (zero seeded appointments) | FR-041 |
| `010` FR-009 / `011` FR-020 (not in golden path) | FR-042 |
| `011` FR-001/002 (profile context) | FR-001, FR-002 |
| `011` FR-003 (no symptom search) | FR-003 |
| `011` FR-006 (sticky CTA) | FR-006 |
| `011` FR-007 (doctor search resolves) | FR-005 |
| `011` FR-008 (seeded values, banned words) | FR-026 |
| `011` FR-009/012 (no geo permission, directions) | FR-033, FR-034 |
| `011` FR-010/011 (imagery, coordinates) | FR-025, FR-024 |
| `011` FR-013/014 (rating, HID mitigations) | FR-027, FR-028 |
| `011` FR-015/016 (scoping, access logs) | FR-035, FR-036 |
| `011` FR-017 (DEMO-MODE tags) | FR-029, expanded 4 → 7 |
| `011` FR-018 (no reviews/offers/See All) | FR-039 |
| `011` FR-019 (four screen states) | FR-040 |
| `011` FR-022 (Core ownership, client-agnostic) | FR-037 |
| `011` FR-023 (`is_booked` DEMO-MODE) | FR-029 |

`011`'s eight decisions of 20 Aug — including the three owner overrides
(ratings, HID chip, real map + Directions) — stand unchanged. `010`'s unlock
provenance stands unchanged: if the `CLAUDE.md` changelog entry of 1 Aug 2026
is ever reverted, this spec reverts with it.

---

## Build placement

P1, after P0 is demo-ready. Estimated **~4–5 days** at 4h/day: `011`'s
discovery and seed work (~2–2.5) plus `010`'s slot/confirm/queue (~1.5–2) plus
this spec's lifecycle and reminders (~1). The maps integration remains the
single largest line item and the most likely thing to overrun.

**Pause trigger unchanged:** if P0 is not demo-ready, this pauses entirely.
**Slip rule unchanged:** never cut the emergency flow to fund this.

**Trim order** if it must be reduced rather than paused:

1. FAQ tab
2. Nearby-clinics rail
3. Map (falls back to address text + Directions)
4. Reschedule (cancel-and-rebook covers it, at worse UX)
5. T−24h reminder (keep T−2h)
6. Clinic page (B2 collapses into B3's doctor list)

Cutting below step 6 means cutting the feature; do that instead of shipping a
booking flow with no clinic context.

---

## Open Decisions

| Decision | Owner | Notes |
|---|---|---|
| ~~The ~7 fictional clinic names~~ | Adi | **Closed 6 Sep 2026 — names land as-is.** Gulmohar Health Centre · Shantiniketan Family Clinic · Nisarg Multispecialty · Anandvan Child Care · Chandrakala Heart Care · Tulip Poly Clinic · Riverside Family Clinic. The **avoid list stays binding** for anything added later: Ruby Hall, Sahyadri, Jehangir, Deenanath Mangeshkar, Noble, Poona Hospital, Sancheti, Inamdar, Aditya Birla. *Residual risk, recorded not resolved:* accepted on judgement rather than a register check, so a collision with a small real Pune practice is possible. A name collision is fixed by editing one seed row; **FR-024 (coordinates) and FR-025 (imagery) are the two that cannot be undone by a rename**, and neither is closed by this decision. |
| **Maps provider + key handling** | Adi | Google Maps SDK vs. a lighter tile provider. An API key in a demo APK is a key that leaks — restrict by package ID + signing fingerprint before any build ships. |
| **Clinic and doctor imagery** | Adi | Illustration or abstract tiles remove FR-025's leak vector entirely and cost less than sourcing safe photography. Doctor photos are the higher risk: a stock portrait of a real person presented as a named doctor. |
| **Specialty chip icon set** | Adi | Complete set required before demo (FR-003's chip row). |
| **Grace period before `lapsed`** | Adi | 4h assumed. Long enough that a running-late appointment doesn't grey out mid-visit; short enough that yesterday reads as past. |
| **Reschedule cap** | Adi | 2 assumed. Arbitrary but bounded; revisit only if a demo hits it. |
| **Does the platform fee apply to a pay-at-clinic booking?** | Adi | TDC collects nothing else on that path, so either the fee is charged online while the consultation is paid at the counter (odd but defensible — it is a coordination fee, not a consultation fee), or pay-at-clinic bookings are free to make. Affects the revenue model more than the UI. |
| **Pay-at-clinic: keep at all?** | Adi | Specced as `clinics.accepts_pay_at_clinic` — my call, since the 9 Sep multi-select confirmed the platform fee but settled neither payment option. The reference shows prepay-only. Confirm or overrule. |
| **Session capacity source** | Adi | `capacity` is the clinic's number. Until TDC Clinic publishes it, seeded values stand in — a `DEMO_MODE` fixture with `tdc_clinic` as its named real path, no external gate. |
| **Payment-provider cashback / offers** | Adi | The reference shows *"Flat ₹50 cashback via CRED UPI & PhonePe"*. Rejected as specced (FR-031e) — promotion + sponsored placement, and a liability. If wanted, it needs a funding source, a T&C, and an exemption argued against Principle VIII. |
| **"Instant 100% refund if the doctor misses the slot"** | Adi | A real trust signal and a real commercial guarantee. Needs: who funds it, what counts as "missed", and whether "instant" survives a bank settlement window (Principle VI — the words are literal claims). Not specced until decided. |
| **Teleconsult** | Adi | The reference books a *telehealth* consultation with a "Live Session" badge. Out of scope here — no token, no queue, no Directions. Not prohibited; needs its own spec. |
| **Pay Later (Simpl / LazyPay)** | Adi | Razorpay offers it, but it is **consumer credit attached to a health bill**. Different regulatory and ethical surface from paying for a consult. Default: off until argued. |
| **Meetup script beat order** | Soham | Card first or booking first? Lean card — it's the differentiator; booking answers the follow-up. |
| **Whether the script pauses on B2** | Soham | The clinic page invites browsing mid-demo. Decide: linger, or drive through to the token. |

---

## Downstream edits this spec requires

**All applied 6 Sep 2026** except `seed_demo.py`, which does not exist yet.

| File | Change |
|---|---|
| `010-consult-booking-queue/spec.md` | Supersession notice at the top; status → Superseded by `012`. Keep the unlock record intact — it is the provenance. |
| `011-care-discovery/spec.md` | Supersession notice; status → Superseded by `012`. Keep the eight decisions and the three overrides — they are the decision record `012` inherits. |
| `008-navigation-app-shell/spec.md` | Route map: add `/appointment/:apptId/reschedule`; retag the `/book` sub-tree from `010`/`011` to `012`; **also remove the stale `/who-for` route, deleted by `002`'s rewrite.** |
| `004-seed-data-and-summary/spec.md` | Seed ripple: consult fees, `cancellation_window_min`, ~30% pre-booked slots. |
| `006-medications-reminders-adherence/spec.md` | Note that appointment reminders reuse the same local-notification primitive, so scheduling and cancellation stay one code path. |
| `001-tdc-phr-patient-app/spec.md` | §4.1 Loop 2: alerts strip gains a bounded appointment row (FR-030). |
| `health/CLAUDE.md` | Data model: `appointments` rewrite, two new `clinics` columns. Screens list. Changelog entry recording the `010`+`011` → `012` merge. |
| `012/checklists/security.md` | New — Principles X / XI / XIV, carrying `011`'s checklist forward plus the appointment write path, notification-as-PHI surface, and concurrency section. |
| `docs/repo-structure.md` | Pointer only — the 20 Aug narrative names `011` as the directory owner; noted that `012` now owns the flow. |
| `seed_demo.py` | **Not applicable yet — the file does not exist.** Workflow rule 5 (same-day seed update on any model change) attaches when it is created; `004` §Booking lifecycle seed carries the contract until then. |

---

## Review & Acceptance Checklist

### Content quality
- [x] Whole flow owned by one spec; the `010`/`011` ownership seam removed.
- [x] Every `010` and `011` FR mapped forward, none silently dropped.
- [x] Screens specified to build detail, including the two `010` described in one line each.
- [x] Divergences from the reference screenshots retained with reasons.

### Requirement completeness
- [x] Fences restated as FRs (no UHI, no payments, no real clinics, no symptom search, no live claims, no geo permission).
- [x] Lifecycle states each name their transition owner; `completed` / `no_show` are clinic-written and never asserted by this app.
- [x] Cancellation and reschedule have policies, not just verbs.
- [x] Reminder cancellation specified — no orphaned notifications.
- [x] Slot race condition handled server-side (FR-015).
- [x] Payments shaped for Phase 2 without shipping anything that collects.
- [x] `# DEMO-MODE` tags enumerated (7).
- [x] Trim order defined, not just a pause trigger.
- [x] Fictional clinic names resolved (owner decision, 6 Sep — land as-is, residual risk recorded).
- [ ] Maps provider chosen and key restricted.
- [ ] `access_logs` write design closed in `001` — **blocks the appointment-write tasks** (`checklists/security.md`, Audit #1).
- [ ] Slot-race mechanism chosen — row lock or unique constraint, never read-then-write (`checklists/security.md`, Concurrency).

## Execution Status
- [x] Owner clarifications recorded (2026-09-06)
- [x] Flow, screens, lifecycle, model, API, requirements defined
- [x] Carry-forward mapping complete
- [x] `checklists/security.md` created — 1 blocking GAP (carried from `001`), 7 non-blocking
- [x] Supersession notices applied to `010` and `011`
- [x] Ripples applied — `008` (routes, deep links, dead-link rule, stale `/who-for` removed) · `004` (fee/window/pre-booked seed, names unblocked) · `006` (shared-primitive note) · `001` (bounded alerts row) · `CLAUDE.md` (model, API, scope, changelog)
- [x] `seed_demo.py` — **N/A, no code exists yet.** The same-day rule (Workflow rule 5) attaches when the file is created; `004` carries the contract until then.
- [ ] `flowchart.md` drawn
- [ ] `plan.md` / `tasks.md` — not started (`/speckit.plan` is the next step)
