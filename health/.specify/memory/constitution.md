# TDC PHR — Project Constitution

<!--
Constitution for TDC Health — the TDC ecosystem's PHR patient app.
One project, built to production. There is no track split: every principle
below binds every spec, plan, and task under `specs/` until superseded.
-->

**Version:** 3.0.0 · **Ratified:** 2026-07-15 · **Last amended:** 2026-09-06
**Scope:** **All principles bind unconditionally.** I–IX are product
principles; X–XV are compliance principles. Neither set is deferred, staged,
or conditional.

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

## One Standard

**Superseded 2026-09-06 (v3.0.0): the Track A / Track B split is abolished.**

This section previously defined three binding classes — `[A]` (binds the demo
now), `[B→A]` (binds production, demo must not foreclose), `[B]` (production
only) — so that a demo-grade prototype could defer compliance work without
losing track of it. Owner decision, 6 Sep 2026: **there is one project, and it
ships to production.** Nothing is deferred to a later track, because there is
no later track.

Consequences that are not optional:

- **Every principle binds every feature.** A plan may not mark a principle
  N/A on the grounds that it is "for later". It may mark one N/A only when the
  feature genuinely has no such surface — and must say why.
- **`[A]` / `[B→A]` / `[B]` markers are removed** from this file and from
  every spec, checklist, and template. A marker surviving anywhere is a stale
  document, not a live exemption.
- **"Track B" is not a valid deferral target.** Work is sequenced by
  dependency, not by track. If something cannot be built yet because an
  external gate is closed (ABDM certification, UHI onboarding), the spec names
  **the gate and its owner** — not a track.

External requirements, with sources and verification dates, live in
`docs/compliance-baseline.md`. This file states rules; that file states facts.
When NHA publishes a new IG version or MeitY moves a deadline, the baseline doc
changes and these principles usually do not.

**On sequencing.** Every feature is a priority: none is cut to fund another,
and the old "cut from the bottom of P1" slip rule is repealed. That is not a
claim that everything is built at once. Capacity is one developer at roughly
four hours a day, so features ship in **dependency order**, one at a time, each
to completion. A spec that cannot state what it depends on is not ready to
plan.

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

### IV. Production-Grade, With Demo Data Switchable
**Rewritten 2026-09-06 (v3.0.0). This principle previously said the opposite.**
It read *"This is a seeded, demo-quality prototype… we do NOT build payments,
real ABDM data exchange, DPDP-grade hardening, or real auth hardening."* That
is repealed. Every feature is built to production quality; demo-grade is no
longer an acceptable answer to anything.

**The prototype survives as a switch, not as the product.** A demo path exists
so the app can be shown without live data, and it is governed by three rules:

1. **One switch, off by default.** `DEMO_MODE` is the single control. It is
   off in every build that leaves a developer machine, and a build with it on
   MUST fail its release check rather than ship quietly.
2. **Production correctness never depends on the demo path.** The real path is
   the one that is written first and tested. A demo path is a *substitute data
   source*, never a substitute *behaviour* — it may return fixture rows; it may
   not skip authorization, skip logging, weaken validation, or invent a state
   the real system cannot produce.
3. **Every demo path is inventoried** under Principle XIV.

Seed data (`seed_demo.py`) remains a first-class development and test fixture
and is the reason Principle V still stands. It is never shipped as content.

**What "production" does and does not mean here.** It means the code, the
authorization, the logging, the error handling and the copy are all
release-quality. It does **not** mean every external integration is live:
ABDM, UHI and payment rails each sit behind an external gate this project does
not control. A feature blocked on such a gate names **the gate and its owner**
and ships everything on our side of it — see *One Standard*. Claiming an
integration we do not have remains a Principle VI violation.

### V. Reset-in-One-Command
Demos and test runs mutate state. `seed_demo.py` MUST rebuild the full demo
world (3 profiles, seeded records, meds, emergency profiles, linked cards)
from one command. If a feature cannot be reset cleanly, it is not finished —
an unresettable feature is also an untestable one, which is why this principle
outlived the prototype it was written for.

### VI. Data Sovereignty Framing
Approved trust claims — "encrypted", "stored only in India", "never sold",
"you control who sees what", "every access is logged", "built on India's ABDM
framework (integration in development)" — MUST be literally true of the
prototype's behaviour or clearly scoped as roadmap. We do not ship a trust
screen we cannot defend.

### VII. Elderly-First Accessibility
Primary users include elderly patients (60s+). Every screen MUST be readable
at arm's length: large type, large tap targets, high contrast. The warm/calm
palette (site blue `#4B83F2` family) and a semantic Flutter theme (`ThemeData`)
are the design baseline. Concrete token values live in `docs/design-tokens.md`.

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

*Rationale and sources: `docs/compliance-baseline.md`. These bind exactly as
hard as I–IX; there is no class to check first.*

### X. Authorization Is Derived, Never Accepted
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

Broken object-level authorization is OWASP API #1 and the single most common
WASA blocker. It costs nothing to do correctly the first time and means
auditing every endpoint ever written if retrofitted. It also matters in front
of people: any demo shows one family's data, and a scoping bug on stage shows
someone else's.

### XI. Every PHI Access Leaves a Log
Every read or write of health data through Core API or Fastlane MUST emit one
`access_logs` row: **actor, subject profile, action, object type + id,
timestamp, purpose, source service.** Writes happen in the same transaction as
the access; a failed log write fails the request.

This binds for a reason that is not about audits: **"every access is logged"
is already on the approved-copy list**, and Principle VI requires approved
trust claims to be literally true of the system's behaviour. Until
`access_logs` exists, the Trust screen makes a claim we cannot defend. The
compliance requirement and the honesty requirement land on the same table.

`scan_events` is the Fastlane-side precedent and MUST NOT be replaced by it —
`scan_events` records card mechanics (counter, replay state); `access_logs`
records who saw what. Keep both.

### XII. FHIR Is Pinned
Any code, fixture, mapping, or spec that emits or parses FHIR MUST target
**FHIR R4.0.1** against the **NRCeS FHIR Implementation Guide for ABDM
v6.5.0**. FHIR R5 is banned. The v7.0.0 IG is a **draft** (as of 2026-07-15)
and MUST NOT be built against.

No FHIR is emitted yet, so this is currently inert — but it binds the moment a
single FHIR resource appears, including in a mock, a fixture, or a slide. The
pin is dated: re-verify the released IG version at the start of any FHIR work
rather than trusting this line.

### XIII. PHI Does Not Cross a Boundary in Plaintext
**On the ABDM network:** health data exchanged with ABDM MUST be encrypted with
Fidelius — ephemeral Curve25519 ECDH keypair and fresh 32-byte nonce per
transaction, HKDF-derived AES-256-GCM session key, no shared-secret reuse —
implemented from the official Fidelius CLI reference parameters, not from
default X25519 settings in a general-purpose library.

**Structural rule, binding now even before ABDM exchange exists:** keep
payload *assembly* separable from payload *transport*. No module may assume a
bundle is readable at the point it is handed off. A persisted or
contract-level shape that only makes sense as plaintext is what this clause
forbids — the point is that it costs nothing today and is a rewrite later.

**Also binding, on a surface that is live now:** the Fastlane responder page serves real PHI
over a public URL. It MUST be HTTPS-only, MUST NOT include PHI in query
strings, log lines, or referrer-leaking URLs, and MUST NOT be indexable. That
part is not deferred — it is live on demo day.

### XIV. Every Demo Path Is Inventoried, and the Inventory Shrinks
**Reframed 2026-09-06 (v3.0.0).** This principle used to say prototype
shortcuts were a remediation list for a future track to work through. There is
no future track, so the list is **a burn-down with an owner and a blocking
gate**, not a bequest.

Every `# DEMO-MODE` tag MUST carry, on the same line or the line below: what
the demo path substitutes, what the real path is, and — if it is not built yet
— **which external gate blocks it and who owns that gate**:

```python
# DEMO-MODE: OTP fixed to 000000 — fixture only, real path is the default.
# Real: SMS provider + attempt throttling + lockout. Built. No gate.
```

```python
# DEMO-MODE: seeded provider directory.
# Real: UHI/HFR provider discovery. GATE: UHI onboarding — owner Adi.
```

Rules:

- A tag without a real-path note is an **incomplete tag** and fails review.
- A tag whose real path is unbuilt and names **no gate** is a **defect** — it
  means work was deferred without a reason, which is precisely what abolishing
  the track split was meant to stop.
- **The demo path may never be the only path.** If removing `DEMO_MODE` breaks
  a feature, that feature is not finished (Principle IV, rule 2).
- The inventory is reviewed at every release check, and a tag that has sat
  gate-free across two reviews is escalated rather than renewed.

### XV. Consent Artifacts, Retention, and Erasure
**The obligation:** ABDM consent is a cryptographically signed artifact carrying
purpose, HI types, date range, care contexts, and an expiry. `dataEraseAt` is a
**legal obligation on us** — as a PHR, TDC is the HIU, so the duty to purge or
anonymise fetched data on consent expiry lands here, not upstream. Consent
transactions MUST be logged auditably.

**Binding now, before any ABDM exchange exists:** records MUST carry
provenance (`source`, already in the model) and MUST remain individually
deletable — nothing may make a record's origin unrecoverable or its deletion
require rewriting an aggregate. A denormalized snapshot (`emergency_payload`)
is permitted precisely because it is regenerable from its sources.

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

- **One codebase:** Flutter targeting Android + web. No iOS work.
- **Two-service split, not one:** the emergency responder MUST live in a
  service physically separate from Core API, with its own uptime budget —
  this is what makes Principle I's "survives Core being down" guarantee real
  rather than aspirational (see `001` research.md R1 for the rejected
  single-service alternative).
- **Security posture: production.** Authorization scoping (X), access logging
  (XI), and a live demo-path inventory (XIV) are the floor, not the ceiling.
  Card replay protection (reject `ctr ≤ last_seen`, verify CMAC) is required.
  **Auth hardening is no longer deferred** — rate limiting, lockout, session
  and token handling, and secret management are in scope for the features that
  own them, and the phrase "full auth hardening is Track B" is repealed.
- **A WASA audit is the target, not a someday.** `docs/compliance-baseline.md`
  records that a CERT-In empanelled audit is a **precondition for ABDM M1** and
  that one critical finding stalls certification. Every feature is written to
  survive that audit, which is the practical meaning of "production" here.

---

## Development Workflow

1. Every feature begins as a `spec.md` under `specs/NNN-slug/` and MUST pass
   the Review & Acceptance Checklist before planning.
2. `plan.md` MUST include a Constitution Check gate; violations are documented
   in Complexity Tracking with justification or the plan is revised.
3. `tasks.md` is generated from the plan and ordered by **dependency**. The old
   P0/P1 ladder and its "cut from the bottom of P1" slip rule are **repealed**
   (v3.0.0): no feature is cut to fund another. Principle I still governs what
   is never destabilised — the emergency flow is not reordered behind anything
   — but it is now a stability rule, not a triage rule. A feature that cannot
   state its dependencies is not ready to plan.
4. Every demo path MUST be tagged `# DEMO-MODE` in code, carry its real-path
   note and (if unbuilt) its blocking gate and gate owner, and be listed in the
   owning spec (Principle XIV). `DEMO_MODE` is off in every build that leaves a
   developer machine.
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
7. **Every feature owns a complete spec-kit set** — `spec.md`, `plan.md`,
   `tasks.md`, and its checklists. "Every feature gets its own space" (owner,
   6 Sep 2026) means none is a subsection of another's plan, and none ships on
   a neighbouring feature's tasks.

---

## Governance

This constitution supersedes any patient-app scope in older handover or
Hyperledger-era documents. Amendments require a version bump (semver:
MAJOR for principle removal/redefinition, MINOR for new principles, PATCH for
clarifications) and a note in the amendment history. **There is no forking
track to re-ratify** — this file is the only constitution the project has.

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
- 2.0.1 (2026-08-15) — **PATCH: corrected a stale duplicated fact.** Principle
  VII and Constraints & Standards both still said Expo React Native/NativeWind;
  `CLAUDE.md` locked Flutter as the stack on 2026-08-15 (owner decision, see
  its changelog) and this file hadn't followed. No principle changed — this is
  exactly the kind of duplication-drift the 1.2.0 entry above already warned
  about.
- 3.0.0 (2026-09-06) — **MAJOR: the track split is abolished.** Owner decision:
  one project, built to production, every feature a priority with its own
  space. Removed the *Binding Classes* section and the `[A]` / `[B→A]` / `[B]`
  markers from Principles X–XV — **all fifteen principles now bind
  unconditionally**, and "Track B" is no longer a valid deferral target; a
  feature blocked by an external gate names the gate and its owner instead.
  **Principle IV was inverted**, from *"Demo-Grade, Not Production-Grade"* to
  *"Production-Grade, With Demo Data Switchable"*: the prototype survives as a
  `DEMO_MODE` switch that is off by default, may substitute data but never
  behaviour, and may never be the only path — if removing it breaks a feature,
  the feature is unfinished. **Principle XIV was reframed** from a remediation
  list bequeathed to a later track into a burn-down: a demo tag must name its
  real path, and an unbuilt real path must name its blocking gate and that
  gate's owner, or it is a defect. Principle V survives on new grounds — an
  unresettable feature is an untestable one. Security posture moved from
  prototype to production, repealing *"full auth hardening is explicitly Track
  B"*. Workflow rule 3 repealed the P0/P1 ladder and the cut-from-the-bottom
  slip rule in favour of dependency ordering; Principle I is now a stability
  rule rather than a triage rule. Added workflow rule 7 (every feature owns a
  complete spec-kit set). **Capacity is unchanged** — one developer, ~4h/day —
  so this amendment raises the quality bar without raising throughput:
  features ship in dependency order, one at a time, each to completion.
  Supersedes `002`'s 18 Aug 2026 governance clause (*"tracks stay separate…
  the constitution keeps its per-principle binding classes"*), which is void.
