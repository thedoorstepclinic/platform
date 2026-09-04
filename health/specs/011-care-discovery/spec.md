# Feature Specification: Care Discovery (Track A — Meetup Loop, seeded directory)

**Feature Branch:** `011-care-discovery`
**Created:** 2026-08-20
**Status:** Draft
**Owner:** Adi (dev) / Soham (meetup script)
**Decision record:** owner decisions 20 Aug 2026 — discovery specced as **step 1
of the booking flow**, not as a Home surface or a nav destination. Eight
decisions were taken explicitly and are listed under *Decisions taken*; three of
them are owner overrides of a concern raised during specification and are marked
as such. If any is reverted, the affected sections revert with it.
**Depends on:** [`010`](../010-consult-booking-queue/spec.md) (booking flow this
feeds; amended by this spec) · [`008`](../008-navigation-app-shell/spec.md)
(routes, sheet conventions, four screen states) ·
[`004`](../004-seed-data-and-summary/spec.md) (fictional-facility rule, seed
ripple) · [`001`](../001-tdc-phr-patient-app/spec.md) (P1 slot, scope fences).

## Why this exists

`010` unlocked consult booking with a **seeded list of 3 clinics** — enough for a
3-tap beat, not enough to answer the question a stranger at a meetup actually
asks, which is *"can I find a doctor near me, and is this place any good?"*
Booking without discovery is a form; booking with discovery is a product. This
spec adds the find-care step, the clinic information page, and the directory
behind both.

Same two-audience frame as `010`: this is a **meetup-loop** beat. The investor
golden path (`001`) still contains neither booking nor discovery.

## Decisions taken (20 Aug 2026)

1. **Track A, seeded.** No UHI calls, no real provider directory.
2. **Discovery lives inside the booking flow**, entered from a profile. Home is
   untouched — Principle VIII is preserved without amendment, not argued around.
3. **The reference screenshots are the target layout** (find-care list + clinic
   information page), rethemed to TDC tokens.
4. **No symptom search.** Rejected as medical inference — see fences.
5. **Flow gains a clinic page and a doctor step:** find care → clinic → doctor →
   slot → confirm. This amends `010` FR-002 (was clinic → slot → confirm).
6. **⚠ Override — ratings kept.** Each seeded clinic carries a dummy star
   rating. Review *counts* and the Reviews tab are dropped; only the rating
   renders. Concern raised (fabricated social proof on fictional clinics,
   Principle VI/VIII); owner decided to keep. Mitigation in FR-013.
7. **⚠ Override — HID chip kept.** The doctor block shows a seeded masked
   provider ID. Concern raised (implies an ABDM/HPR provider-registry lookup we
   do not perform; adjacent to the banned "ABDM certified" claim); owner decided
   to keep. Mitigation in FR-014 — the chip must never carry verification copy.
8. **⚠ Override — real map + Directions.** Each fictional clinic gets seeded
   real coordinates; the location block renders an interactive map and a working
   Directions hand-off. Concern raised (adds a maps SDK + API key to the demo
   build, and pins a fake clinic at a real place); owner decided to keep.
   Mitigation in FR-011 — coordinates must not land on a real healthcare
   facility.

## What this is / is NOT

**IS:** A seeded provider directory (~10 fictional clinics, their doctors,
specialties, facilities, FAQs and slots), a find-care screen over it, and a
clinic information page. Everything renders from seed data via ordinary API
queries.

**IS NOT — hard fences:**
- **No symptom search.** The reference's "Search clinics, doctors, or symptoms"
  becomes **"Search clinics or doctors"**. Symptom → specialty inference is
  medical inference: it needs a curated mapping, a not-medical-advice
  disclaimer, and a compliance pass Track A has no room for. Revisit in Track B.
- **No written reviews.** Ratings render (decision 6); review text, review
  counts and the Reviews tab do not.
- **No live wait/queue data.** Wait and queue values are seeded; copy never
  claims otherwise (Principle VI — the same rule `010` FR-006 applies to its
  queue view).
- **No real clinics.** `004`'s fictional-facility rule, zero exceptions. This is
  the highest-risk surface in the app for a real-name leak, because a directory
  *wants* to be filled with names that sound real.
- **No location permission.** Distances are seeded. The map centres on the
  clinic, not on the user.
- **No payments, no teleconsult, no favourites, no offers, no sponsored
  placement, no insurance filters, no "See All" browse destination.**

## Constitution check

**VIII (Home Stays Quiet) — preserved, not amended.** Discovery is reachable only
from a profile's action row (`Book consultation`, beside Health Summary). No Home
entry, no nav slot, no tab. Home's only relationship to this feature remains the
one `010` established: an upcoming appointment appears in the alerts strip and
resolves after the visit. A stranger cannot reach a clinic list without first
choosing *who it is for* — which is the positioning: TDC coordinates care for a
person, it does not browse inventory. The "Other clinics nearby" rail on the
clinic page is bounded navigation *within* a chosen flow (no See All, no
recommendation score, seeded-distance order only), not a destination.

**VI (claims must be literally true).** No badge, label or copy on any discovery
surface may imply partnership, verification, certification, or a live feed.
Wait/queue chips use approximate phrasing. The words "live", "real-time",
"now serving", "verified", "certified" and "partner" are prohibited on these
screens. Decisions 6 and 7 sit closest to this line; FR-013 and FR-014 are what
keep them on the right side of it.

**IX (one snapshot, never a parallel template).** Not triggered — these are
primary surfaces, not previews of another party's view.

**X `[A]` (authorization scoped server-side).** The directory (clinics, doctors,
specialties, slots) is **non-PHI reference data**, readable by any authenticated
user — an intentional unscoped read, recorded as such in `checklists/security.md`.
The *write* it leads to (`appointments`) is profile-scoped from `caregiver_grants`
like every other profile-owned resource.

**XI `[A]` (access logging).** Browsing the directory logs nothing — no PHI is
read. Confirming a booking writes an `access_logs` row.

**XIV `[A]` (`# DEMO-MODE` tags name their Track B replacement).** Four tags
required: the seeded directory (→ UHI provider discovery), the seeded wait/queue
values (→ TDC Clinic queue feed), the seeded ratings (→ real post-visit
feedback), and the seeded HID chip (→ HPR provider-registry lookup).

---

## Screens & flow

`011` owns D1–D3. `010` owns the slot grid, confirm, appointment detail and queue
view. Ownership is stated here so neither spec silently grows into the other.

### D1 — Find care (`/profile/:profileId/book`)

Rethemed to `docs/design-tokens.md` (`primary #4B83F2`, warm neutrals, large type
/ arm's-length legibility). Top to bottom:

| Element | Behaviour |
|---|---|
| **Profile context header** | *"Booking for Aai (Asha K.)"* + avatar. Replaces the reference's hamburger/brand/city header. Non-negotiable: it keeps entry "through the person" and prevents the wrong-patient booking error class. |
| **Area label** | Static seeded label (`Pune`) — not a picker, no GPS. Track B makes it a real location control. |
| **Search field** | Placeholder **"Search clinics or doctors"**. Matches clinic name, doctor name and specialty name. Debounced server query. |
| **Specialty chips** | Icon + label chips from `specialties`, 4 primary visible + **More** to expand. Tap filters, tap again clears. |
| **Nearby clinics list** | Card per clinic: image, name, `2.5 km away`, `~10 min wait`, `3 waiting`, chevron. Sorted by seeded distance ascending. |

### D2 — Clinic information page (`/profile/:profileId/book/:clinicId`)

The second reference screenshot, rethemed. Top to bottom:

| Element | Behaviour |
|---|---|
| **App bar** | Back + clinic name. **Share icon dropped** — sharing a fictional clinic page has no destination in Track A. |
| **Hero image** | Seeded clinic image (FR-010 governs sourcing). |
| **Identity card** | Clinic name · seeded star rating, no review count · lead doctor: name, `Senior Cardiologist • 15+ years`, seeded masked `HID: ****8821` chip. |
| **Location** | Interactive map centred on the clinic's seeded coordinates · fictional Pune address · **Directions** hands off to the OS maps app. |
| **Other clinics nearby** | Horizontal rail, 3–4 cards, seeded-distance order. **No "See All."** Tapping replaces the current clinic page. |
| **Tabs** | **Facilities** (icon tiles: parking, wheelchair access, pharmacy, wi-fi …) and **FAQ** (seeded Q&A). **No Reviews tab.** |
| **Sticky CTA** | **Book Appointment** — opens D3. Always visible, never scrolls away. |

### D3 — Choose doctor (`/profile/:profileId/book/:clinicId/doctors`) — sheet

Per `008`'s sheet convention for quick actions. One row per doctor: name,
specialty, qualification, next-available label (derived from `slots`, not
stored). One decision, one tap. A doctor-name hit in D1's search opens this sheet
with that doctor highlighted, clinic already resolved.

### Then `010` takes over

`/book/:clinicId/:doctorId/slots` (slot grid) →
`/book/:clinicId/:doctorId/confirm` (summary card, token, 🔒 Pay at clinic,
Confirm) → `appointments` row → appointment detail + seeded queue view.

Demo tap count: clinic card → Book Appointment → doctor → slot → Confirm.

### Screen states

All three screens implement `008`'s four states: skeleton cards while loading ·
*"No clinics match that."* with a Clear-search action when empty · plain-language
error + Retry · no offline pretence.

### Divergences from the reference screenshots (deliberate)

| Reference | TDC | Why |
|---|---|---|
| App-level header: hamburger, brand, city dropdown, account | Profile-context header: who this booking is for | Principle VIII — entry through the person, not a services surface |
| "…clinics, doctors, or **symptoms**" | "Search clinics or doctors" | Symptom search rejected (decision 4) |
| Filter icon in the search field | Not built | Specialty chips already are the filter; one decision per screen |
| Bare `10 mins wait` / `3 in queue` | `~10 min wait` / `3 waiting` | Principle VI — must not read as a live feed |
| `4.8 (124 Reviews)` + Reviews tab | Rating only; no count, no tab | Decision 6 — rating kept, fabricated review corpus not |
| "Similar Clinics & Doctors · **See All**" | "Other clinics nearby", no See All | A See All destination is where a browse feed starts |
| Share icon in the app bar | Dropped | Nothing to share to in Track A |
| `123 Wellness Blvd, Medical District, NY` | Fictional Pune address | Seed set is Pune-based (`004`) |
| Orthopedics chip rendering as `H4` | Complete icon set required before demo | The reference ships a visible missing-glyph fallback; ours must not |

---

## Ownership & placement (20 Aug 2026 decision)

**All four tables live in TDC Core API** (`platform/services/core-api/`), in a
`directory` Django app. They are not in the Flutter client, and they are
explicitly **not** in TDC Doctor — that app is a Flutter client with no
database, so "put the doctor table in the doctors app" is not a placement this
architecture has. Every client consumes the same directory endpoints in the
same shape; see `docs/repo-structure.md` → *Every app is a client*.

This decision moved the services out of `health/`: a service the doctor app
calls cannot sit inside the patient app's product directory.

### Track B: Core aggregates, one API out

Track A seeds the directory. Track B makes Core the **aggregation point**, not
the sole author — it ingests TDC's own facilities from TDC Clinic (the CARE
fork) and external providers from UHI / HFR / HPR, and still serves one API to
every client. Clients never learn where a row came from; that opacity is what
keeps them clients.

Two columns exist on `clinics` and `doctors` from day one to make that
possible without a rewrite:

- **`source`** — `seed` | `tdc_clinic` | `uhi`. Track A writes only `seed`,
  `# DEMO-MODE` tagged per Principle XIV `[A]`.
- **`external_ref`** — nullable; the HFR facility id or HPR professional id in
  Track B, null in Track A.

`external_ref` is what makes the HID chip (decision 7) coherent rather than
arbitrary: today `hid_masked` is an invented string, and in Track B it becomes
a masked projection of the HPR `external_ref`. The chip survives the
transition instead of being deleted in it — which is the argument for keeping
it that the override was missing.

### The one thing Track A must not pretend it owns

**`slots.is_booked` is a demo simplification and must be tagged as one.**
Availability is genuinely owned by the clinic's HMS. In Track A, confirming a
booking flips a local flag. In Track B, booking is a **request to TDC Clinic or
UHI that can be declined**, and a patient-side availability table that thinks
it is authoritative is actively wrong — it will happily confirm a slot the
clinic already filled. Nothing in Track A may be built on the assumption that
a local flip is a confirmation: no "confirmed" language beyond what `010`'s
seeded flow already shows, and no logic that treats `is_booked = false` as a
guarantee.

## Data model

Four new tables plus an amendment. All new tables are **reference data, not
PHI**, owned by Core and served to every client.

**`specialties`** — id · name · icon_ref · sort_order · `is_primary` (the 4 shown
before *More*).

**`clinics`** — id · `source` · `external_ref` (nullable) · name (fictional) ·
`address_line` · area · city ·
`lat` / `lng` (seeded, FR-011) · `distance_km` (seeded static) · `hero_image_ref`
· `rating` (seeded decimal, FR-013) · `typical_wait_min` (seeded) · `queue_count`
(seeded) · `facility_tags` (string array — parking, wheelchair, pharmacy, wifi,
lift, lab) · `faqs` (JSON: question/answer pairs) · active.

**`doctors`** — id · `source` · `external_ref` (nullable — HPR id in Track B) ·
clinic FK · name (fictional) · specialty FK · qualification ·
`experience_years` · `hid_masked` (seeded string in Track A, masked
`external_ref` in Track B — FR-014) · `is_lead` (surfaced on D2's identity
card) · photo_ref · active.

**`slots`** — id · doctor FK · slot_ts (timestamptz) · is_booked *(Track A
demo simplification — see* The one thing Track A must not pretend it owns
*above)*.

**`appointments` (amends `010`)** — the `clinic_name` / `doctor_name` /
`specialty` copied strings are replaced by `clinic` FK · `doctor` FK · `slot` FK.
Rationale: with real tables the copies are pure drift risk. **Track B note:** a
production appointment SHOULD snapshot provider details at booking time, since a
directory row can legitimately change after a visit is booked; Track A does not,
and this note is the record of why that is a deliberate simplification and not an
oversight.

No queue table (unchanged from `010` — the queue stays a deterministic
client-side simulation off `token_no` + `slot_ts`).

## API (`/api/v1/`)

- `GET /clinics?q=&specialty=` — search + filter, distance-ascending.
- `GET /clinics/{id}` — D2's payload: clinic, lead doctor, facilities, FAQs.
- `GET /clinics/{id}/nearby` — the rail, 3–4 rows, seeded-distance order.
- `GET /specialties` — chip source.
- `GET /clinics/{id}/doctors` — D3's sheet, next-available derived.
- `GET /doctors/{id}/slots` — next 3 days, unbooked only (`010`'s grid).

All directory endpoints are readable by any authenticated user and unscoped by
design. Booking itself remains `010`'s endpoint, now taking slot/doctor/clinic
ids.

**Client-agnostic by rule:** no field in these responses may vary by which app
asked. TDC Doctor calls the same `GET /api/v1/clinics` TDC Health calls. If the
doctor app needs a different projection, that is a new endpoint, not a branch
inside an existing one. `source` and `external_ref` are internal — they are
**not** serialized to clients.

## Seed ripple (`004`, same-day per Workflow rule)

- **~10 bookable clinics.** The three existing consult facilities are reused
  (Sunrise Poly Clinic · Prabhat Multispecialty Hospital · Kavya Family Clinic);
  **Ashirwad Diagnostics is a lab and MUST NOT appear as a bookable clinic.**
  ~7 new fictional clinics are needed — see *Open Decisions*; they require Adi's
  local check before landing.
- **~10 specialties**, the 4 primary chips matching the reference (General
  Medicine, Cardiology, Pediatrics, Orthopedics) plus the *More* set. Include
  **Diabetology** — booking a diabetes follow-up for Asha is the seed's most
  natural demo path and costs nothing to enable.
- **2–4 doctors per clinic**, one flagged `is_lead`. Dr. Kavya (already named in
  `010`'s alerts-strip example copy) must exist as a real seeded row.
- **Seeded coordinates** per clinic, placed on neutral ground per FR-011.
- **Seeded ratings** in a believable band (4.2–4.9), not all identical.
- **Facilities and 3–5 FAQs per clinic**, generic and non-promotional.
- **Slots generated relative to run date**, never fixed timestamps — a reseed on
  the morning of a demo must not produce yesterday's slots (`004` NFR-001).
- **`appointments` still seeds zero rows** (`010` FR-008 unchanged) — booking
  live is the beat. Directory tables *are* seeded; they are inventory, not user
  data.

---

## Requirements

- **FR-001**: Discovery MUST be reachable only from a profile context. No Home
  entry, no nav entry, no deep link that lands on discovery without a profile.
- **FR-002**: The find-care screen MUST display the profile the booking is for,
  above the fold, at all times.
- **FR-003**: Search MUST match clinic names, doctor names and specialty names.
  It MUST NOT accept, suggest or interpret symptoms; the placeholder MUST read
  "Search clinics or doctors".
- **FR-004**: All clinic, doctor and specialty names MUST be fictional, drawn
  from `004`'s facility set as extended by this spec's ripple.
- **FR-005**: The full flow MUST be find care → clinic → doctor → slot → confirm,
  one decision per screen, ending in an `appointments` row (amends `010` FR-002).
- **FR-006**: The clinic page's Book Appointment CTA MUST remain visible
  regardless of scroll position.
- **FR-007**: A doctor-name search result MUST resolve clinic and doctor and open
  the doctor sheet directly — never a dead end.
- **FR-008**: Wait and queue values MUST be seeded and rendered with approximate
  phrasing. The words "live", "real-time", "now serving", "verified",
  "certified" and "partner" MUST NOT appear on any discovery surface.
- **FR-009**: The app MUST NOT request location permission. Distance MUST come
  from the seeded `distance_km` field; the map MUST centre on the clinic, never
  on the user.
- **FR-010**: Clinic and doctor imagery MUST NOT depict an identifiable real
  healthcare facility or a real practising clinician, and MUST NOT be sourced
  from photographs of one.
- **FR-011**: Seeded clinic coordinates MUST NOT fall on a real healthcare
  facility. Placing a fictional clinic's pin on a real hospital is a worse leak
  than a real name, because it implies an address as well as an identity. Neutral
  ground (a street segment, a commercial block) only.
- **FR-012**: The Directions action MUST hand off to the OS maps app and MUST NOT
  request the user's location to do so. If no maps app is available it MUST fail
  quietly to the address text, not to an error dialog.
- **FR-013**: Clinic ratings are seeded values and MUST render as a rating alone
  — no review count, no review text, no Reviews tab, no "rated by" attribution,
  and no sort-by-rating control. *(Owner override, decision 6: implementing this
  as a rating-only surface is the mitigation that keeps a fabricated number from
  becoming a fabricated corpus.)*
- **FR-014**: The doctor HID chip is a seeded masked string and MUST carry no
  verification, certification, ABDM, HPR or registry copy, no tick/shield
  iconography, and no tap affordance. *(Owner override, decision 7: the chip may
  exist; a claim about what it proves may not — "ABDM certified" remains a banned
  phrase.)*
- **FR-015**: Directory endpoints MUST expose no profile-linked data and MUST be
  readable by any authenticated user; the booking write MUST be scoped from
  `caregiver_grants` (Principle X `[A]`).
- **FR-016**: Confirming a booking MUST emit an `access_logs` row; browsing the
  directory MUST NOT (Principle XI `[A]`).
- **FR-017**: The seeded directory, seeded wait/queue values, seeded ratings and
  seeded HID chip MUST each carry a `# DEMO-MODE` tag naming their Track B
  replacement (Principle XIV `[A]`).
- **FR-022**: The four directory tables MUST live in TDC Core API. No client —
  TDC Health, TDC Doctor, or any future one — may hold a local directory table
  or a cache presented as data. Directory responses MUST NOT vary by calling
  client, and `source` / `external_ref` MUST NOT be serialized to clients.
- **FR-023**: `slots.is_booked` MUST carry a `# DEMO-MODE` tag naming its Track
  B replacement (a booking request to TDC Clinic / UHI that may be declined).
  No Track A code may treat a local flag flip as a confirmed reservation.
- **FR-018**: Discovery MUST NOT display written reviews, offers, promotions,
  sponsored placement, a See All browse destination, or any surface that
  refreshes to be re-checked.
- **FR-019**: Every discovery screen MUST implement `008`'s four screen states,
  with an empty-search state that names the recovery action.
- **FR-020**: Discovery MUST NOT appear in the `001` golden-path script (extends
  `010` FR-009 — the meetup script owns it).
- **FR-021**: No UHI API call. UHI may appear only in roadmap-pattern copy
  (`010` FR-007 unchanged).

## Build placement

P1, immediately before `010`'s booking screens (it is now step 1 of that flow).
Estimated ~2–2.5 days on top of `010`'s 1.5–2 — the seed set and the clinic page
carry most of that, and the maps integration (decision 8) is the single largest
line item and the most likely thing to overrun. **Pause trigger unchanged:** if
P0 is not demo-ready, `010` pauses and this pauses with it. Slip rule unchanged —
never cut the emergency flow to fund this. If this feature must be trimmed rather
than paused, cut in this order: FAQ tab → nearby rail → map (falls back to
address text) → clinic page (D2 collapses into D3's doctor list).

## Open Decisions

| Decision | Owner | Notes |
|---|---|---|
| The ~7 new fictional clinic names | Adi | **Must be checked against real Pune facilities before landing.** Candidates to verify: Gulmohar Health Centre · Shantiniketan Family Clinic · Nisarg Multispecialty · Anandvan Child Care · Chandrakala Heart Care · Tulip Poly Clinic · Riverside Family Clinic. Known-real names to avoid outright: Ruby Hall, Sahyadri, Jehangir, Deenanath Mangeshkar, Noble, Poona Hospital, Sancheti, Inamdar, Aditya Birla. |
| Does core-api get its own git repo? | Adi | You said "the core api has its own repo" while agreeing to the `platform/services/core-api/` path. The path is applied; the repo split is **not**, and I'd hold it for Track B. `docs/repo-structure.md`'s whole argument for a monorepo is that a data-model change ripples Core → Fastlane snapshot → seed script → app screen in one sitting — and core-api is the epicentre of exactly that ripple. Splitting it out is the one split that costs the most during a 4h/day demo build. Revisit when TDC Doctor needs an independent release cadence. |
| Maps provider + key handling | Adi | Google Maps SDK vs. a lighter tile provider. An API key in a demo APK is a key that leaks; restrict it by package ID + fingerprint before the build ships. |
| Clinic and doctor imagery | Adi | Illustration or abstract tiles remove FR-010's leak vector entirely and are cheaper than sourcing safe photography. Doctor photos are the higher risk of the two — a stock portrait of a real person presented as a named doctor. |
| Whether D3 is skipped for single-doctor clinics | Adi | Auto-advancing is smoother but breaks predictability mid-demo. Lean: never skip. |
| Specialty chip icon set | Adi | The reference ships a missing-glyph fallback (`H4` for Orthopedics); ours must not. |

## Review & Acceptance Checklist
- [x] Decision provenance recorded (20 Aug 2026), with the three owner overrides
      marked and each given a written mitigation rather than a silent pass.
- [x] Principle VIII preserved without amendment; the argument is made, not assumed.
- [x] Hard fences restated as FRs (no symptom search, no reviews, no geo
      permission, no live claims, no real clinics, no See All).
- [x] Divergences from the reference designs listed with reasons.
- [x] Model change + seed ripple identified same-day.
- [x] Ownership split with `010` stated explicitly.
- [x] Real-facility leak treated as the top risk, now covering names, imagery
      **and** coordinates.
- [x] Trim order defined, not just a pause trigger.
- [ ] Fictional clinic names verified by Adi (blocks seed landing).

## Execution Status
- [x] Owner decisions recorded
- [x] Flow, model, requirements defined
- [x] `checklists/security.md` created (Principles X/XI/XIV)
- [x] Ripples applied (`CLAUDE.md`, `010`, `008`, `004`)
- [ ] Clinic name verification returned
- [ ] Maps provider chosen and key restricted
