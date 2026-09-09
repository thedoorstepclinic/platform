# Feature Specification: Emergency Profile, Card Manager & Family Consent Surfaces

**Feature Branch:** `007-emergency-profile-card-consent`
**Created:** 2026-07-16
**Status:** Draft
**Owner:** Adi (dev)
**Depends on:** [`001-tdc-phr-patient-app`](../001-tdc-phr-patient-app/spec.md) —
implements FR-009/010/011 and the app-side halves of FR-012–015 at UX level.
Covers screens 9–11 (Emergency profile editor, Card manager, Family &
consent). The Fastlane service itself (CMAC verify, replay-kill, responder
render, blast) stays specced in `001` §6 + `contracts/fastlane-api.md` —
this spec does not restate it.
**Ripples into:** [`004-seed-data-and-summary`](../004-seed-data-and-summary/spec.md)
(scan-log seed) and [`../../CLAUDE.md`](../../CLAUDE.md) (`scan_events`
gains `is_test`) — updated same-day per the model-change rule.

## What this is / is NOT

**IS:** The app-side surfaces of the emergency flow — editing what
responders will see, managing the physical card's lifecycle (link, test,
revoke, replace), and the family-consent ledger. Includes the abuse-case
pass required by `CLAUDE.md` for anything touching grants or Fastlane.

**IS NOT:** The Fastlane service internals (already locked in `001` §6),
card *personalization* (writing the SDM URL to physical NTAG chips — a
D10 bench task, not an app feature), or any ABDM consent surface (`003`).

---

## Consent framing (do not drift — restated because this is where it drifts)

Every screen in this cluster is a **trust surface**. The standing rules:
- Consent happened **once, in-app, at profile setup** — the
  `caregiver_grants` row with its timestamp IS the consent record.
- The card is a **capability token exercising that standing grant**. No
  copy on any of these three screens may describe a tap as consent,
  permission, or approval.
- Approved copy only (`CLAUDE.md`): "every access is logged" · "you control
  who sees what" · "one-time permission."

---

## Screen 9 — Emergency profile editor

The fields that feed the card payload, per profile: blood group, allergies,
conditions, ABHA number (free text), emergency contacts (2+ with priority).

### The preview is the contract
The **"what responders will see" preview** MUST render from the same
denormalized `emergency_payload` snapshot that Fastlane reads (`001`
FR-010) — not from a second, parallel template. If the preview and the real
responder page can drift apart, the preview is a lie; sharing the data
source makes drift structurally impossible. Practically: on save → Core
writes the snapshot → the preview renders that snapshot in responder-page
order (blood group largest, allergies red, etc.).

### Behavior
- Saving any field updates the snapshot immediately for every active card
  bound to the profile (`001` FR-010).
- Contacts: minimum 2 to mark the emergency profile "complete" (the setup
  checklist chip from `001` §4.1); fewer is allowed to *save* (never trap
  a user mid-edit), but the chip stays open and the preview shows a
  "add a second contact" hint.
- The current-meds section of the payload is **derived from active
  medications** (`006`), not hand-typed here — one source of truth. The
  editor shows them read-only with a "manage in Medications" link.

## Screen 10 — Card manager

Card lifecycle from the caregiver's side. States: **no card** → **linked +
active** → **revoked**; replacement = revoke old, link new.

### One active card per profile (decision)
A profile has **at most one active card** at a time, enforced at the data
level, with old rows retained (`active = false`) as history. A lost card is
replaced by revoking it and linking the new one — two taps, and the revoked
card's UID stays in the scan log's history. Multiple simultaneously-active
cards multiply the abuse surface (each is an unauthenticated data URL in
the world) for no benefit.

### Test scan (decision — real round-trip, not a mock)
The test-scan button exercises the **real pipeline**, not an in-app
simulation, because the whole point is proving to the caregiver (and the
investor) that the real thing works:

1. App requests a test scan from Core → Core asks Fastlane to mint a
   **short-lived, single-use test URL** for the card (no physical tap
   involved, so no chip counter is consumed — the replay-kill `last_ctr`
   is untouched).
2. The URL opens in the device browser → the **actual Fastlane-rendered
   responder page** (from the snapshot, zero JS — the very page a stranger
   would see).
3. The family blast fires for real, **labeled as a test**: "Rohan ran a
   test of Asha's emergency card" — so family phones buzz but nobody
   panics.
4. The scan is logged with `is_test = true`, visible in the scan log with a
   "Test" chip.

A physical tap of the real card with the phone's own NFC also works, of
course — that's just a normal scan; the button exists so testing doesn't
require a second device.

### Scan log
Reverse-chronological list of `scan_events` for all of this profile's cards
(active + revoked): timestamp, coarse location if present, Test chip if
`is_test`. Copy framing: *"Every scan is logged and your family is
notified"* — the log is the proof of the trust claim, same surface-of-trust
role as `003`'s access log.

### Revoke
One toggle → `active = false` → Fastlane serves the neutral page
immediately (`001` FR-015). Revoking asks for one confirmation ("Responders
will no longer see Asha's information from this card") — this is the one
place where friction is correct. Re-activation of a revoked card is allowed
(toggle back) as long as no other card is active for the profile.

## Screen 11 — Family & consent

The consent ledger, per `001` FR-003/004:
- Each grant shown with plain-language framing + timestamp: *"You manage
  Asha's profile — granted 12 Jul 2026."*
- Revoking a grant sets `revoked_ts` (one action + one confirmation) and is
  itself shown in history ("revoked 3 Aug 2026") — the ledger never
  forgets, that's what makes it a ledger.
- Family members who joined via QR/invite code (`002`) appear here as
  *linked members*, visually distinct from *managed profiles* — the screen
  must not blur "I manage Aai" with "my brother is in the family group."
- Forward note: this screen is where `003`'s per-profile sync toggle and
  consent-artifact list will live. No new surface later — reserved section,
  hidden until `003`'s gate clears.

---

## Abuse pass ("how would I abuse this?" — required by CLAUDE.md)

| Attack | Defense |
|---|---|
| Guess/enumerate card UIDs to fish for data | Invalid UID → same neutral page as revoked, no existence confirmation (`001` FR-015). |
| Replay a captured scan URL | CMAC + monotonic counter at Fastlane (`001` FR-013); test URLs are single-use + short-lived and don't touch the counter. |
| Screenshot/share a test URL before it's used | Single-use + expiry bounds the window; the page itself contains only what a physical tap would reveal anyway — the card holder chose that exposure at setup. |
| Spam test scans to flood family with pushes | Rate-limit test scans per card (e.g. a few per hour, server-side); every one is logged and labeled. |
| Ex-caregiver retains access after falling out with family | Grant revoke (`revoked_ts`) kills app access; card revoke kills the responder page. Both one-toggle, both on surfaces the account owner controls. |
| Revoked card re-activated by someone with brief phone access | Re-activation lives behind the app's normal auth session. **Re-open (6 Sep 2026):** the old answer leaned on "no real auth hardening, per scope", which is repealed — auth hardening is in scope now. Re-activating a revoked emergency card is a high-consequence action taken from an unlocked phone; app-lock (`docs/backlog.md` §3) and a step-up confirm both bear on it. Owner: Adi. |
| Malicious "family member" joins via leaked invite code | Codes are single-use, short-lived, show inviter identity before join (`002` FR-005), and joining grants *visibility linkage only* — never caregiver rights over any profile. |

---

## Screen 13 — Emergency Alert (added 9 Sep 2026)

**The family-side landing surface after a scan**, at
`/profile/:profileId/alert/:scanEventId`. Added from a supplied reference
design. Previously the family blast (`001` FR-014) deep-linked into the card
manager's scan log — a maintenance screen, not the thing a person opening a
2am notification needs. This is that thing.

| Element | Behaviour |
|---|---|
| **Alert banner** | *"Asha's emergency card was scanned."* + when, + by whom if the responder said. `alert` on `alert-soft`, never white-on-red (Principle VII / `docs/design-tokens.md`). |
| **Where** | Coarse location when the scanner shared it (*"near Kothrud"*, `001` FR-014), gracefully absent when not. **Plus responder-supplied facility name where given**, clearly labelled as such. |
| **Get Directions** | To the responder-supplied facility if there is one, otherwise to the coarse point, otherwise absent. Never to a guess. |
| **Who** | Responder-supplied name and callback number, shown **only when provided**, under a heading that says they are unverified. A tap-to-call on the callback number is the single most useful control on this screen. |
| **Share fuller records** | The record-sharing grant. See below — it is the highest-risk control in the app. |
| **Map** | Centred on whatever location we actually have. Absent if we have none. |
| **Report false call** | One tap. See below. |

### The banner must not claim a protocol

The reference reads *"Soham's emergency protocol is triggered."* **Rejected.**
TDC does not run a protocol and does not trigger anything — a stranger tapped a
card and a page rendered. The banner states the event: **the card was scanned,
when, and where if known.** Anything more implies the app is coordinating a
medical response it has no part in.

### Responder-supplied context — optional, unverified, never blocking

Fastlane serves an **unauthenticated public page**. We do not know who scanned
the card, which facility they are in, or who is treating the patient. The
reference showed all three as fact; they are only available if the responder
volunteers them.

So the responder page gains **optional** fields — facility, responder name,
callback number — with three hard rules:

1. **They MUST NOT gate or delay the medical information.** Blood group,
   allergies, conditions, meds and contacts render first, always. A responder
   who fills in nothing sees exactly what they see today. Principle I is not
   negotiable for a data-collection convenience.
2. **Everything supplied is displayed to family as unverified**, attributed to
   the responder. *"Reported by the person who scanned the card."*
3. **They are not trusted as input.** Free text rendered to a family member is
   an injection and social-engineering surface: escaped on render, length-capped,
   no links made clickable except a validated phone number, and stored on the
   `scan_event` — never used to look up or auto-associate a real facility.

### Share fuller records — the reframe, and why

The reference's **"Medical Consent Required → Provide E-Consent Now"** is not
built. It presented treatment authorization: *"the hospital requires immediate
authorization to proceed with life-saving emergency protocols."* Three reasons,
recorded so this does not get re-proposed:

1. **It is false, and dangerously so.** Clinicians treat under emergency
   doctrine when a patient cannot consent; no hospital waits on an app. A
   screen implying treatment is blocked on a family member's tap tells someone
   whose parent is in an emergency ward that the delay is theirs. If they are
   asleep or out of signal, the app has lied at the worst possible moment.
2. **Principle II (NON-NEGOTIABLE)** — no screen may make a card tap the moment
   consent is asked for. A scan that raises a consent prompt is card-tap-as-
   consent by another route.
3. **TDC cannot broker it.** `caregiver_grants` governs *seeing records*.
   Authorizing a procedure is a legal act between the patient or next-of-kin
   and the clinician, and putting a button on it misrepresents what the button
   does.

**What replaces it is a real feature and fits the existing consent model:** the
responder has seen the emergency card and may need more — past discharges, the
full medication history, prior labs. That is **data access**, which is exactly
what TDC's consent model governs, and the family member grants it deliberately
in-app rather than by anyone tapping anything.

**It widens a public bearer URL, so it is bounded hard:**

- **Explicit, never pre-checked, never automatic.** One tap, one confirmation
  naming what is being shared and with whom.
- **Time-boxed and short** — default **6 hours**, shown as a countdown on this
  screen and in the card manager.
- **Revocable in one tap**, from this screen and from Screen 10, with the same
  one-toggle pattern card revocation already uses.
- **Logged** — an `access_logs` row on the grant, on every expanded read, and
  on revocation (Principle XI).
- **Scoped to this scan event**, not to the card in general. A later scan does
  not inherit it.
- **The expanded payload is a superset of the emergency payload**, assembled
  the same way and from the same source (Principle IX — no parallel template).
- **Open question, flagged not answered:** the emergency URL is a bearer token,
  so expanding what it serves also expands what a screenshot of it serves. The
  safer construction is a **separate short-lived single-use link sent to the
  responder's supplied callback number**, which requires a callback number to
  exist. Owner: Adi. This must be settled before the grant is built — it is
  the difference between sharing a history with a clinician and publishing it.

### Report false call

**Adopted wholesale — the reference has something the spec was missing.** `007`
had revocation but nothing that says *"this scan was not legitimate."*

One tap, and it does three things: flags the `scan_event`, offers card
revocation immediately in the same sheet (the correct response to a card in the
wrong hands), and **immediately ends any active record-sharing grant** for that
scan. It also stops further alerts for that event so a frightened family member
is not re-notified about something they have already judged.

It is deliberately the last element on the screen and never a primary action —
a real emergency must not compete with a fraud report — but it is always
present, because the abuse case this app enables is a stranger scanning a card
to harvest a family's data, and the family needs somewhere to say so.

---

## Requirements

### Functional Requirements
- **FR-001**: The emergency-profile preview MUST render from the same
  `emergency_payload` snapshot Fastlane serves — one data source, zero
  drift by construction.
- **FR-002**: Saving emergency-profile fields MUST refresh the snapshot for
  all active cards of the profile immediately (per `001` FR-010).
- **FR-003**: The payload's current-meds section MUST derive from active
  `medications` rows (`006`), read-only in this editor.
- **FR-004**: A profile MUST have at most one active card at any time;
  linking a new card while one is active requires revoking the old one
  first. Revoked card rows are retained.
- **FR-005**: Test scan MUST exercise the real Fastlane render and real
  family blast via a short-lived, single-use, server-minted URL that does
  not consume the chip counter; the blast and log entry MUST be labeled as
  a test.
- **FR-006**: Test scans MUST be rate-limited server-side and MUST appear
  in the scan log with `is_test = true`.
- **FR-007**: The scan log MUST show all scan events for the profile's
  cards (including revoked cards' history): timestamp, coarse location if
  present, test label.
- **FR-008**: Card revoke MUST be a single toggle behind one confirmation,
  taking effect immediately (neutral page on next scan). Re-activation is
  allowed while no other card is active.
- **FR-009**: The Family & consent screen MUST show every grant with its
  timestamp in plain language, keep revoked grants visible as history, and
  visually distinguish managed profiles from QR-joined linked members.
**Emergency Alert (Screen 13, added 9 Sep 2026)**

- **FR-011**: The family blast (`001` FR-014) MUST deep-link to the Emergency
  Alert screen for that `scan_event`, not to the card manager. It MUST render
  from the scan event, and MUST NOT require the card manager to load first.
- **FR-012**: The banner MUST state the event — the card was scanned, when,
  and where if known. It MUST NOT describe a protocol, a triggered response, or
  any coordination TDC does not perform.
- **FR-013**: Facility, responder name and callback number are **optional
  responder-supplied** values. They MUST be displayed as unverified and
  attributed to the person who scanned the card, MUST be escaped on render and
  length-capped, MUST NOT be used to look up or auto-associate a real facility,
  and only a validated phone number may be made tappable.
- **FR-014**: Collecting responder-supplied context MUST NOT gate, delay, or
  reorder the medical information on the responder page. Blood group,
  allergies, conditions, meds and contacts render first, always (Principle I).
- **FR-015**: The app MUST NOT present, request, or imply **treatment
  authorization**. No screen may state that a clinician is awaiting a family
  member's approval to proceed. Emergency treatment does not wait on this app,
  and saying otherwise is both false and harmful (Principle VI).
- **FR-016**: A record-sharing grant MUST be explicit and never pre-checked,
  MUST be time-boxed (default 6 hours) with the remaining time visible, MUST be
  revocable in one tap from both Screen 13 and Screen 10, MUST be scoped to the
  originating `scan_event` rather than to the card, and MUST write an
  `access_logs` row on grant, on every expanded read, and on revocation.
- **FR-017**: The expanded payload MUST be assembled from the same source as
  the emergency payload — a superset, never a second hand-maintained view
  (Principle IX).
- **FR-018**: The Emergency Alert screen MUST offer **Report false call**,
  which flags the scan event, offers card revocation in the same sheet,
  immediately ends any active record-sharing grant for that scan, and stops
  further alerts for that event. It MUST NOT be a primary action.

- **FR-010**: No copy on these three screens may describe a card tap as
  consent/permission/approval; all copy comes from the approved list.

### Non-Functional Requirements
- **NFR-001 (Immediacy):** revoke (card or grant) takes effect on the next
  request — no caching window that outlives the user's decision.
- **NFR-002 (Elderly-first):** these screens follow the same
  arm's-length/large-target standard; the preview especially, since it may
  be shown *to* the elder to explain what the card does.

---

## Data model change

`scan_events` gains **`is_test` (bool, default false)**. Same-day ripples
applied: `CLAUDE.md` data-model line + changelog; `004` seeds one test
scan event on Asha's card (dated her card-link day) so the scan log screen
demos lived-in rather than empty.

---

## Open Decisions
| Decision | Owner | Notes |
|---|---|---|
| Test-scan rate limit value | Adi | Propose 3/hour/card; any value beats none. |
| Test URL TTL | Adi | Propose 5 minutes, single-use. |
| Whether re-activating a revoked card warrants a stronger confirm than revoking | Adi | **Revisit.** The old lean (same-weight, asymmetry not worth it) rested on demo scope. Revocation is the safe direction and re-activation is the dangerous one — asymmetric confirmation is the normal production answer. See the abuse-table row above. |

---

## Review & Acceptance Checklist

### Content Quality
- [x] Consent framing restated where it's most at risk of drift.
- [x] Abuse pass included per `CLAUDE.md` convention (grants + Fastlane surfaces).
- [x] No duplication of Fastlane internals (referenced, not restated).

### Requirement Completeness
- [x] Requirements testable (single-active-card invariant, snapshot-sourced preview, labeled test blast, ledger retention).
- [x] Both open questions from review carried as explicit decisions with rationale (test scan = real round-trip; one active card).
- [x] Model change flagged with same-day ripples applied.

## Execution Status
- [x] Scenarios/behavior defined for screens 9–11
- [x] Abuse-case pass done
- [x] Requirements generated
- [x] Ripples applied (`CLAUDE.md`, `004`)
- [x] Review checklist passed
