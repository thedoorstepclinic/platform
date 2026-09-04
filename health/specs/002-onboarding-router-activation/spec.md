# Feature Specification: Onboarding — Identity, Emergency Card & ABHA Activation (Track B)

**Feature:** `002-onboarding-router-activation` (directory name retained for
link stability; the "router" screen it was named for is removed — see
§Supersession)
**Created:** 2026-07-15 · **Rewritten:** 2026-08-18
**Status:** Active — Track B (production). Supersedes the Track A demo login.
**Owner:** Adi (dev) / Soham (copy)
**Depends on:** [`001-tdc-phr-patient-app`](../001-tdc-phr-patient-app/spec.md) —
reuses its `profiles` / `caregiver_grants` model.
**Precedes:** [`003-abdm-sync-subscription`](../003-abdm-sync-subscription/spec.md) —
this spec ends where ABHA is linked and verified; `003` picks up from there.
**Absorbs:** Screen 1 (Splash/Login) from
[`009-utility-screens`](../009-utility-screens/spec.md) — `009` cedes S1 and
retains Health Summary, Trust, and Settings.
**Companion:** [`flowchart.md`](./flowchart.md) — diagrams need a matching pass.

> **Supersession notice.** This rewrite replaces three things from the
> 2026-07-15 draft: (1) the standalone **router question** ("Who will you use
> TDC Health for?") is removed — the same signal is now captured by the
> emergency-card step, attached to a real decision instead of an abstract one;
> (2) **Splash/Login** moves here from `009`; (3) the flow is re-ordered so the
> **emergency card is created before ABHA linking**. The assisted-add and
> family-QR mechanisms survive unchanged but move out of the onboarding spine
> (§Post-Onboarding, Not Onboarding).

> **Governance (owner direction, 2026-08-18).** **Tracks stay separate.**
> Track A completes first, then Track B's foundation, and so on. The
> constitution (`.specify/memory/constitution.md`) keeps its per-principle
> binding classes (`[A]` / `[B→A]` / `[B]`) — they are not merged. The only
> permitted crossover is **one or two individual FRs slipping from B into A**,
> named explicitly when it happens; a track boundary is not renegotiated
> wholesale. A Track B constitution still needs ratifying before compliance
> testing; until then this spec carries the existing consent and copy
> discipline forward by intent, and the ratified document wins on conflict.

---

## What this is / is NOT

**IS:** The first five minutes of TDC Health in production — from cold open to
a Home screen with real value already in it. Covers the login surface, phone
identity, emergency-card creation as the activation moment, and ABHA linking
offered as a high-value option that is never a gate.

**IS NOT:** ABDM continuous-sync mechanics (subscriptions, consent artifacts,
backfill) — that is `003`. This spec stops at "ABHA verified and linked."
Also not the card-management surfaces (revoke, scan log, test scan) — those
stay with [`007-emergency-profile-card-consent`](../007-emergency-profile-card-consent/spec.md).

---

## Clarifications

### Session 2026-08-18

- Q: When a new user makes their first card for a parent, whose ABHA should the ABHA offer try to link? → A: ABHA is strictly per-profile. Onboarding links only the **account holder's own** ABHA (whoever is holding the phone). A dependent's profile creation and that dependent's ABHA linking are separate, deferred flows — never bundled into the caregiver's signup.
- Q: How should a dependent's profile be created once onboarding is over? → A: **Bidirectional.** A caregiver can create the profile from their own app (assisted add), *or* the dependent can create it from their own app and connect outward — joined via links, verification codes, and invites in either direction.
- Q: On Home, what form should the family-add affordance take for a time-poor caregiver? → A: **Both.** A dedicated "Add family" button, plus a first-run walkthrough that explains the process simply and lets the user choose to add family or not. The walkthrough introduces the button; the button persists after.
- Q: OTP length — 4 or 6 digits? → A: **6.**
- Q: How many carousel frames on the login screen? → A: **3–4.**
- Q: Keep the splash screen, or fold it into the login screen? → A: **Keep the splash.** It stays a brand moment; the login carousel is a separate surface behind the entry field.
- Q: Custom MPIN, or platform authentication? → A: **Platform authentication only** — device PIN, phone password, or biometric, the way UPI apps do it. No custom MPIN is built, stored, or hashed. Simpler, and already habitual for Indian users.
- Q: Should the digital emergency card link be rate-limited? → A: **No rate limit on valid reads** — an emergency page must never refuse a legitimate responder. It remains rotatable and revocable.
- Q: Which external system supplies identity details for the card? → A: **Aadhaar eKYC, not ABHA.**
- Q: Who is responsible for the clinical content of the emergency card? → A: **The user, solely.** Blood group, allergies, and conditions are the user's own input and choice; no external system populates them.
- Q: Google SSO — keep or cut? → A: **Keep**, but with **no additional screen** — it sits on the existing login surface, not a separate step.
- Q: Liveness / presence verification — keep or cut? → A: ~~Keep~~ → **Cut** (reversed same session). Aadhaar eKYC alone provides identity assurance; no biometric capture is built.
- Q: Where does the card photo come from? → A: **The user chooses it themselves.** eKYC MUST NOT supply the card photo, even though its response contains one.
- Q: Transactional email provider? → A: **Self-hosted on our own server** — keeps mail in-country, consistent with "stored only in India".
- Q: Merge Track A and Track B governance? → A: **No — keep all tracks separate.** Complete Track A, then Track B's foundation, and so on. One or two FRs may slip B→A; that is the only crossover. The constitution keeps its per-principle binding classes.
- Q: 3D card, or something cheaper? → A: **2.5D via an animation framework.** The user should *feel* their emergency card during signup.
- Q: Progress bar fidelity? → A: **Basic for now**, refined after a UX study.

---

## Goal

Get a new user from tap-open to **a real emergency card that exists** in three
mandatory steps, then offer ABHA as a bonus — not a fourth step. The card is
the activation event: it is tangible, it is theirs, and it does not depend on
any external system that can fail.

### Primary User Story

As a new user I open TDC Health, see what the product actually is before I'm
asked for anything, enter my phone, verify it, and build an emergency card
watching it take shape as I type. The card is real the moment I confirm it.
Only then am I offered ABHA — and if I skip it, or it fails, I still have
everything I came for.

### The activation spine (mandatory — the progress bar covers exactly this)

| Step | Screen | Produces |
|------|--------|----------|
| 1/3 | Phone entry | — |
| 2/3 | OTP verify | `users` row — account exists at TDC |
| 3/3 | Emergency card builder | `profiles` + `emergency_profiles` + a live digital card |

**Bar completes at step 3.** ABHA is offered *after* completion, as a distinct
opportunity — never as "step 4 of 5", because a step you're allowed to skip
makes skipping feel like failure (FR-018).

---

## Acceptance Scenarios

1. **Given** a cold open with no session, **When** the login screen loads,
   **Then** the entry field is visible in the first frame and the brand
   carousel is ambient behind it — never a blocking splash, never an
   auto-advancing gate.
2. **Given** a valid existing session, **When** the app opens, **Then** the
   user goes straight to Home and never sees the carousel.
3. **Given** the phone screen, **When** I enter a number and continue,
   **Then** an OTP is sent and the progress bar reads step 1 of 3.
4. **Given** OTP verification succeeds, **When** the account is created,
   **Then** I go directly to the card builder — no router question, no
   interstitial.
5. **Given** the card builder, **When** I choose who the card is for
   (Me / My parent / Someone else), **Then** a profile is created with the
   matching `relation`, and for anyone other than "Me" a `caregiver_grants`
   row is written (`001` FR-003).
6. **Given** the card builder, **When** I type a name, age, or blood group,
   **Then** the 3D card preview updates live with what I typed — the preview
   renders from the same `emergency_payload` the responder page will serve,
   never a parallel template (`007` preview rule).
7. **Given** a completed card, **When** I confirm, **Then** the card exists
   immediately with a working link/QR, a success animation plays, and an
   email receipt is sent if an email was provided.
8. **Given** the card exists, **When** the success state resolves, **Then**
   ABHA is offered with a visible, non-punitive "Not now" — and the offer
   targets **my own** ABHA even if the card I just built was for a parent,
   creating my self profile if I don't have one yet.
9. **Given** I accept the ABHA offer, **When** the lookup runs against my
   registered mobile, **Then** the system tells me itself whether an ABHA
   exists — I am never asked "do you already have one?"
10. **Given** the mobile lookup returns more than one ABHA, **When** the
    result renders, **Then** I choose from a masked list rather than the
    system guessing.
11. **Given** ABHA verification succeeds and the verified name or DOB differs
    from what I typed on the card, **When** the conflict is detected,
    **Then** I am offered a one-tap "update card to match" — never a silent
    overwrite, never a blocking form.
12. **Given** ABHA linking completes, **When** the success screen renders,
    **Then** it links me to my own ABHA on the official portal and states
    "built on India's ABDM framework (integration in development)" — no NHA
    logo lockup, no certification claim.
13. **Given** any point after the card exists, **When** I skip or abandon
    ABHA, **Then** I land on a usable Home and am re-prompted at most once,
    contextually.
14. **Given** I abandon ABHA mid-flow (OTP screen closed, app killed),
    **When** I return, **Then** I resume from where I left off.
15. **Given** the flow completes, **When** Home loads, **Then** I am **not**
    asked who I am using the app for — that signal was already captured at
    the card step.

---

## Edge Cases

- **Mobile has multiple ABHAs** (common — families register several under one
  number) → disambiguation list with masked names; never auto-select.
- **Mobile has zero ABHAs** → fall through to Aadhaar entry; exists-check runs
  there.
- **Aadhaar OTP goes to a number the user doesn't control** (typical for an
  elder's profile set up by a caregiver) → ABHA linking fails gracefully;
  the card and profile are unaffected; `abha_no` stays free text per `001`.
  This is *why* a dependent's ABHA is never attempted during the caregiver's
  onboarding (FR-014b) — the OTP would have nowhere to land.
- **Caregiver tries to link their own ABHA to a parent's profile** → blocked
  outright (FR-014a). Not a warning, not an override; one ABHA belongs to one
  profile.
- **Both sides initiate at once** (caregiver sends an invite while the family
  member sends a request) → the connection resolves once, not twice; the
  second action collapses into accepting the first.
- **ABHA returns a name that contradicts the card** → offer to update, never
  auto-apply. Blood group is never touched by ABHA — see FR-016.
- **User has no email** → skip the receipt silently; email is optional
  everywhere and is never a login identity.
- **OTP resend abused** → cooldown + attempt cap; lockout copy must not
  confirm whether the number is registered.
- **Card confirmed while offline** → card creation is server-side; queue and
  retry, show an honest pending state. Do **not** fabricate a local card that
  the responder page cannot serve.
- **Photo capture denied/unavailable** → card is valid without a photo; the
  responder page has never depended on one.

---

## Screens

### S0 — Splash (retained)

A brand moment, not a gate. Logo + wordmark, **under 1 second**, session check
runs behind it. A valid session goes straight to Home and never sees S1. No
taglines, no marketing copy — the carousel on S1 does that job.

### S1 — Login (absorbed from `009`)

- **Brand carousel** occupying the upper band: **3–4 frames** of photography
  covering TDC services and products. Rules — **static first frame**, no
  auto-advance faster than 5s, swipeable, pauses on interaction, and
  **respects reduce-motion**. Ambient context, never a wall.
- **Entry point visible in frame one**: phone number field + primary Continue.
- **Google sign-in sits on this same surface** — one button, no additional
  screen, no separate step. It is a convenience credential, not a second
  identity (FR-004a).
- ToS / Privacy line applies to **terms only**. It is not the vehicle for any
  health-data or ABDM consent (§What this spec rejects).
- **A valid session never renders this screen.**

### S2 — OTP verification

**6-digit** numeric entry, per-digit boxes ≥48dp, resend with visible
cooldown, back-navigation preserved. Progress: **1 of 3 → 2 of 3**.

### S3 — Emergency card builder (the activation moment)

The screen this product is really about.

- **"Who is this card for?"** — *Me · My parent · Someone else*. One tap.
  This replaces the removed router: same persona signal, attached to a real
  decision. Sets `profiles.relation` and drives Home's featured action.
- **2.5D card canvas** — the card renders with perspective, depth, and motion
  via an animation framework, and the user types **directly onto it**, seeing
  exactly what a responder will see. The goal is tactile: the user should
  *feel* the card being made, not fill in a form. Full 3D is explicitly not
  required — 2.5D buys the effect at a fraction of the build (NFR-006).
- **Fields:** blood group · allergies · name · age/DOB · photo (optional,
  **user-chosen** — never pulled from eKYC). Ordered by responder priority,
  not form convention: **blood group first.**
- **Optional email**, asked here and only here, with a concrete reason:
  *"Email me a copy of this card."* Recovery use is stated, not buried. If the
  user signed in with Google, the email is already known — prefill it and ask
  only for confirmation.
- **Confirm → card is created immediately** and a flip/reveal animation
  resolves to the live card.

**Why these fields:** Aadhaar eKYC returns name, DOB, gender, address, and
photo. It returns **no blood group, no allergies, no conditions** — and
neither does ABHA. Those are the fields that actually save a life on the
responder page, and they exist only because the user typed them.

**Clinical content is the user's sole responsibility (FR-013a).** Blood group,
allergies, and conditions are the user's own input and choice. No external
system populates them, no system infers them, and the app makes no claim that
they are verified. This must be stated plainly in-product — it is both a
liability boundary and an honesty commitment (Principle VI).

### S4 — ABHA activation (offered, not a step)

**Scope: the account holder's own ABHA, and nothing else.** ABHA is a
per-patient identity; the phone that just verified belongs to the person
signing up, so that is the only ABHA this flow may link. If the first card was
built for a parent or someone else, this offer still links the *signer's* ABHA
to the *signer's* self profile — creating that self profile if it does not
exist yet. A dependent's ABHA is linked later, from that dependent's own
profile, using their own Aadhaar and their own mobile OTP (§Post-Onboarding).

Presented after the progress bar completes, as an offer card with a visible
"Not now". Sub-flow:

1. **Silent lookup** of the registered mobile against ABDM.
2. **Exactly one ABHA found** → "We found an ABHA on this number" → Aadhaar
   OTP to prove ownership → link.
3. **Multiple found** → masked disambiguation list → then OTP → link.
4. **None found** → Aadhaar number entry → OTP → exists-check → link or
   create.
5. **Prefill** verified name / DOB / gender / photo onto the card, with
   conflict resolution per FR-015.
6. **Consent to connect the PHR** is its own explicit screen — never bundled,
   never pre-checked.

### S5 — Success & app lock

- States both outcomes plainly: card is live, ABHA is linked (or not — and
  that's fine).
- Links the user to **their own ABHA** on the official portal. Factual and
  useful.
- **Offers app lock via platform authentication** — device PIN, phone
  password, or biometric, exactly as UPI apps do it. Optional, dismissible,
  offered here at the end and never mid-flow.
- → Home.

**No custom MPIN is built.** The app delegates entirely to the OS credential
prompt (Android `BiometricPrompt` with device-credential fallback). Nothing is
stored, hashed, or recovered by TDC. This is simpler, removes a whole class of
credential-storage risk from the WASA surface, and matches the habit Indian
users already have from UPI.

---

## Security & Compliance Constraints (Track B — binding)

### Aadhaar number handling
- **Never persisted.** No column, no cache, no shared-preferences write.
  Transit-only, to the ABDM gateway, server-side.
- **Never logged.** Excluded from access logs, error logs, APM traces, and
  crash reports. A scrubber must be in place, not merely a convention.
- **Masked on entry** once the field loses focus (`XXXX XXXX 7237`).
- Rationale: this is the highest-frequency WASA failure class
  (`docs/compliance-baseline.md` §3) and a single critical finding stalls
  certification.

### Digital card token (new threat model)
The NFC chip's safety comes from CMAC + a monotonic counter. A digital card
is a URL — **a bearer token**, and a screenshot of it is permanent access
unless designed otherwise. Therefore:
- Long, high-entropy, unguessable token; **not** derived from profile id.
- **User-rotatable** (invalidates the old link) and **revocable**, reusing
  `007`'s one-toggle revocation pattern. Rotation and revocation are the
  entire defense — design them to be obvious and one-tap.
- **No rate limit on reads of a valid token.** An emergency page that refuses
  a legitimate responder has failed at its only job. Abuse protection instead
  applies to **invalid** token attempts (per-IP caps on failures), which stops
  enumeration without ever throttling a real scan.
- Same accountability as a chip scan: **access logged, family notified.**
- `X-Robots-Tag: noindex`, no PHI in the query string — same rules Fastlane
  already applies (`contracts/fastlane-api.md` §Public-page hardening).

### Auth provider — unresolved, with a residency constraint

Owner is considering **Firebase** for auth and email, "depending on price and
compliance rules." Two facts the decision needs:

- **Data residency.** "Stored only in India" is approved copy and a Principle
  VI commitment, and ABDM/DPDP both care where identity data lives. Firebase
  Authentication / Identity Platform does **not** guarantee India-resident
  storage of auth records the way the locked DigitalOcean-Bangalore stack
  does. Adopting it for auth means either dropping that claim from all copy or
  proving residency — and the claim is already printed on the Trust screen.
  FCM is a different matter and stays fine: it carries push tokens, not
  identity records.
- **Firebase does not send transactional email.** It sends only its own
  auth-template mail (verification, password reset). FR-012's card receipt
  needs real SMTP — SES, SendGrid, Postmark, or the Firebase Trigger Email
  extension, which requires you to supply SMTP credentials anyway. Firebase
  does not remove this dependency; it only relocates it.

Adopting Firebase Auth also displaces **Django/DRF + SimpleJWT**, a locked
stack decision. That is a `CLAUDE.md` change with a changelog entry, not an
implementation detail.

### Carried forward from Track A
- **Principle X:** every new endpoint derives its queryset from the caller's
  own profiles ∪ unrevoked `caregiver_grants`. No `objects.all()`.
- **Principle XI:** one `access_logs` row per PHI access, same transaction.
- **App lock** delegates to platform authentication. No credential is stored
  by TDC, so there is no credential store to harden or breach.

### Liveness / presence verification — cut

**Not built.** Identity assurance comes from **Aadhaar eKYC only**.

Reasons: liveness output is biometric data, a heavier DPDP category than a
stored photo, carrying its own notice, purpose-limitation, and retention
obligations. It requires a third-party SDK, since a hand-rolled check is
trivially defeated and worse than none — it invites reliance it cannot
support. And it does not address the real risk: for an emergency card the
danger is a **wrong blood group**, not an impostor at signup. Aadhaar OTP plus
eKYC already provides stronger identity assurance at no extra build cost.

**Consequence for the card photo:** the user chooses their own photo (FR-013a
extended below). eKYC's response contains a photo, and the app deliberately
does not use it.

---

## What this spec explicitly rejects

**Pre-checked "agree to all" consent boxes** are not used anywhere ABDM or
health-data consent is involved. No checkbox defaults to checked; nothing is
folded into a "by continuing you agree" clause. ABDM's Consent Manager
framework requires explicit, reviewable, per-artifact approval, and DPDP's
affirmative-action bar rules out default-on consent for health data. The
login screen's terms line is acceptable **for terms only** — it carries no
health-data consent.

Perceived speed comes from **merging screens, not defaults**: one real screen,
one real tap, visible and editable scope. See `003` §2 for the sync-permission
screen.

**Also rejected:** the NHA logo as a trust signal. Displaying NHA branding at
a success moment implies endorsement and drifts into "ABDM certified", which
is banned copy and factually untrue — production ABDM access is four gates and
6–9 months away (`docs/compliance-baseline.md` §2).

---

## Post-Onboarding, Not Onboarding

These survive from the previous draft but are **not** in the activation spine.
Onboarding activates *one* person; Home is where a family gets built (`001`
§4.1 Loop 1).

### How family-add surfaces on Home

Two affordances, and they do different jobs:

- **A first-run walkthrough** explains family profiles and linking in plain
  language and lets the user **choose to add family or not**. It runs once,
  is dismissible, never repeats, and its job is to introduce the button.
- **A dedicated, persistent "Add family" action** on Home. This is the durable
  affordance — always there for the moment a time-poor caregiver finally has
  one. It is an *action*, not a content slot, so it does not compete with
  Home's quiet (Principle VIII, non-negotiable).

The per-profile setup chips from `001` §4.1 remain for per-profile state
(records, meds, emergency profile, card). The walkthrough and button are
app-level; the chips are profile-level. `001` §4.1 needs amending to say so.

**Linking is bidirectional.** A family connection can originate from either
side — the caregiver reaching toward a parent, or an adult family member
reaching outward from their own app. Both directions use the same primitives:
a short-lived link, a verification code, or an invite. Neither direction is
privileged, and neither requires the other person to have signed up first.

| Path | Direction | Mechanism |
|------|-----------|-----------|
| **Assisted add** | Caregiver → dependent with no device (Asha, Prakash) | Caregiver fills name / relation / DOB; profile added under the caregiver's account; `caregiver_grants` row written — the consent moment (`001` FR-003). |
| **Invite out** | Caregiver → adult with their own phone | Caregiver generates a short-lived code/QR/link → invitee does their own phone+OTP → enters code → attached to the family group with their **own** account, own ABHA, own consent. |
| **Request in** | Adult family member → caregiver | The family member initiates from their own app, sending a link/code the caregiver accepts. Same result as Invite out; only the initiator differs. Approval is always required from the receiving side. |

Assisted add creates a profile the caregiver manages. Invite-out and
request-in both create (or connect) an **independent account** merely *linked*
for shared visibility — never a `caregiver_grants` relationship, and never a
shared ABHA.

**ABHA stays per profile in every one of these paths.** An assisted-add
profile starts with no ABHA and gains one only through that person's own
Aadhaar and mobile OTP; an invited or requesting account brings its own. No
path lets one person's ABHA attach to another person's profile.

---

## Requirements

### Functional Requirements

**Login & identity**
- **FR-001**: Splash MUST resolve in under 1 second, MUST carry no tagline or
  marketing copy, and MUST perform the session check behind it. The login
  screen that follows MUST render its entry field in the first frame with the
  brand carousel ambient behind it.
- **FR-002**: The carousel MUST be 3–4 frames, MUST start on a static frame,
  MUST NOT auto-advance faster than 5s, MUST be swipeable, and MUST honor
  reduce-motion.
- **FR-003**: A valid session MUST bypass the login screen entirely.
- **FR-004**: A verified phone number MUST be the canonical account identity.
  Every account MUST have one, regardless of how the user first signed in.
- **FR-004a**: Google sign-in MUST be offered on the login surface itself with
  **no additional screen or step**. It is a linked credential, never a second
  account: a Google sign-in that resolves to an existing verified phone MUST
  attach to that account, and a Google-first signup MUST still complete phone
  verification before an account exists.
- **FR-005**: Email MUST be optional, MUST be collected only at the card step
  with a stated purpose, and MUST NOT function as a login identity. When
  Google sign-in supplied an email, it MUST be prefilled for confirmation
  rather than re-typed.
- **FR-006**: OTP MUST be **6 digits**, MUST enforce a resend cooldown and an
  attempt cap, and its failure copy MUST NOT reveal whether a number is
  registered.

**Activation spine**
- **FR-007**: The progress indicator MUST cover exactly the three mandatory
  steps (phone → OTP → card) and MUST reach completion when the card is
  created.
- **FR-008**: There MUST NOT be a standalone router question at any point in
  onboarding, and Home MUST NOT ask who the app is for.
- **FR-009**: The card step MUST capture who the card is for (Me / My parent /
  Someone else), MUST set `profiles.relation` from it, and MUST write a
  `caregiver_grants` row for any answer other than "Me".
- **FR-010**: The card preview MUST render from the same `emergency_payload`
  snapshot the responder page serves — never a parallel template (`007`).
- **FR-011**: The card builder MUST order fields by responder priority, with
  **blood group first**.
- **FR-012**: On confirmation the system MUST create a live digital emergency
  card immediately, reachable by link/QR, and MUST send an email receipt when
  an email was provided.
- **FR-013**: A card MUST be valid without a photo.
- **FR-013a**: Blood group, allergies, and conditions MUST come solely from
  the user's own input. No external system may populate, overwrite, or infer
  them, and the product MUST NOT present them as verified.
- **FR-013b**: The card canvas MUST render in 2.5D with live typing directly
  on the card. Full 3D is not required.
- **FR-013c**: The card photo MUST be chosen by the user. The system MUST NOT
  populate it from Aadhaar eKYC, and MUST NOT perform liveness or any other
  biometric capture.

**ABHA**
- **FR-014**: ABHA MUST be offered only after the progress bar completes, as a
  skippable offer — never as a numbered step.
- **FR-014a**: An ABHA MUST be linked to exactly one profile — the profile of
  the person that ABHA belongs to. The system MUST NOT allow one person's ABHA
  to be attached to another person's profile under any flow.
- **FR-014b**: The onboarding ABHA offer MUST target only the account holder's
  own profile, creating that self profile if the first card was built for
  someone else. A dependent's ABHA MUST be linked only from that dependent's
  own profile, using their own Aadhaar and their own mobile OTP.
- **FR-015**: The system MUST determine ABHA existence itself. The user MUST
  NOT be asked "do you already have an ABHA?"
- **FR-016**: ABHA lookup MUST run against the registered mobile first; when
  it returns more than one account the user MUST choose from a masked list,
  and the system MUST NOT auto-select.
- **FR-017**: Verified **Aadhaar eKYC** data — not ABHA — MAY prefill name,
  DOB, and gender **only**. It MUST NOT supply the card photo (FR-013c), and
  MUST NOT overwrite blood group, allergies, conditions, or emergency contacts
  under any circumstance (FR-013a).
- **FR-018**: A conflict between verified ABHA data and user-typed card data
  MUST surface a one-tap reconciliation. Silent overwrite is prohibited.
- **FR-019**: Aadhaar-linked-mobile OTP MUST be verified before any
  exists/create branch is evaluated (ABDM's contract, not a UX choice).
- **FR-020**: Consent to connect the PHR MUST be its own explicit screen, and
  no consent-adjacent control anywhere in onboarding MUST be pre-checked.
- **FR-021**: Interrupted ABHA linking MUST be resumable from Home, not
  restarted.

**Family connection (post-onboarding, not in the spine)**
- **FR-021a**: The system MUST support three family-connection mechanisms:
  assisted add (caregiver-entered profile), invite-out (caregiver initiates),
  and request-in (the family member initiates from their own app).
- **FR-021b**: Invite-out and request-in MUST be symmetric in outcome — only
  the initiator differs — and both MUST require explicit approval from the
  receiving side before any connection exists.
- **FR-021c**: Invite/request codes and links MUST be short-lived and
  single-use, and MUST display the initiator's identity to the recipient
  before acceptance.
- **FR-021d**: An accepted invite-out or request-in MUST create or connect an
  independent account, never a `caregiver_grants` relationship and never a
  shared ABHA.

**Skip, lock, exit**
- **FR-022**: Every optional step MUST offer a visible, non-punitive skip that
  leads to a usable Home.
- **FR-023**: A skipped optional step MUST be re-offered at most once,
  contextually — never as a repeat blocking prompt.
- **FR-024**: App lock MUST use **platform authentication** (device PIN, phone
  password, or biometric). TDC MUST NOT build, store, hash, or recover a
  custom MPIN. It MUST be offered at the end of the flow or later, MUST be
  optional, and MUST NOT appear inside the activation spine.
- **FR-024a**: Home MUST carry a dedicated, persistent **"Add family"** action
  — an action affordance, not a content slot (Principle VIII).
- **FR-024b**: A **first-run walkthrough** MUST explain family profiles and
  linking in plain language, MUST let the user choose to add family or not,
  and MUST introduce the persistent Add-family action. It MUST be dismissible
  and MUST NOT repeat once completed or skipped.
- **FR-025**: The success screen MUST link to the user's own ABHA on the
  official portal and MUST NOT display NHA branding as an endorsement.

**Security**
- **FR-026**: Aadhaar numbers MUST NOT be persisted, MUST NOT appear in any
  log, trace, or crash report, and MUST be masked on entry blur.
- **FR-027**: Digital card links MUST be high-entropy, user-rotatable, and
  revocable, and MUST produce the same access-log and family-notification
  behavior as a chip scan.
- **FR-027a**: Reads of a **valid** card token MUST NOT be rate-limited — an
  emergency page may never refuse a legitimate responder. Abuse protection
  MUST instead cap **invalid** token attempts per source, which blocks
  enumeration without throttling real scans.

### Non-Functional Requirements
- **NFR-001 (Activation):** primary metric is % of signups reaching **a live
  emergency card** in the first session. Card creation — not ABHA linking —
  is the activation event.
- **NFR-002 (Resumability):** all multi-step sub-flows persist progress per
  profile.
- **NFR-003 (Accessibility):** elderly-first standard from `001` NFR-004
  applies throughout; the phone is often handed to the elder mid-flow. Card
  builder text remains legible while the 3D transform is applied.
- **NFR-004 (Time to card):** cold open → live card achievable in under 90
  seconds, excluding OTP delivery latency.
- **NFR-005 (Honest offline):** no step may present a created card that the
  server has not actually created.
- **NFR-006 (Build ceiling):** the card animation must be achievable inside a
  solo-dev, ~4h/day budget. 2.5D is chosen over full 3D on exactly this
  ground — the felt effect matters, the dimensionality does not.
- **NFR-007 (Progress fidelity):** the progress indicator ships **basic** and
  is refined after a UX study rather than over-designed up front.

---

## Key Entities (changes to `001`'s model)

**`users` gains `email`** — nullable, unique-if-present. Recovery and receipts
only, never a login identity. Directly addresses the account-recovery gap in
`docs/track-b-backlog.md` §1.

**`cards` splits into card identity vs chip binding.** The current table
conflates them (`uid` as PK), which the digital-first decision makes
untenable:

| Table | Fields | Notes |
|---|---|---|
| `cards` | `id` PK · `profile_id` · `token` (digital link secret) · `active` · `created_at` · `rotated_at` | The emergency card as a product object. Exists from onboarding. |
| `card_chips` | `uid` PK (NTAG 424) · `card_id` FK · `sdm_key_ref` · `last_ctr` · `bound_at` | The physical chip, bound later. Nullable relationship — a card may have no chip. |

`emergency_payload` keys off `card_id` rather than `card_uid`. Fastlane's
`GET /e/{uid}` resolves chip → card → payload and is otherwise unchanged;
the digital card needs its own token-authenticated path with the security
properties in §Security.

**`onboarding_router_answer` is deleted.** The persona hint now derives from
the first profile's `relation`, set at the card step. One fewer stored
concept, and a more reliable signal than self-report.

**`family_invite_codes` gains a direction.** Bidirectional linking means the
same table serves both an invite sent outward and a request sent inward:
`direction` (`invite_out` / `request_in`) · `initiator_profile_id` ·
`accepted_by` (nullable) · `expires_at` · `single_use`. Approval always comes
from the non-initiating side.

**ABHA uniqueness is a constraint, not a convention:** `profiles.abha_no` (and
its linked identity) MUST be unique across profiles — one ABHA, one profile
(FR-014a). Enforce in the database, not only in application code.

**Retained:** `profiles.abha_status`.

---

## Copy Guardrails (in addition to `001` §Copy Guardrails)

**Banned, additionally:**
- Any consent control described or implemented as pre-checked / opt-out.
- "Do you already have an ABHA?" as a user-facing question.
- NHA logo, seal, or wordmark used as a trust or endorsement signal.
- Any framing of the digital card link as equivalent in security to the chip.

**Required framing:** the card is a capability token exercising a standing
grant. Neither a chip tap nor a link open is consent (`001` Principle II).

---

## Open Decisions

| Decision | Owner | Notes |
|---|---|---|
| **Auth provider — Firebase vs. Django/SimpleJWT** | Adi | Locked stack is Django/DRF + SimpleJWT. Firebase Auth replaces it *and* raises data-residency exposure against "stored only in India". Self-hosted mail (below) already argues for staying in-country. |
| **Self-hosted mail — port 25 + deliverability** | Adi | DigitalOcean blocks outbound port 25 on new droplets by default; needs a support request. Then SPF + DKIM + DMARC and warmed IP reputation, or receipts land in spam. Resolvable, but it is ops work, not a config line. |
| Carousel content — which 3–4 frames | Soham | Must not imply capabilities that don't exist yet. |
| Physical chip fulfilment — on-demand vs. marketing hook | Adi/Soham | Digital-first confirmed. No fulfilment state machine yet; `card_chips` must not preclude one. |
| Track B constitution ratification | Adi | Tracks stay **separate**; binding classes retained. Ratify before compliance testing. |
| Animation framework for the 2.5D card | Adi | Respect the 4h/day ceiling. |

---

## Downstream edits this rewrite requires

| File | Change |
|---|---|
| `009-utility-screens/spec.md` | Cede Splash + Login (S1); drop its FR-001/FR-002; retitle to Health Summary / Trust / Settings. |
| `001-.../spec.md` §4.1 | Router reference → card-step persona signal; add app-level walkthrough + Add-family button alongside profile-level chips. |
| `001-.../data-model.md` | `cards` split; `emergency_payload` re-key; `users.email`; Google credential link; ABHA uniqueness constraint. |
| `007-emergency-profile-card-consent/spec.md` | Card manager must handle a chipless card; add link rotation; no rate limit on valid reads. |
| `008-navigation-app-shell/spec.md` | Persistent Add-family action + first-run walkthrough must fit hub-and-spoke without a tab bar. |
| `002/flowchart.md` | All three diagrams redrawn — router node gone, card step added, ABHA scoped to account holder. |
| `health/CLAUDE.md` | Track B activation, cards model, `users.email`, Google credential, platform-auth app lock, screens list, auth-provider decision if Firebase wins, changelog entry. |
| `.specify/memory/constitution.md` | Ratify a Track B constitution before compliance testing. Binding classes **retained** — tracks are not merged. |
| `seed_demo.py` | Same-day rule on any model change. |

---

## Review & Acceptance Checklist

### Content Quality
- [x] Focused on the first-five-minutes experience, not implementation.
- [x] Explicitly scoped against `003` (no sync mechanics) and `007` (no card management).
- [x] All mandatory sections completed.

### Requirement Completeness
- [x] Requirements testable (3-step spine, no router, no silent overwrite, no persisted Aadhaar).
- [x] Success criteria measurable (NFR-001 card-creation activation, NFR-004 90s).
- [x] Security constraints stated as requirements, not prose (FR-026, FR-027).
- [x] OTP length confirmed — 6 digits.
- [x] Liveness resolved — cut; Aadhaar eKYC only.
- [x] Card photo source resolved — user-chosen, never eKYC.
- [x] Transactional email resolved — self-hosted on our own server.
- [ ] Auth provider resolved against data residency — **[NEEDS CLARIFICATION: Firebase vs. SimpleJWT]**.

## Execution Status
- [x] User description parsed
- [x] Scenarios defined
- [x] Requirements generated
- [x] Entities identified
- [ ] Review checklist fully passed (blocked on OTP length + email provider)
- [ ] Downstream edits applied
