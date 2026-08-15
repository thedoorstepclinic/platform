# TDC PHR — Project Constitution

<!--
Constitution for TDC Health — the TDC ecosystem's PHR patient app.
This governs Track A (investor-demo prototype). Track B (production MMP) will
ratify its own amended constitution post-funding. Principles here are binding
on every spec, plan, and task under `specs/` until superseded.
-->

**Version:** 2.0.0 · **Ratified:** 2026-07-15 · **Last amended:** 2026-08-10
**Scope:** Principles I–IX govern Track A (investor-demo prototype).
Principles X–XV are **compliance principles** and carry an explicit binding
class each — some bind Track A now, some bind Track B while constraining what
Track A is allowed to foreclose. See *Binding Classes* below.

---

## Purpose

The TDC PHR app makes one promise felt: **"A family's health, handled — even
when the worst happens."** Every principle below exists to protect that promise
and the trust it depends on. When a decision is ambiguous, choose the option
that keeps a real family safe and a real investor convinced.

## Relationship to CLAUDE.md

Two documents govern this repo, and they operate at different altitudes —
neither restates the other's content:

- **This constitution** holds cross-cutting **principles** — the *why*,
  binding on every spec regardless of which feature it's in.
- **`CLAUDE.md`** (repo root) holds concrete **locked decisions** — the
  *what*: names, package IDs, domains, stack choices, banned/approved
  phrases, data model. It is the source of truth for those specifics.

If a locked decision in `CLAUDE.md` isn't yet reflected here, `CLAUDE.md`
wins and this file is due an amendment — not the other way around. If a
principle here would require a `CLAUDE.md` decision to change, the
principle wins and `CLAUDE.md` is due a correction. They should never need
to be checked against each other for a feature spec to proceed; if they
ever visibly disagree, that disagreement is itself a bug, fixed the same
day.

---

## Binding Classes

Principles I–IX bind Track A unconditionally and carry no class marker.

Principles X–XV exist because TDC will need ABDM M1–M3 certification, a WASA
security audit by a CERT-In empanelled auditor, and DPDP compliance — none of
which Track A performs, all of which Track A can quietly make expensive. Each
carries one of three markers:

- **`[A]` — binds Track A now.** Cheap to satisfy at generation time,
  ruinous to retrofit. A plan or implementation that violates it does not pass
  the Constitution Check.
- **`[B→A]` — binds Track B; Track A MUST NOT foreclose it.** Track A does not
  build the thing. Track A *does* have to avoid decisions that make building it
  later a rewrite. The non-foreclosure clause is the testable part and is the
  only part a Track A plan is graded on.
- **`[B]` — binds Track B only.** Recorded here so it is not rediscovered
  late. Track A ignores it entirely.

**These principles do not repeal Principle IV.** Where a compliance rule would
require production-grade work, it binds as `[B→A]` or `[B]` — never `[A]`. If
a future amendment promotes a rule to `[A]`, Principle IV's scope statement is
amended in the same commit or the promotion is invalid.

External requirements, with sources and verification dates, live in
`docs/compliance-baseline.md`. This file states rules; that file states facts.
When NHA publishes a new IG version or MeitY moves a deadline, the baseline doc
changes and these principles usually do not.

---

## Core Principles

### I. Emergency Flow Is Sacred (NON-NEGOTIABLE)
The Fastlane responder page and the family-blast are the crown jewels. They
MUST work on demo day. When scope is cut, it is cut from the bottom of P1
upward — **never** from the emergency flow. Fastlane MUST survive Core API
being down: it reads a denormalized `emergency_payload` snapshot, never live
joins. The responder page MUST render readable with zero JavaScript and load
in under 2 seconds on 4G.

### II. Consent Is a Standing Grant, Not a Card Tap (NON-NEGOTIABLE)
Consent is granted once, in-app, at setup — logged and revocable. The NFC card
is a **capability token exercising a standing grant**. No screen, no copy, no
log line may describe a card-tap as the moment of consent. Every caregiver
relationship is recorded in `caregiver_grants` with a timestamp; that record
is the consent moment.

### III. Copy Discipline
UI copy is constrained to the approved list and MUST avoid the banned list
(see the feature spec §Copy Guardrails). Specifically banned in any
user-facing surface: "blockchain", "Hyperledger", cipher names (e.g.
"AES-256"), "ABDM certified", "military-grade", and any phrasing implying a
card-tap equals consent. Clinical-sounding claims we cannot back are
prohibited.

### IV. Demo-Grade, Not Production-Grade — Stated Honestly
This is a seeded, demo-quality prototype. We do NOT build payments, OCR/AI
extraction, real ABDM data exchange, DPDP-grade hardening, iOS, offline mode,
or real auth hardening. Mocks (OTP `000000`, manual ABHA field, fake
webhook) are acceptable and MUST be labelled as demo-mode in the codebase so
Track B does not inherit them silently.

### V. Reset-in-One-Command
Demos mutate state. `seed_demo.py` MUST rebuild the full demo world (3
profiles, seeded records, meds, emergency profiles, linked cards) from one
command. If a feature cannot be reset cleanly, it is not demo-ready.

### VI. Data Sovereignty Framing
Approved trust claims — "encrypted", "stored only in India", "never sold",
"you control who sees what", "every access is logged", "built on India's ABDM
framework (integration in development)" — MUST be literally true of the
prototype's behaviour or clearly scoped as roadmap. We do not ship a trust
screen we cannot defend.

### VII. Elderly-First Accessibility
Primary users include elderly patients (60s+). Every screen MUST be readable
at arm's length: large type, large tap targets, high contrast. The warm/calm
palette (site blue `#4B83F2` family) and shadcn-style tokens via NativeWind are
the design baseline. Concrete token values live in `docs/design-tokens.md`.

### VIII. Home Stays Quiet (NON-NEGOTIABLE once any new surface is proposed)
No content feed, no health tips, no "trending," nothing that refreshes to
harvest attention. Alerts resolve and disappear — they are an alerts strip,
never an inbox. This is a business-model commitment, not a design
preference: TDC monetizes coordination and trust, not attention, and the
navigation structure exists to enforce that split (see
`008-navigation-app-shell` — hub-and-spoke, no persistent tab bar, so there
is no idle nav real estate to fill). Any future surface — a service, a
promotion, a "did you know" — attaches to a person's profile context or to
an existing alert; it never claims a persistent slot competing with Home's
quiet. Reference apps: steal Tata 1mg's/Eka Care's record-viewing patterns;
never their commerce-feed home.

### IX. One Snapshot, Never a Parallel Template
Any screen that previews what another party or another format will show
MUST render from the exact same structured data source as the real thing —
never a second, hand-maintained template. Established instances: the
emergency-profile "what responders will see" preview renders from the same
`emergency_payload` snapshot Fastlane serves (`007`); the Health Summary
screen renders the same field mapping as its PDF (`009`, per `004`). Drift
between "what we show" and "what actually goes out" is a bug class, not a
copy-editing task — any new preview-style screen inherits this rule by
default.

---

## Compliance Principles

*Rationale and sources: `docs/compliance-baseline.md`. Read the binding class
before applying any of these to a Track A plan.*

### X. Authorization Is Derived, Never Accepted `[A]`
**No endpoint may trust a client-supplied identifier to decide what the caller
may see.** Every queryset MUST be filtered by a permitted-scope set derived
server-side from the authenticated principal — for TDC Health that set is the
caller's own profiles plus the profiles reachable through an unrevoked row in
`caregiver_grants`.

Testable form:

- A DRF viewset touching `profiles`, `records`, `medications`, `dose_events`,
  `emergency_profiles`, `appointments`, `cards`, or `scan_events` MUST override
  `get_queryset()` with a scope filter. A viewset whose `get_queryset()`
  returns an unfiltered `Model.objects.all()` is a **build failure**, not a
  review comment.
- `get_object()` MUST resolve out of the scoped queryset. Never
  `Model.objects.get(pk=...)` followed by a permission check.
- A revoked grant MUST stop access on the next request — no cached scope.

This is `[A]` despite Principle IV because broken object-level authorization is
OWASP API #1, is the single most common WASA blocker, and costs nothing to do
correctly the first time. Retrofitting it means auditing every endpoint ever
written. It also matters *inside the demo*: the golden path shows one family's
data, and a scoping bug on stage shows someone else's.

### XI. Every PHI Access Leaves a Log `[A]`
Every read or write of health data through Core API or Fastlane MUST emit one
`access_logs` row: **actor, subject profile, action, object type + id,
timestamp, purpose, source service.** Writes happen in the same transaction as
the access; a failed log write fails the request.

This is `[A]` for a reason that is not about audits: **"every access is
logged" is already on the approved-copy list**, and Principle VI requires
approved trust claims to be literally true of the prototype's behaviour. Until
`access_logs` exists, the Trust screen makes a claim we cannot defend. The
compliance requirement and the honesty requirement land on the same table.

`scan_events` is the Fastlane-side precedent and MUST NOT be replaced by it —
`scan_events` records card mechanics (counter, replay state); `access_logs`
records who saw what. Keep both.

### XII. FHIR Is Pinned `[A]` for any FHIR-touching code
Any code, fixture, mapping, or spec that emits or parses FHIR MUST target
**FHIR R4.0.1** against the **NRCeS FHIR Implementation Guide for ABDM
v6.5.0**. FHIR R5 is banned. The v7.0.0 IG is a **draft** (as of 2026-07-15)
and MUST NOT be built against.

Track A emits no FHIR, so this principle is usually inert — but it binds the
moment a single FHIR resource appears, including in a mock, a fixture, or a
slide. The pin is dated: re-verify the released IG version at the start of any
FHIR work rather than trusting this line.

### XIII. PHI Does Not Cross a Boundary in Plaintext `[B→A]`
**Track B:** health data exchanged with the ABDM network MUST be encrypted with
Fidelius — ephemeral Curve25519 ECDH keypair and fresh 32-byte nonce per
transaction, HKDF-derived AES-256-GCM session key, no shared-secret reuse —
implemented from the official Fidelius CLI reference parameters, not from
default X25519 settings in a general-purpose library.

**Track A must not foreclose it:** keep payload *assembly* separable from
payload *transport*. No module may assume a bundle is readable at the point it
is handed off. A persisted or contract-level shape that only makes sense as
plaintext is the foreclosure this clause forbids.

**Track A also binds, narrowly:** the Fastlane responder page serves real PHI
over a public URL. It MUST be HTTPS-only, MUST NOT include PHI in query
strings, log lines, or referrer-leaking URLs, and MUST NOT be indexable. That
part is not deferred — it is live on demo day.

### XIV. Demo Shortcuts Are the Audit Remediation List `[A]`
Principle IV requires prototype shortcuts to be tagged `# DEMO-MODE`. This
principle states what that tag is *for*: it is the pre-written remediation list
a WASA auditor will otherwise generate at our expense.

Therefore every `# DEMO-MODE` tag MUST carry, on the same line or the line
below, what makes it unsafe and what replaces it:

```python
# DEMO-MODE: OTP fixed to 000000 — no rate limit, no delivery.
# Track B: real OTP provider + attempt throttling + lockout.
```

A tag without a replacement note is an incomplete tag. Mock OTP, the manual
ABHA field, the fake HMS webhook, and any permissive CORS or debug setting are
all in scope. The inventory of these tags is a Track B deliverable, not a
follow-up task.

### XV. Consent Artifacts, Retention, and Erasure `[B→A]`
**Track B:** ABDM consent is a cryptographically signed artifact carrying
purpose, HI types, date range, care contexts, and an expiry. `dataEraseAt` is a
**legal obligation on us** — as a PHR, TDC is the HIU, so the duty to purge or
anonymise fetched data on consent expiry lands here, not upstream. Consent
transactions MUST be logged auditably.

**Track A must not foreclose it:** records MUST carry provenance
(`source`, already in the model) and MUST remain individually deletable —
nothing may make a record's origin unrecoverable or its deletion require
rewriting an aggregate. A denormalized snapshot (`emergency_payload`) is
permitted precisely because it is regenerable from its sources.

**Do not conflate two different consent systems.** ABDM consent is
per-patient and lives on the ABDM network. `caregiver_grants` is TDC's own
family layer and is **not** ABDM delegation. Principle II governs the second;
this principle governs the first. Any spec that blurs them is rejected at the
gate.

---

## Constraints & Standards

Principle-level constraints only — for concrete values (exact domains,
package IDs, hosting provider, app names across the ecosystem) see
`CLAUDE.md`, which is authoritative and updates independently of this file.
For external compliance facts (milestone definitions, WASA scope, Fidelius
construction, FHIR IG version pins, statutory dates) see
`docs/compliance-baseline.md`, which carries its own verification date.

- **One codebase:** Expo React Native targeting Android + web. No iOS work.
- **Two-service split, not one:** the emergency responder MUST live in a
  service physically separate from Core API, with its own uptime budget —
  this is what makes Principle I's "survives Core being down" guarantee real
  rather than aspirational (see `001` research.md R1 for the rejected
  single-service alternative).
- **Security posture:** Prototype-grade, with three exceptions that are not
  negotiable even on Track A because they are cheap now and a rewrite later:
  authorization scoping (X), access logging (XI), and documented `# DEMO-MODE`
  tags (XIV). Replay protection on cards (reject `ctr ≤ last_seen`, verify
  CMAC) is required because it is part of the story; full auth hardening is
  explicitly Track B.

---

## Development Workflow

1. Every feature begins as a `spec.md` under `specs/NNN-slug/` and MUST pass
   the Review & Acceptance Checklist before planning.
2. `plan.md` MUST include a Constitution Check gate; violations are documented
   in Complexity Tracking with justification or the plan is revised.
3. `tasks.md` is generated from the plan and ordered by the build order in the
   spec. The build-order rule (cut from bottom of P1, never the emergency flow)
   governs task prioritisation.
4. Prototype shortcuts MUST be tagged `# DEMO-MODE` in code and listed in the
   spec so Track B can find and replace them.
5. Any change to a shared data model (`records`, `medications`, `dose_events`,
   `emergency_profiles`, `scan_events`, or any table `seed_demo.py` populates)
   MUST update `seed_demo.py` and `004-seed-data-and-summary` the same day —
   a stale seed script is treated as a broken build, not a follow-up task.
6. Every feature spec that touches PHI, authentication, grants, Fastlane, or
   any ABDM surface MUST carry a **compliance checklist** at
   `specs/NNN-slug/checklists/security.md` (and `abdm.md` where an ABDM
   surface is involved), generated from
   `.specify/templates/checklist-security.md` / `checklist-abdm.md`. The
   checklist is reviewed against `plan.md` **before** implementation —
   compliance defects are cheap at plan time and expensive at audit time.
   Features touching none of the above may skip it and say so in the plan.

---

## Governance

This constitution supersedes any patient-app scope in older handover or
Hyperledger-era documents. Amendments require a version bump (semver:
MAJOR for principle removal/redefinition, MINOR for new principles, PATCH for
clarifications) and a note in the amendment history. Track B will fork and
re-ratify.

**Amendment history**
- 1.0.0 (2026-07-15) — Initial ratification for Track A prototype.
- 1.1.0 (2026-07-16) — Added Principle VIII (Home Stays Quiet), promoted from
  repeated feature-level statements in `001` §4.1 and `008` once the
  navigation-shell discussion made clear it's a business-model commitment,
  not a per-screen style choice.
- 1.2.0 (2026-07-16) — Added Principle IX (One Snapshot, Never a Parallel
  Template), generalizing a rule that had already been independently applied
  twice (`007`'s responder preview, `009`'s summary screen) without being
  named once at this level. Added the "Relationship to CLAUDE.md" section to
  resolve a real problem: both documents implied they were the final word
  and neither referenced the other. Added the same-day seed-ripple rule to
  Development Workflow (already de facto practice across `004`/`006`/`007`,
  never actually written down here). Trimmed Constraints & Standards to stop
  duplicating facts that live in `CLAUDE.md` — that duplication is what let
  this file go stale for a full day of otherwise-tracked spec work
  (`002`–`009`) without anyone noticing until asked to check.
- 2.0.0 (2026-08-10) — **MAJOR: the constitution's scope changed.** It was
  "Track A only"; it now also carries forward-binding compliance principles
  (X–XV) covering ABDM M1–M3, WASA, and DPDP. That redefinition is what makes
  this MAJOR rather than MINOR — no principle was removed, but the question
  "does this file apply to my work?" now has a different answer and is
  answered per-principle by the new **Binding Classes** section. Rules that
  would contradict Principle IV are bound `[B→A]` or `[B]` instead of being
  bolted on as Track A mandates; only three clauses bind Track A now
  (X authorization scoping, XI access logging, XIV documented DEMO-MODE
  tags), each chosen because it is cheap today and a rewrite later.
  Principle XI adds one table, `access_logs` — the first Track A build cost
  this file has ever imposed, justified because Principle VI already promised
  it in approved copy. External facts moved out to
  `docs/compliance-baseline.md` so version pins and statutory dates can rot
  in one place instead of inside principle text. Added Development Workflow
  §6 (per-feature compliance checklists).
