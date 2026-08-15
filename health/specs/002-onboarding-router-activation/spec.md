# Feature Specification: Onboarding — Router, Activation & ABHA Linking (Track B)

**Feature Branch:** `002-onboarding-router-activation`
**Created:** 2026-07-15
**Status:** Draft — Track B (post-funding). Does not block Track A demo scope.
**Owner:** Adi (dev) / Soham (copy)
**Depends on:** [`001-tdc-phr-patient-app`](../001-tdc-phr-patient-app/spec.md) —
reuses its `profiles` / `caregiver_grants` model.
**Precedes:** [`003-abdm-sync-subscription`](../003-abdm-sync-subscription/spec.md) —
this spec ends where ABHA is linked and verified; 003 picks up from there.
**Companion:** [`flowchart.md`](./flowchart.md) — the diagrams this spec describes.

> **Note on governance:** the Track A constitution (`.specify/memory/constitution.md`)
> is scoped to Track A only. This spec carries forward its consent and copy-discipline
> spirit by intent, pending a Track B constitution ratification. Where this spec
> and a future Track B constitution conflict, the ratified constitution wins.

## What this is / is NOT

**IS:** The design of the first five minutes of TDC Health — from opening the
app to landing on a Home screen with at least one meaningful thing in it —
including how the app learns who is signing up, how family members get added
(assisted or self-serve), and how ABHA linking is offered without ever gating
core value on it.

**IS NOT:** The ABDM continuous-sync mechanics (subscriptions, consent
artifacts, backfill) — that is `003`. This spec stops at "ABHA verified and
linked to profile."

---

## Goal

Get a new user from tap-open to a **populated, useful Home in 4 taps or
fewer**, correctly identify whether they're onboarding themselves, a family,
or someone else, and offer ABHA linking as a high-value option — never a
gate — weighted by who's actually holding the phone.

### Primary User Story
As a new user, I open TDC Health, verify my phone, answer one question about
who I'm here for, and by the time I reach Home I already have something real
in the app — my own record, a family member's profile, or a linked ABHA —
without having faced more than one decision screen at a time.

### Acceptance Scenarios
1. **Given** a fresh install, **When** I complete phone+OTP, **Then** I see
   exactly one question ("Who will you use TDC Health for?") before anything
   else is asked of me.
2. **Given** I answer "Just me", **When** I reach Home, **Then** my own
   profile already exists and the featured first action is record upload or
   ABHA creation — not a family-add prompt.
3. **Given** I answer "Me and family", **When** I reach Home, **Then** I am
   prompted to add my first family member, and doing so writes a
   `caregiver_grants` row for that relationship (per `001` FR-003).
4. **Given** I answer "Setting up for someone else", **When** I proceed,
   **Then** I go straight into guided assisted setup for that person's
   profile — no router-style branching mid-flow.
5. **Given** I choose to add a family member who has their own phone,
   **When** I generate a family code/QR, **Then** they can join by entering
   phone+OTP on their own device plus the code — their account and any ABHA
   they link remain entirely their own.
6. **Given** I choose "Create or link ABHA" for a profile, **When** I enter
   an Aadhaar number and complete the mobile OTP, **Then** the system tells
   me automatically whether an ABHA already exists for that Aadhaar and
   either links it or creates a new one — I am never asked "do you already
   have one?"
7. **Given** any screen in this flow, **When** I choose "Skip" or "Not now",
   **Then** nothing breaks, I land on a usable Home, and I am re-prompted
   contextually at most once later.
8. **Given** I abandon ABHA linking mid-flow (e.g., OTP screen closed),
   **When** I return later, **Then** I resume from where I left off rather
   than restarting.

### Edge Cases
- Aadhaar OTP goes to a mobile number the elder profile's caregiver doesn't
  control → ABHA linking fails gracefully; profile creation is unaffected;
  the field stays manual (falls through to `abha_no` free text per `001`).
- Family code is entered by someone not actually related → code is
  single-use / time-boxed and shows the inviter's name before joining so the
  invitee can bail.
- User answers the router question, then later needs a different mode (e.g.,
  "Just me" user later adds a parent) → router answer is a **flow hint at
  signup only**, not a permanent role; adding a family member later works
  identically regardless of the original answer.

---

## The router

One screen, right after OTP verification, three options, single tap:

> **Who will you use TDC Health for?**
> 1. **Just me**
> 2. **Me and my family** — *I help manage their health*
> 3. **I'm setting this up for someone else** — *parent, grandparent*

This is the only branching decision in the entire flow. It sets which
first-action is featured on Home (see below) and, for option 3, skips
straight to assisted setup instead of a second "add family member" prompt.
It does **not** create a stored "role" on the account — it only orders what
Home shows first.

---

## Family member add — two paths

| Path | Who it's for | Mechanism |
|------|---------------|-----------|
| **Assisted add** | A family member with no device of their own in the app (the elder personas — Asha, Prakash) | Caregiver fills name / relation / DOB directly; profile is added under the caregiver's account; `caregiver_grants` row written (the consent moment, per `001` FR-003). |
| **Family QR / code join** | An adult family member with their own phone who wants their own account | Caregiver generates a short-lived code/QR from the Family screen → invitee does their own phone+OTP → enters the code → is attached to the family group with their **own** account, own ABHA, own consent. Nothing about their data or ABDM consent routes through the caregiver. |

The two paths are not interchangeable: assisted add creates a profile the
caregiver manages; QR/code join creates an independent account that is
merely *linked* to the family for shared visibility (e.g., appearing in the
Family & Consent screen), never a `caregiver_grants` relationship.

---

## Home: persona-weighted first actions, never a gate

Every path converges on the same Home screen. What's **featured** differs by
router answer — this is the "don't let the user see an empty app" lever,
played honestly instead of forced:

| Router answer | Featured first action | Why |
|---|---|---|
| Just me | Upload a record **or** create/link ABHA | Self-users are the ABHA-ready segment — own Aadhaar-linked phone in hand. |
| Me and family | Add first family member | The emotional hook for a caregiver is a family member's card appearing on Home. |
| Setting up for someone else | (already mid-assisted-setup) — next featured action is that profile's emergency basics | Serve the mission they came with. |

**Fast-follow for "Just me" (refinement):** even self-focused signups tend to
be motivated by a family member's health, not their own — so once a "Just
me" user completes their featured first action (an upload or an ABHA link),
Home's *next* nudge becomes "Add a family member," same as the "Me and
family" path. This is a sequencing refinement, not a replacement: the
router still weights what's featured *first*; it does not collapse to one
universal prompt for everyone regardless of path.

Rules that keep this from becoming choice overload:
- **Max 3 options ever shown at once.**
- **"Skip" is always visible, never penalized**, and never re-asked
  immediately — a quiet Home chip carries the option forward instead.
- **Hard cap: 4 taps from OTP-verified to a useful Home**, on every path.

---

## ABHA linking sub-flow

Embedded as one field + action **on the existing profile/patient-details
form** — not a separate multi-screen wizard.

1. Enter Aadhaar number (on the profile's details screen).
2. OTP to the Aadhaar-linked mobile. **This step is mandatory and cannot be
   skipped or reordered** — it is ABDM's own API contract for Aadhaar-based
   ABHA lookup/creation, not a UX choice.
3. On OTP success, the system queries ABDM: does an ABHA already exist for
   this Aadhaar?
   - **Yes** → link the existing ABHA to this profile.
   - **No** → create a new ABHA for this profile.
4. Profile is now `abha: verified`. (See `003` for what happens next —
   sync-permission is a separate, later screen, never bundled into this one.)

The user is never asked "do you already have an ABHA?" — the system answers
that question itself. This removes a decision most users can't reliably
answer.

---

## What this spec explicitly rejects

**Pre-checked "agree to all" consent boxes**, of the kind common in
mainstream consumer apps, are **not used anywhere ABDM consent is involved**.
No checkbox defaults to checked; nothing is bundled into a "by continuing you
agree" clause. This isn't a style preference — ABDM's Consent Manager
framework requires an explicit, reviewable, per-artifact approval, and DPDP's
affirmative-action bar rules out default-on consent for health data.
Perceived speed is instead achieved by **merging screens, not defaults**: one
real screen, one real tap, visible and editable scope. See `003` §2 for how
this applies to the sync-permission screen specifically.

---

## Requirements

### Functional Requirements
- **FR-001**: The system MUST show exactly one router question after
  OTP verification, before any other onboarding prompt.
- **FR-002**: The router answer MUST determine only the *featured first
  action* on Home — it MUST NOT be stored as a permanent, exclusive account
  role.
- **FR-003**: Selecting "Me and family" or "Setting up for someone else"
  MUST lead to profile creation that writes a `caregiver_grants` row (reusing
  `001`'s model) whenever the resulting profile is assisted-managed.
- **FR-004**: The system MUST offer two distinct family-member-add
  mechanisms: assisted add (caregiver-entered) and family QR/code join
  (self-serve, own account).
- **FR-005**: A family QR/code MUST be short-lived / single-use and MUST
  display the inviter's identity to the invitee before joining.
- **FR-006**: Joining via family code MUST create an independent account for
  the invitee — never a `caregiver_grants` relationship.
- **FR-007**: Every onboarding screen with an optional step MUST offer a
  visible, non-punitive "Skip"/"Not now" that leads directly to a usable
  Home.
- **FR-008**: A skipped optional step MUST be re-offered at most once,
  contextually (i.e., at the moment it becomes relevant), never as a repeat
  blocking prompt.
- **FR-009**: ABHA linking MUST be reachable as one field + action on the
  profile/patient-details form, not a separate wizard entry point.
- **FR-010**: ABHA linking MUST require Aadhaar-linked-mobile OTP
  verification before any exists/create branch is evaluated.
- **FR-011**: On OTP success, the system MUST query ABDM for an existing
  ABHA on that Aadhaar and automatically link (if found) or create (if not)
  — the user MUST NOT be asked to self-report whether they have one.
- **FR-012**: No consent-adjacent checkbox anywhere in onboarding MUST be
  pre-checked by default.
- **FR-013**: Interrupted ABHA linking MUST be resumable from Home on
  return, not restarted from scratch.
- **FR-014**: The path from OTP-verified to a usable Home MUST NOT exceed 4
  taps on any router path.

### Non-Functional Requirements
- **NFR-001 (Activation):** primary success metric is the % of signups
  reaching "1 profile + 1 meaningful item (record, med, emergency basics, or
  ABHA link) in the first session."
- **NFR-002 (Resumability):** all multi-step sub-flows (ABHA linking
  especially) persist progress per profile.
- **NFR-003 (Accessibility):** router and assisted-setup screens meet the
  same elderly-first standard as `001` (large type/tap targets), since the
  phone may be handed to the elder mid-flow.

### Key Entities (additions to `001`'s model)
- **family_invite_codes** — code/QR value, inviter profile, expiry,
  single-use flag, consumed_by (nullable).
- **onboarding_router_answer** — account-level, non-exclusive hint
  (`self` / `family` / `assisted`), used only to weight Home's featured
  action; not a role.
- `profiles.abha_status` — enum: `none` / `aadhaar_otp_pending` / `verified`
  / `linked` / `created` (see `003` for what follows `linked`/`created`).

---

## Copy Guardrails (onboarding-specific, in addition to `001` §8)

**Banned, additionally:**
- Any consent checkbox described or implemented as pre-checked / opt-out.
- "Do you already have an ABHA?" as a user-facing question (the system
  determines this, not the user).
- Treating the router answer as a label shown back to the user as their
  fixed "role" (e.g., never show "Account type: Caregiver" as if permanent).

---

## Open Decisions
| Decision | Owner | Notes |
|---|---|---|
| Exact copy for the three router options | Soham | Must read naturally in Hindi/Marathi translation too, not just English. |
| Family code expiry window | Adi | Propose 24h; confirm against real caregiver latency (elder's phone often handed over hours later). |
| Whether "Setting up for someone else" ever also prompts a router-style question for a *second* dependent in the same session | Adi/Soham | Lean: no — keep it one profile per pass, offer "add another" from Home afterward. |

---

## Review & Acceptance Checklist

### Content Quality
- [x] Focused on the first-five-minutes user experience, not implementation.
- [x] Explicitly scoped against `003` (no ABDM sync mechanics here).
- [x] All mandatory sections completed.

### Requirement Completeness
- [x] Requirements testable (tap-count cap, resumability, no self-report ABHA question).
- [x] Success criteria measurable (NFR-001 activation metric).
- [ ] Router copy finalized — **[NEEDS CLARIFICATION: Open Decision, owner Soham]**.

## Execution Status
- [x] User description parsed
- [x] Scenarios defined
- [x] Requirements generated
- [x] Entities identified
- [ ] Review checklist fully passed (blocked only on router copy sign-off)
