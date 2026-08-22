# Feature Specification: TDC PHR Patient App — Investor-Demo Prototype (Track A)

**Feature:** `001-tdc-phr-patient-app` (directory-scoped; trunk-based on
`main` — no per-feature git branch exists post-monorepo-migration, see
`plan.md` for detail)
**Created:** 2026-07-15
**Status:** Active
**Owner:** Adi (dev) / Soham (demo)
**Source spec version:** 0.1 (15 Jul 2026)
**Input:** Build the investor-demo prototype of the TDC patient/family PHR app
(display name: **TDC Health**). Investor pitch: **16 Aug 2026**; meetup/user
demos begin early Aug (meetup loop adds `010`'s booking beat — this spec's
5-minute golden path is the investor script and is unchanged). Seeded data,
demo-grade quality, Android + web via one codebase.

> **Supersession notice:** This specification supersedes any patient-app scope
> in older handover / Hyperledger-era documents.

---

## What this is / is NOT

**IS:** The build spec for the investor-demo prototype of the TDC patient/family
app. Seeded data, demo-grade quality, Android + web from one codebase.

**IS NOT:** The production MMP spec. No payments, no real ABDM data exchange, no
DPDP-grade hardening. **Track B re-specs all of this post-funding.**

---

## Demo Goal & Golden Path

The prototype exists to make an investor **feel one story**:
> *"A family's health, handled — even when the worst happens."*

### Primary User Story (the 5-minute script)
As **Rohan** (34, Pune, caregiver) I open the app and see **3 family profiles**:
Rohan, Asha (Aai, 61, diabetic + penicillin allergy), Prakash (Baba, 66,
hypertension). I demonstrate that a whole family's health is handled from one
phone — including the worst-case emergency — in under five minutes.

### Acceptance Scenarios (golden path, ordered)
1. **Given** the app opens as Rohan, **When** the Home screen loads, **Then**
   three family profile cards are shown (Rohan, Asha, Prakash).
2. **Given** Asha's profile, **When** I tap it, **Then** a records timeline
   renders newest-first with seeded prescriptions and lab PDFs.
3. **Given** Asha's timeline, **When** I camera-upload one paper prescription
   live, **Then** it appears in her timeline immediately.
4. **Given** Asha's profile, **When** I tap "Health Summary", **Then** a
   one-page summary renders and can be **shared as a PDF via WhatsApp**.
5. **Given** the Meds tab for Asha, **When** it loads, **Then** her Metformin
   schedule is shown, the phone displays an 8pm reminder notification, and the
   stock counter reads **"4 days left"**.
6. **THE MONEY SHOT — Given** a second, logged-out phone, **When** it taps
   Asha's NFC card, **Then** the responder page loads in **<2s with no login**
   showing blood group, allergies, conditions, meds, and emergency contacts,
   **and simultaneously** Rohan's phone (and Soham's, as a second family member)
   buzzes with *"Asha's emergency card was just scanned near Kothrud."*
7. **Given** the demo close, **When** I open the Trust screen, **Then** it
   states: encrypted, stored in India, every access logged, you control sharing.

### Stretch Scenario (only if HMS slice lands — P1)
- **Given** a doctor writes an Rx in TDC HMS, **When** the slice fires, **Then**
  the Rx appears in Asha's timeline. One sentence: *"and it's connected to our
  clinic system."*

### Edge Cases
- Camera/gallery upload fails → record is not created; user sees a retry, no
  half-saved state (seed reset stays clean).
- Scanner does not share location → family blast still fires, omits the "near
  {place}" clause gracefully.
- Core API is down during the demo → Fastlane responder page STILL renders from
  its snapshot (Principle I).
- Invalid or revoked card scanned → neutral page only: *"This card is inactive.
  If this is an emergency, call 108."* No data, no confirmation the card exists.
- FCM latency on stage → a backup screen-recording of the golden path exists.

---

## Personas & Seeded Data

| Persona | Role | Seeded data |
|---------|------|-------------|
| **Rohan S.** (34) | Account owner, caregiver over both parents | Own profile mostly empty (realistic) |
| **Asha K.** (61, Aai) | Demo-rich profile | 8–10 records across 18 months (Rx, HbA1c lab PDFs, 1 discharge summary), 2 active meds, complete emergency profile (blood group, penicillin allergy, diabetic) |
| **Prakash P.** (66, Baba) | Lighter profile — proves multi-profile without clutter | 3 records, 1 med, hypertension |

`seed_demo.py` MUST rebuild this exact state in **one command** (Principle V).

---

## Scope

### P0 — MUST work on demo day
| # | Feature | Notes |
|---|---------|-------|
| 1 | Phone-OTP onboarding | Mock OTP `000000` in demo mode; real OTP infra is Track B |
| 2 | Family profiles + assisted add | Caregiver-grant recorded to `caregiver_grants` log — **this IS the consent moment** |
| 3 | Record upload (camera/gallery) + type tag (Rx / Lab / Discharge / Other) + timeline | No OCR — store image/PDF + manual title. *"Digitize the plastic bag"* |
| 4 | One-page Health Summary per profile → share as PDF | Template render from structured fields + record list; **not AI-generated** |
| 5 | Medications: schedule, local reminder notifications, stock countdown | Flutter local notifications; low-stock badge at ≤5 days |
| 6 | Emergency profile: blood group, allergies, conditions, meds, ABHA no. (text field), 2+ contacts | Per profile |
| 7 | Card link + revoke: bind NTAG 424 UID to profile; active flag kill-switch | Revocation = one toggle |
| 8 | Responder page (Fastlane, server-rendered) | The crown-jewel screen — see §Emergency Card Flow |
| 9 | Family blast: FCM push to all linked family devices on card scan | Include coarse location if scanner shares it; degrade gracefully if not |
| 10 | Trust screen | Approved copy only — see §Copy Guardrails |

### P1 — build ONLY if P0 is done (cut bottom-up per slip rule)
- **Consult booking + seeded queue tracking** (`010` — unlocked 1 Aug 2026,
  meetup-demo driver; seeded/fictional only, not in the investor golden path).
- **Care discovery** (`011` — specced 20 Aug 2026: find-care screen, clinic
  information page, seeded ~10-clinic directory). Step 1 of `010`'s flow, so
  it builds immediately before it and pauses with it. Same fences: profile-
  context entry only, no Home surface, seeded everything.
- HMS→app vertical slice (Rx created in CARE fork lands in timeline; shared
  DB/webhook fake is acceptable).
- ABHA M1 create-via-Aadhaar — sandbox creds **arrived** (2026-08-17); the
  condition that gated this is resolved, so it's buildable if P0 time
  allows. Still P1 (cut bottom-up first if a day slips) — the field stays
  manual if it doesn't get built.
- Marathi toggle on responder page.
- Test-scan celebration screen (shareable moment).

### Out of scope — do not build, do not fake beyond a lock icon
Payments/paywall (show 🔒 "Family Plan" badge max; 🔒 "pay at clinic" inside
booking) · OCR/AI extraction · ABDM M2/M3 data exchange · **real UHI
integration** (seeded booking + seeded directory per `010`/`011` only) · iOS ·
offline mode · real auth hardening · doctor-app features · pet profiles ·
**real-time HMS queue feeds** (`010`'s queue is a seeded simulation) ·
**symptom search** (rejected 20 Aug 2026 — medical inference, and it would put
health complaints in search logs) · written reviews/review counts · any browse
destination reachable without first choosing a profile (Principle VIII).

---

## Screens (12)

1. **Splash/Login** — phone + OTP, demo bypass.
2. **Home** — family profile cards, alerts strip (low stock, missed dose).
3. **Profile timeline** — records newest-first, filter chips by type, FAB → upload.
4. **Upload** — camera/gallery → crop → title + type + date → save. Full
   capture flow (multi-page scan, platform scoping) in
   [`005-records-capture-timeline`](../005-records-capture-timeline/spec.md).
5. **Record detail** — full-screen image/PDF, share, delete.
6. **Health Summary** — the one-pager; share-as-PDF button prominent.
7. **Meds list (per profile)** — today's bundled dose checklist + stock bars.
8. **Med add/edit** — name, dose, times, stock count, refill threshold.
   Full dose-occasion / reminder / stock model in
   [`006-medications-reminders-adherence`](../006-medications-reminders-adherence/spec.md).
9. **Emergency profile editor** — fields that feed the card payload; "what
   responders will see" preview.
10. **Card manager** — card status (linked/active), test-scan button, revoke
    toggle, scan log.
11. **Family & consent** — profiles, caregiver grants with timestamps
    (*"You manage Asha's profile — granted 12 Jul 2026"*), revoke.
    Screens 9–11 fully specced (incl. abuse pass) in
    [`007-emergency-profile-card-consent`](../007-emergency-profile-card-consent/spec.md).
12. **Trust/security screen** — static, approved copy.

**Design system:** shared Flutter `ThemeData` tokens; warm/calm patient palette
(site blue `#4B83F2` family); large type + tap targets (elderly users); every
screen readable at arm's length.

---

## 4.1 Post-Onboarding UX Loops

Home (screen 2) is not one screen with one job — it's the surface for three
distinct loops. Getting the balance between them right is what keeps the app
useful without becoming noisy.

### Loop 1 — First session (setup)
Right after onboarding (see `002-onboarding-router-activation` for the router
and account-creation detail), Home opens close to empty: one profile card
(the user's own) and a single featured prompt, weighted by the router answer
per `002` §"Home: persona-weighted first actions" — for most caregivers this
resolves quickly to **"Add a family member"**, since assisted-add writes the
`caregiver_grants` row (`001` FR-003) and a new profile card appears
immediately.

Each profile card then carries its own **setup checklist as chips**:
*Add records · Add medicines · Complete emergency profile · Link card.*
Empty states do the guiding rather than a separate onboarding wizard per
profile:
- Timeline empty state: *"Digitize the plastic bag — snap her first
  prescription."*
- Meds empty state: *"Add her medicines to get reminders."*

Chips disappear as each is completed. **"Setup complete"** = emergency
profile filled (FR-009) + card linked and active (FR-011) — this moment is
celebrated via the test-scan / shareable-moment screen (P1, §3 build order).
A fully set-up profile card renders in a visually "calm" state (no chips,
no accent border) — the quiet state is itself the reward.

### Loop 2 — Daily (retention)
Home = family profile cards + an **alerts strip** above them. The alerts
strip, not the app icon, is the retention engine: *"Metformin: 4 days left,"
"Aai's 8pm dose," "HbA1c due Sep."* On most days the user does not open the
app unprompted — a local reminder notification (FR-008) deep-links straight
to the relevant med or profile.

- **NFR-006 (Daily-loop latency):** notification → glance → resolved action
  MUST be achievable in under 10 seconds (no more than one intermediate
  screen between notification tap and the action it names).

### Loop 3 — Event (high-stakes, rare)
Two triggers: a doctor visit (open profile → Health Summary → share PDF via
WhatsApp, FR-007) and a card scan (responder page + family blast, FR-012/014
→ tap notification → Card manager scan log, e.g. *"scanned 2 min ago near
Kothrud, payload viewed"*). These are infrequent — a handful of times a
year — but are the moments that justify the app's existence to the user, not
just the daily-loop conveniences.

### Design principle: Home stays quiet
No content feed, no "health tips," nothing competing with the three loops
above. The app's job is to finish onboarding, then go quiet until something
actually needs the user's attention. For the 61-year-old user this is an
accessibility choice; for the caregiver evaluating the app this quietness is
itself a trust signal (echoes Principle VI, data-sovereignty framing, in the
constitution — the app doesn't need to manufacture engagement to be worth
paying for).

### Forward-compatibility note (Track B)
When ABHA sync (`003-abdm-sync-subscription`) lands post-funding, it slots
into this exact structure with **no new surfaces**: sync becomes one more
setup-checklist chip ("Turn on automatic sync") and inbound records become
one more alerts-strip source ("New lab report from Ashirwad Diagnostics added to Aai's
timeline") — the loop model doesn't change, only its inputs.

### Why this shapes the demo script
`seed_demo.py` (FR-017) exists precisely because an investor should never see
Loop 1 from a cold start — that's too slow to demo convincingly. The golden
path (§Demo Goal) instead opens on a **lived-in** Home (Asha's low-stock
alert already visible), demonstrates one Loop-1 action live (the camera
upload), and closes on a Loop-3 event (the card scan). Lived-in state + one
live setup action + one emergency beat is the complete UX story in five
minutes.

---

## Emergency Card Flow (Fastlane) — the crown jewel

- **URL on chip (SDM):**
  `https://e.thedoorstepclinic.com/e/{uid}?ctr={counter}&cmac={cmac}`
  NTAG 424 DNA SDM mirrors counter + CMAC; server verifies CMAC with per-card
  key and **rejects `ctr ≤ last_seen`** (replay kill).
- **Responder page:** server-rendered HTML, **zero JS required to read**, **<2s
  on 4G, no login**. Render order:
  1. **BLOOD GROUP** (largest)
  2. **ALLERGIES** (red)
  3. Conditions
  4. Current meds
  5. Emergency contacts (tap-to-call)
  6. ABHA number ("for hospital records access")
  7. "Access logged & family notified" banner
  8. EN/MR toggle (P1)
- **On every valid scan:** log `scan_event(uid, ctr, ts, ip, coarse_geo?)` → FCM
  blast to family devices → increment card counter.
- **Consent framing (do not drift):** consent was granted once, at setup,
  in-app, logged, revocable. **The card is a capability token exercising a
  standing grant — never describe card-tap as consent** (Principle II).
- **Invalid/revoked card:** neutral page — *"This card is inactive. If this is
  an emergency, call 108."* No data, no existence confirmation.

---

## Requirements

### Functional Requirements
- **FR-001**: The system MUST let a user onboard via phone + OTP, accepting mock
  OTP `000000` in demo mode.
- **FR-002**: The system MUST support multiple family profiles owned by one
  account, each with a `relation` to the owner.
- **FR-003**: Adding a caregiver relationship MUST write a `caregiver_grants`
  record with a timestamp; this record is the recorded consent moment.
- **FR-004**: A user MUST be able to revoke a caregiver grant, recording a
  `revoked_ts`.
- **FR-005**: The system MUST let a user upload a record via camera or gallery,
  tag its type (Rx / Lab / Discharge / Other), set a manual title and date, and
  see it in the profile timeline newest-first. No OCR.
- **FR-006**: The timeline MUST support filtering by record type.
- **FR-007**: The system MUST render a one-page Health Summary per profile from
  structured fields + record list and share it as a PDF (WhatsApp target). Not
  AI-generated.
- **FR-008**: The system MUST let a user define medications (name, dose,
  times[], stock count, refill threshold), schedule local reminder
  notifications, and show a low-stock badge when remaining days ≤ 5.
- **FR-009**: The system MUST maintain a per-profile emergency profile: blood
  group, allergies, conditions, current meds, ABHA number (free text), and 2+
  emergency contacts with tap-to-call.
- **FR-010**: On any change to data feeding the emergency profile, the system
  MUST update a denormalized `emergency_payload` snapshot that Fastlane reads.
- **FR-011**: The system MUST bind an NTAG 424 UID to a profile and expose an
  `active` kill-switch; revocation is a single toggle.
- **FR-012**: Fastlane MUST serve a server-rendered responder page at
  `GET /e/{uid}` that reads only the snapshot, renders readable with zero JS,
  and loads in <2s on 4G with no login.
- **FR-013**: Fastlane MUST verify the CMAC with the per-card key and reject any
  request where `ctr ≤ last_seen`.
- **FR-014**: On a valid scan, Fastlane MUST log a `scan_event`, send an FCM
  blast to all linked family devices (including coarse location when provided,
  gracefully omitted when not), and increment the card counter.
- **FR-015**: For an invalid or revoked card, Fastlane MUST return a neutral
  page with no data and no confirmation the card exists.
- **FR-016**: The Trust screen MUST display only approved copy (see Copy
  Guardrails).
- **FR-017**: `seed_demo.py` MUST rebuild the full demo world (3 profiles,
  seeded records, meds, emergency profiles, linked cards) in one command.
- **FR-018**: No user-facing surface may use any banned term or imply a
  card-tap equals consent.
- **FR-019** *(P1)*: An Rx created in the CARE-fork HMS MUST be able to appear
  in the corresponding profile's timeline (shared DB / fake webhook acceptable).
- **FR-020** *(P1)*: The responder page MUST support an EN/MR language toggle.
- **FR-021** *(P0, cross-cutting — added 2026-08-17, constitution Principle
  XI `[A]`)*: The system MUST emit one `access_logs` row — actor, subject
  profile, action, object type + id, timestamp, purpose, source service —
  for every PHI read or write through Core API or Fastlane, written in the
  same transaction as the access it records; a failed log write MUST fail
  the request. Distinct from `scan_events` (card mechanics only). This is
  what makes the Trust screen's "every access is logged" claim (FR-016,
  Principle VI) literally true rather than aspirational. Design in
  `data-model.md` §access_logs and `contracts/core-api.md` §Access Logging;
  built by `tasks.md` T006/T025.
- **FR-022** *(P0, cross-cutting — added 2026-08-17, constitution Principle
  X `[A]`)*: Every Core API endpoint touching `profiles`, `records`,
  `medications`, `emergency_profiles`, `cards`, or `scan_events` MUST derive
  its result set server-side from the authenticated caller — the caller's
  own profiles union profiles reachable through an unrevoked
  `caregiver_grants` row — never from a client-supplied id, header, or body
  field. An endpoint returning an unfiltered/unscoped result set is a build
  failure, not a review comment. Revoking a grant MUST deny access on the
  very next request (no cached scope). A request for a resource outside the
  caller's scope and a request for a nonexistent resource MUST be
  indistinguishable to the caller (see FR-015's analogous rule for Fastlane;
  Core API's error conventions collapse both to `404`, corrected
  2026-08-17). Design in `contracts/core-api.md` §Authorization Scoping and
  `research.md` R9; built by `tasks.md` T005.

### Non-Functional Requirements
- **NFR-001 (Perf):** Responder page < 2s on 4G.
- **NFR-002 (Availability):** Fastlane survives Core API downtime (snapshot
  isolation).
- **NFR-003 (Security, prototype-grade):** Card replay protection via CMAC +
  monotonic counter. Full auth hardening is Track B.
- **NFR-004 (Accessibility):** All screens readable at arm's length; large type
  and tap targets for elderly users.
- **NFR-005 (Resettability):** One-command demo reset.
- **NFR-006 (Daily-loop latency):** Reminder notification tap → resolved
  action in under 10 seconds, no more than one intermediate screen (see
  §4.1 Post-Onboarding UX Loops).

### Key Entities
- **users** — account holders.
- **profiles** — FK owner, `relation`; one per family member.
- **caregiver_grants** — grantor, grantee, `ts`, `revoked_ts` (the consent log).
- **records** — profile, type, title, file, `record_date`.
- **medications** — profile, name, dose, `times[]`, `stock_count`, `threshold`.
- **emergency_profiles** — profile 1:1, blood_group, allergies, conditions,
  abha_no.
- **emergency_contacts** — profile, name, phone, relation, priority.
- **cards** — uid, profile, sdm_key_ref, last_ctr, active.
- **scan_events** — card, ctr, ts, ip, geo, `is_test`.
- **emergency_payload** — denormalized snapshot per card/profile, read by
  Fastlane (derived, not a primary source of truth).
- **access_logs** — actor, subject profile, action, object type + id, `ts`,
  purpose, source service (the audit trail — FR-021, distinct from
  `scan_events`).

Full field-level detail in [`data-model.md`](./data-model.md).

---

## Copy Guardrails

**Approved:** "encrypted" · "stored only in India" · "never sold" · "you control
who sees what" · "every access is logged" · "built on India's ABDM framework
(integration in development)".

**Banned:** blockchain · Hyperledger · "AES-256" or any cipher names in UI ·
"ABDM certified" · "military-grade" · any implication that card-tap = consent.

---

## Build Order (remaining ~12 working days)

| Day | Work |
|-----|------|
| D1–2 | Flutter skeleton, auth, profiles + grants, seed script v0 |
| D3–4 | Records upload + timeline + detail |
| D5 | Health summary + PDF share |
| D6–7 | Meds + reminders + stock |
| D8–9 | Fastlane service, payload snapshot, responder page, scan log |
| D10 | Card personalization (write SDM URL to 2 physical cards), test scan, FCM blast end-to-end |
| D11 | Trust screen, polish pass, full seed, APK build |
| D12 | Demo rehearsal ×3, record backup video of golden path |

**Rule:** if a day slips, cut from the bottom of P1, then polish — **never** from
the emergency flow.

---

## Demo-Day Checklist & Risks

**Checklist**
- ☐ 2 physical cards written + 1 spare
- ☐ demo phone (app) + responder phone (logged out, NFC on) + Soham's phone
  (family blast)
- ☐ hotspot fallback
- ☐ `seed_demo.py` re-run before demo
- ☐ backup screen-recording of full golden path
- ☐ airplane-mode test of what breaks

**Top risks**
- NFC read variability across phones → test on 3+ devices by D10.
- FCM latency on stage → pre-warm connection; video backup.
- ~~Sandbox creds not arriving → ABHA stays a text field~~ — **RESOLVED:**
  sandbox creds arrived 2026-08-17. ABHA M1 is now buildable in P1 (see
  Scope); the manual-field fallback remains the plan only if it doesn't get
  built in time.

---

## Open Decisions
| Decision | Owner | By |
|----------|-------|-----|
| ~~App display name~~ — **RESOLVED: "TDC Health"** (15 Jul 2026) | Soham | ~~D5~~ ✅ |
| Summary PDF layout sign-off (clinical review) | Sharvari | D6 |
| Include HMS slice in demo? (default **NO**) | Team | D9 |

---

## Review & Acceptance Checklist

### Content Quality
- [x] Focused on user value and the demo story.
- [x] Scope explicitly bounded (P0 / P1 / out-of-scope).
- [x] Written so both dev and demo owners can act on it.
- [x] All mandatory sections completed.

### Requirement Completeness
- [x] Requirements are testable (map to golden-path acceptance scenarios).
- [x] Success criteria measurable (<2s responder, ≤5-day badge, one-command reset).
- [x] Dependencies/assumptions identified (sandbox creds, HMS slice, NFC devices).
- [x] App display name resolved — **"TDC Health"** (decided 15 Jul 2026).

---

## Execution Status
- [x] User description parsed
- [x] Key concepts extracted
- [x] Ambiguities marked
- [x] Scenarios defined
- [x] Requirements generated
- [x] Entities identified
- [x] Review checklist passed
