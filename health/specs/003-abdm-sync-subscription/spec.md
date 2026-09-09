# Feature Specification: ABHA Sync — Subscription-Based Record Inflow

**Feature Branch:** `003-abdm-sync-subscription`
**Version:** 0.2 · **Created:** 2026-07-15 (v0.1 uploaded doc) · **Revised:** 2026-07-15
**Status:** Draft — **GATE: ABDM certification** (WASA audit is a precondition for M1; production access is four gates out — `docs/compliance-baseline.md`). Gate owner: Adi. In scope like every other feature; blocked on an external approval, not deferred by choice. Zero code before the
tenant-isolation audit (Sprint 1) passes — ABDM sandbox credentials arrived
2026-08-17 (see `CLAUDE.md` changelog), so that half of the prerequisite is
satisfied, but Sprint 1 (post-funding) has not started, so this feature
remains gated.
**Owner:** Adi (dev) / Soham (copy sign-off)
**Depends on:** [`002-onboarding-router-activation`](../002-onboarding-router-activation/spec.md) —
this spec begins where a profile's ABHA is already `verified` / `linked` /
`created`. It does not cover account creation, the router, or ABHA linking
mechanics.
**Companion:** [`flowchart.md`](../002-onboarding-router-activation/flowchart.md)
— diagram 2 (state machine) and the sync-permission node in diagram 1 depict
this spec's flow.

> **Revision note (v0.2):** incorporates three decisions made in spec review
> on top of the original v0.1 draft: (1) the subscription grant and first
> historical-fetch consent are shown as **one merged screen**, not two
> sequential ones; (2) the historical fetch is **fire-and-forget**, not a
> blocking wait with a fixed timer; (3) one additional banned phrase closing
> a caregiver-consent drift risk. Changes are marked inline.

**What this is:** The spec for how TDC Health continuously pulls a patient's
records from the ABDM network into their TDC timeline after a single
onboarding permission. Built on ABDM's native **subscription** +
patient-set **auto-approve** primitives, exercised through the Consent
Manager (CM). TDC acts as HIU here.

**What this is NOT:** "Auto consent." The app never grants, implies,
bundles, or defaults consent on the user's behalf. Consent is always an
explicit, per-patient, in-app approval producing a signed CM artifact —
revocable at any time. This document supersedes any looser phrasing ("auto
consent", "fetch everything on install") used in prior chats, decks, or
notes. **The phrase "auto consent" is banned from all TDC docs, code
comments, and pitch materials.**

---

## 1. Product intent (one paragraph)

User links ABHA once (per `002`) and taps **Allow** on one clearly-worded
permission screen that covers both the standing subscription and the first
historical fetch together. From then on, every new record created about them
at any ABDM-live facility (prescription, lab report, discharge summary, OPD
note, immunization) flows automatically into their TDC timeline. Lived
experience: *"I set it up once; now every doctor visit anywhere just shows up
in my app."* Legal reality: one subscription artifact + one consent artifact
(the first fetch) + an optional standing auto-approve policy for subsequent
ones, all patient-approved, all revocable, all access logged.

## 2. Consent model (do not drift)

| Layer | Mechanism | Who grants | When | Revocation |
|---|---|---|---|---|
| Discovery of new records | **ABDM subscription** — CM notifies TDC when a new care context is created for this ABHA anywhere on network | Patient, explicitly, in-app | On the merged permission screen (§3.1), or later from settings | Family & Consent screen, one toggle → subscription cancelled at CM |
| Fetching record content | **Consent artifact** (HIU request) — first fetch approved as part of the same merged screen; patient may opt into CM **auto-approve** policy for subsequent TDC requests | Patient | Same merged screen + optional "don't ask again" toggle | Same screen; also revocable from any ABHA PHR app |
| Family/caregiver layer | **TDC in-app caregiver grant** (existing `caregiver_grants` model, see `001`) — NOT an ABDM construct | Each patient individually (caregiver may *assist* the tap for an elder profile per `002`, but never taps *for* them) | Profile add | Existing revoke flow |

Hard rules:
1. ABDM consent is strictly **per patient**. Rohan approving for himself does
   nothing for Asha; Asha's subscription is approved on/with her own ABHA
   (assisted onboarding = Rohan helps her tap, the artifact is hers).
2. No pre-checked boxes, no consent inside T&Cs, no "by continuing you
   agree." One dedicated screen, one dedicated tap — see §3.1 for why this is
   now a *single merged* screen rather than two sequential ones.
3. Default consent scope is **pre-filled but editable**: all 5 major record
   types, lookback 5 years, purpose `SELF-CARE` (patient) / care-management
   equivalents where applicable, validity 12 months with in-app renewal
   prompt. Never request "all time / all purposes" — high patient rejection
   rate and reads as overreach.
4. Every inbound record write logs: consent artifact ID, subscription ID,
   source HIP, timestamp. Surfaced to user in the access log (same trust
   surface as the card scan log in `001`).

## 3. User-facing flow

### 3.1 Onboarding (per profile with a verified ABHA)

1. ABHA already verified/linked (handled entirely by `002` — not repeated
   here).
2. **Sync permission screen — one screen, one Allow** (revised from v0.1: the
   subscription grant and the first historical-fetch consent are now
   presented together, not as two sequential CM interactions, so the "one
   tap" promise in §1 is actually true for the default path):
   > **Keep {name}'s records up to date**
   > Automatically add new prescriptions, lab reports and hospital records
   > from any clinic on India's ABDM network to this timeline.
   > • You approve this once · • You can turn it off anytime · • Every
   > access is logged
   > *Scope: all record types · last 5 years · [Customize]*
   > [ **Allow automatic sync** ] [ Not now ]

   The `[Customize]` link expands the editable scope inline (record types,
   lookback window) without leaving the screen or adding a second screen for
   the default path. Under the hood this still produces two distinct CM
   artifacts (subscription + consent) — that's an implementation detail the
   user never has to click through separately.
3. On **Allow** → subscription request and first-fetch consent request are
   both fired from this one action → CM approval handled in-app →
   **historical backfill begins asynchronously in the background** (revised
   from v0.1 — see §3.1.1; this is not a blocking wait).
4. Optional toggle, shown after the first successful fetch completes (async,
   via notification, not blocking onboarding): *"Don't ask again for routine
   syncs"* → sets patient auto-approve policy at CM for TDC requests.
5. **"Not now"** → nothing breaks; manual per-request consent remains
   available; re-prompt allowed max once, contextually (e.g., when user
   manually uploads a record: "want this to happen automatically?").

#### 3.1.1 Historical fetch is fire-and-forget (revised from v0.1)

CM fetch flows are callback-driven (§4) — a HIP may respond in seconds or
minutes, not on a fixed clock. The UI therefore does **not** wait on a timer
before offering to move on:
- The moment "Allow" is tapped, the user is returned to the app immediately
  — there is no spinner screen and no fixed-delay skip button. Continuing is
  always instantly available; there is nothing to skip *past*, because
  nothing blocks in the first place.
- The backfill job runs in the background. When it completes:
  - **Records found** → timeline updates + push notification ("New lab
    report from Ashirwad Diagnostics added to Aai's timeline").
  - **No data yet** → the existing §7 empty-state copy applies ("Sync is on.
    Records will appear here automatically...").
- Manual upload (the FAB, per `001` FR-005) is **always** present regardless
  of backfill outcome — it is never framed as a fallback shown only after a
  failed wait, it's simply always there.

### 3.2 Ongoing (invisible)
New care context at any HIP → CM notification → TDC requests under standing
policy → HIP transfers encrypted FHIR bundle → decrypt → normalize → timeline
entry + optional push ("New lab report from Ashirwad Diagnostics added to Aai's
timeline").

### 3.3 Revoke
Family & Consent screen, per profile: subscription toggle + list of active
consent artifacts with scope/expiry + "stop automatic sync" → cancels
subscription and revokes standing artifacts at CM. Local copies of
already-fetched records are retained (they're the patient's own PHR data) —
this retention is stated on the revoke screen, not hidden. See
`flowchart.md` diagram 3.

## 4. System flow

```
HIP (any facility)                ABDM Gateway / CM               TDC (HIU)
 new care context ──────────────▶ subscription notification ────▶ POST /abdm/notify (Core API webhook receiver)
                                                                    │ verify sig, dedupe, enqueue (Celery)
                                                                    ▼
                                  consent request (or standing  ◀── consent/fetch job
                                  auto-approve policy applies)
 encrypted FHIR bundle ─────────▶ data push ───────────────────▶ POST /abdm/data (data receiver endpoint)
                                                                    │ Fidelius decrypt (pyfidelius — Sprint 4 gate)
                                                                    │ validate FHIR R4 (NRCeS IG v6.5.0)
                                                                    │ map → records + structured fields
                                                                    ▼
                                                                  patient timeline (+ FCM push, + access log)
```

- All CM interaction lives in **TDC Core (Django/DRF)** — likely
  extending/forking the `care_abdm` plugin patterns rather than greenfield;
  verify what the plugin already implements for HIU/subscription flows
  before writing anything (research task, 1 session — see §9.4).
- Async everywhere: CM flows are callback-driven; Celery queues for
  notify-handling, fetch, decrypt, ingest. Idempotency keys on care-context
  IDs (HIPs re-notify; duplicates are normal).
- Fastlane is **not** in this loop. Emergency payload remains the
  denormalized snapshot; ABHA-synced data may *feed* the snapshot
  (meds/conditions) only via the normal profile-save path.

## 5. Data model additions (Core API)

`abha_links` (profile 1:1 — abha_address, abha_number, linking_token ref,
verified_ts)
`abdm_subscriptions` (profile FK, subscription_id, categories[], status:
active|revoked, granted_ts, revoked_ts)
`abdm_consents` (profile FK, artifact_id, scope json, purpose, expiry,
status, auto_approve bool)
`abdm_inbound_log` (consent FK, subscription FK, hip_id, care_context, txn_id,
received_ts, record FK)
`records` gains: `source` enum (`manual_upload` | `tdc_hms` | `abdm_sync`),
`source_hip_name` — timeline shows provenance chips.

`family_invite_codes` and `onboarding_router_answer` (from `002`) are
independent of this data model — no ABDM table references them, by design
(§2 rule 1: ABDM consent stays strictly per-patient, never routed through a
family/caregiver join).

## 6. Milestone / sprint mapping

| Piece | ABDM milestone | Sprint |
|---|---|---|
| ABHA link/verify (prereq, spec `002`) | M1 | Sprint 4 |
| Consent request + fetch + Fidelius decrypt + FHIR ingest | M3 (HIU) | Sprint 4–5 |
| Subscription (new-record notifications) | M3 subscription APIs | Sprint 5 |
| Auto-approve policy support | CM policy flows | Sprint 5 |
| TDC-as-HIP publishing (own HMS records outbound) | M2 | parallel track, separate spec |

Prereqs, hard: ~~sandbox credentials~~ **arrived 2026-08-17** · Sprint 1
tenant-isolation audit passed · pyfidelius validated against sandbox (CARE
issue #1871 path) · webhook receiver endpoints on api.thedoorstepclinic.com
with signature verification.

## 7. Expectation management (deck + UX honesty)

The subscription only surfaces records from facilities that are live HIPs —
currently a thin slice of Indian care delivery. Therefore:
- Empty-state copy after sync setup: *"Sync is on. Records will appear here
  automatically as your clinics join India's digital health network — and
  you can add anything yourself anytime."* Manual upload stays first-class
  forever.
- Deck framing: automatic sync is positioned as compounding — **TDC's free
  HMS is itself putting clinics on that network**, so the patient app's sync
  gets stronger with every clinic TDC signs. Flywheel, stated honestly.
- Never promise "all your past records instantly."
- Because manual upload never depends on ABHA or sync status, this is also
  the honest counter to any onboarding design that treats ABHA as a gate —
  the app is fully useful before, during, and after sync setup.

## 8. Copy guardrails

**Approved:** "one-time permission" · "automatic sync" · "you approve once,
you can stop anytime" · "every access is logged" · "connected to India's
ABDM network (integration in development)".

**Banned:** "auto consent" · "we fetch everything" · "no permission needed" ·
"ABDM certified" (until true) · consent language inside T&C/onboarding
legalese · any suggestion the app consents on the user's behalf · **any
phrasing implying consent was granted "on behalf of" another adult family
member** (new in v0.2 — a caregiver may *assist* a tap, per `002`, but the
copy must never describe the caregiver as the one consenting for the elder).

## 9. Open decisions

1. Default lookback for first historical fetch — 5 years proposed; Sharvari
   to confirm clinical usefulness vs. noise. (Due: Sprint 4 planning.)
2. Push notification per inbound record vs. daily digest — lean per-record
   at MVP, digest as setting. (Adi, Sprint 5.)
3. ~~Auto-approve toggle: offer at first fetch or after N manual
   approvals?~~ **Resolved in v0.2:** offered as a secondary toggle after the
   first successful fetch completes (async), not bundled into the initial
   merged Allow screen — keeps the first screen to one decision.
4. Whether `care_abdm` plugin already covers subscription flows or only
   M1/M2 — research task, blocks §4 build estimates. (Adi, 1 session, before
   Sprint 4 planning.)
5. Renewal UX at 12-month artifact expiry — silent re-request under
   auto-approve vs. explicit re-consent. Compliance lean: explicit. (Decide
   Sprint 5.)

## 10. Out of scope

Fixture-only timeline behaviour (`DEMO_MODE` seeds stay seeded) · delegated adult
consent inside ABDM (does not exist in production ABDM; family layer is
TDC's own grant model, see `001`) · minor/dependent ABHA flows (Phase 2) ·
outbound M2 publishing (separate spec) · scan-and-share QR flows (not to be
confused with `002`'s family-invite QR, which is a TDC-internal join
mechanism, not ABDM) · NHCX/claims.

---

## Review & Acceptance Checklist

### Content Quality
- [x] Consent mechanics kept strictly separate from the family/caregiver layer.
- [x] No implementation detail beyond what's needed to bound the UX (system
      flow is informative, not prescriptive of internals).
- [x] All mandatory sections completed.

### Requirement Completeness
- [x] Flow is testable against the merged-screen and fire-and-forget
      revisions.
- [x] Success criteria measurable (one screen, one tap; async backfill; no
      blocking timers).
- [ ] Default lookback window sign-off — **[NEEDS CLARIFICATION: Open
      Decision #1, owner Sharvari]**.

## Execution Status
- [x] v0.1 content parsed and preserved
- [x] Review-session decisions incorporated (merged screen, fire-and-forget,
      banned-phrase addition)
- [x] Cross-references to `002` added
- [ ] Review checklist fully passed (blocked only on lookback-window sign-off)
