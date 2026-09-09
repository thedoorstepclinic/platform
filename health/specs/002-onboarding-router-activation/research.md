# Research & Decision Log: Onboarding, Emergency Card & ABHA Activation

Companion to [`spec.md`](./spec.md). This file records **why** the flow is
shaped the way it is — the reasoning, the rejected alternatives, and the
external facts the design leans on. `spec.md` states the rules; this states
the thinking behind them, so a future contributor does not re-litigate a
settled question or, worse, quietly reverse it.

**Convention:** each entry is `R{n}` — decision, reasoning, alternatives
rejected, and what would make us revisit. Entries are append-only; a
superseded entry is struck through, never deleted.

Last updated: 2026-08-18.

---

## R1 — The router question is removed, but its signal is kept

**Decision.** Delete the standalone "Who will you use TDC Health for?" screen.
Capture the same persona signal at the emergency-card step: *"Who is this card
for? Me / My parent / Someone else."*

**Reasoning.** The router asked users to classify themselves in the abstract,
before they had done anything — the kind of question people answer badly
because it has no consequence they can see yet. The card step asks the same
question at the moment it *matters*, attached to a real artifact they are
building. Same information, one fewer screen, more reliable answer.

It also survives the "working individual, short of time" constraint better:
every screen in a signup flow is a chance to lose someone, and an abstract
self-classification screen buys nothing the user can feel.

**Rejected:** keeping the router but merging it into the OTP screen (still an
abstract question, now crowding a screen with one job); inferring persona
silently from later behavior (too slow — Home needs the signal immediately).

**Revisit if:** the card step's three options prove too coarse to weight Home,
or if analytics show users pick "Me" reflexively and then add parents anyway.

---

## R2 — Card before ABHA, not after

**Decision.** The emergency card is created in the mandatory spine (step 3 of
3). ABHA is offered afterward, outside the progress bar.

**Reasoning.** Two independent arguments landed on the same order.

1. **Dependency risk.** The card depends on nothing outside our own system.
   ABHA depends on ABDM, Aadhaar OTP delivery, and a gateway we do not
   control. Putting the failure-prone step *after* the value-delivering step
   means a user whose ABHA linking fails still finishes onboarding with the
   thing they came for.
2. **Activation honesty.** NFR-001 defines activation as *a live emergency
   card*, not an ABHA link. If ABHA were in the spine, the metric and the
   flow would disagree about what success means.

**Rejected:** Aadhaar-first with the card prefilled from eKYC. It needs fewer
keystrokes and removes the reconciliation problem in R7 — genuinely
attractive — but gates the emotional payoff behind an external system.

**Settled.** This was briefly reopened by liveness, which only has meaning
at/after eKYC. Liveness was then cut (R9), so card-first stands unchallenged
and the ordering is final.

---

## R3 — Blood group, allergies, and conditions are always user-entered

**Decision.** No external system populates, overwrites, or infers the clinical
fields on the emergency card. They are the user's own input and choice, and
the product never presents them as verified.

**Reasoning.** This started as a technical observation and became a policy.

*The technical fact:* Aadhaar eKYC returns name, DOB, gender, address, and
photo. It returns no blood group. Neither does ABHA. There is no Indian
identity system that will hand us the single most important field on the
responder page — the one rendered largest, the one a paramedic reads first.
So the card step must ask for it regardless of what else we integrate.

*The policy that follows:* since the user is the only possible source, the
user is also the only responsible party, and the product must say so plainly.
Claiming or implying verification of a field we cannot verify is exactly the
kind of overclaim Principle VI exists to prevent — and here it is not merely
a trust problem, it is a clinical one.

**Consequence:** FR-013a and FR-017 both encode this from opposite directions
— one says the user is the source, the other forbids eKYC from overwriting.

**Revisit if:** a verified clinical source ever exists (a linked lab result
carrying a blood group, for instance). Even then, *display provenance* rather
than silently replacing user input.

---

## R4 — ABHA is strictly per profile

**Decision.** One ABHA belongs to exactly one profile. Onboarding links only
the account holder's own ABHA. A dependent's ABHA is linked later, from that
dependent's profile, with their own Aadhaar and their own mobile OTP. Enforced
as a database constraint, not a convention (FR-014a).

**Reasoning.** The caregiver's phone is the account's phone. Without this
rule, a caregiver who builds their first card for a parent would land on the
ABHA offer and link *their own* ABHA onto the *parent's* profile — silently
merging two people's health identities.

That is not a UX wrinkle. ABDM consent is strictly per-patient
(`CLAUDE.md`, §Consent model; `docs/compliance-baseline.md` §6), and TDC's
family layer is our own `caregiver_grants` construct, explicitly *not* ABDM
delegation. Conflating them is the single most likely compliance error in this
product — flagged as such in the baseline before this spec existed, and the
onboarding flow is exactly where it would have happened.

**Rejected:** a profile picker at the ABHA step (adds a screen and still
invites the wrong answer); attempting the parent's ABHA in-flow with the
parent's Aadhaar (the OTP goes to a phone the caregiver does not hold — see
the edge case in `spec.md`).

---

## R5 — Family linking is bidirectional

**Decision.** A family connection may originate from either side: a caregiver
inviting outward, or a family member requesting inward. Same primitives
(short-lived link, verification code, invite), symmetric outcome, approval
always required from the receiving side.

**Reasoning.** The original draft assumed the caregiver always initiates. That
matches the demo persona but not reality — an adult child who installs the app
after a parent already has it should not have to ask the parent to start over
from the parent's phone. Making direction irrelevant costs one `direction`
column and removes a whole class of "who has to go first" support tickets.

**Constraint kept:** invite-out and request-in both create or connect an
**independent account**, never a `caregiver_grants` relationship and never a
shared ABHA (R4). Assisted add remains the only path that produces a managed
profile, and it remains the recorded consent moment (`001` FR-003).

---

## R6 — Aadhaar numbers are never stored

**Decision.** Transit-only to the ABDM gateway. No column, no cache, no local
write. Excluded from access logs, error logs, APM traces, and crash reports by
an actual scrubber, not a convention. Masked on entry blur.

**Reasoning.** Two reinforcing pressures. Legally, Aadhaar storage carries
obligations we have no reason to take on when the number is only ever a
lookup key. Practically, `docs/compliance-baseline.md` §3 names broken
object-level authorization and data-exposure findings as the highest-frequency
WASA failures, and a single critical finding stalls ABDM certification
entirely. A plaintext Aadhaar in an APM trace is the cheapest possible way to
fail an audit we will otherwise pass.

The current `data-model.md` has no Aadhaar field. That absence is the design,
and it should be defended deliberately rather than preserved by accident.

---

## R7 — Verified data reconciles, never overwrites

**Decision.** When eKYC returns a name or DOB that differs from what the user
typed on the card, offer a one-tap "update card to match". Never apply
silently, never block on a form.

**Reasoning.** Card-first ordering (R2) means users type identity data before
the system has authoritative data. Silent overwrite is the tempting fix and
the wrong one: it changes a document the user believes they authored, and for
an emergency card the user may have typed a *preferred* name deliberately.
Blocking on a reconciliation form is the other wrong fix — it stops the flow
for a cosmetic difference.

Clinical fields are exempt entirely: eKYC has no opinion on blood group and
must never touch it (R3).

---

## R8 — App lock delegates to the platform; no custom MPIN

**Decision.** Use device PIN, phone password, or biometric via the OS
credential prompt. TDC builds, stores, hashes, and recovers nothing.

**Reasoning.** Owner call, and the better engineering answer. A custom MPIN
means a credential store, a hashing scheme, a reset flow, a lockout policy,
and a new surface in the WASA assessment — all to reproduce something the
platform already does well and the user already trusts. UPI apps established
this habit across exactly our user base, so the familiar pattern is also the
cheap one.

`docs/backlog.md` §3 notes the threat model here is *household*, not
remote — the person you least want reading a parent's records is often in the
same room. Platform auth handles that case as well as a custom PIN would.

**Placement:** offered at the end of the flow, never inside the spine. A lock
prompt mid-signup interrupts the one flow we are actively trying to shorten.

---

## R9 — Liveness is cut; identity assurance is Aadhaar eKYC alone

**Decision.** No liveness, no presence verification, no biometric capture of
any kind. The card photo is **chosen by the user**, and eKYC's photo is
deliberately unused.

**History.** Briefly retained mid-session, then cut in the same session. Worth
recording the arc, because the argument for keeping it was not stupid: eKYC
returns the holder's photo, so a liveness capture *could* be face-matched
against it, which is a real identity check and a well-established Indian KYC
pattern. That is the only version worth building — standalone liveness with
nothing to compare against proves nothing.

**Why it was cut anyway.**
- **DPDP weight.** Liveness output is biometric data, a heavier category than
  a stored photo, carrying its own notice, purpose limitation, and retention
  rule. That is real compliance surface for a marginal gain.
- **Vendor dependency.** It needs a third-party SDK. A hand-rolled check is
  defeated by a photo of a photo and is worse than none, because it invites
  reliance it cannot support.
- **It does not address the actual risk.** For an emergency card the danger is
  a **wrong blood group**, not an impostor at signup (R3). Liveness does
  nothing about the failure mode that can actually kill someone.
- **Aadhaar OTP already clears the bar.** Possession of the Aadhaar-linked
  mobile plus a successful eKYC is stronger assurance than a selfie, and it is
  already in the flow at zero extra cost.

**Second-order benefit.** Cutting it also settled the S3/S4 ordering: liveness
was the only thing pulling Aadhaar ahead of the card step. With it gone, R2's
card-first ordering stands unchallenged.

**User-chosen photo.** Letting the user pick their own photo is not a
consolation prize. An emergency card is a document its owner should recognise
as theirs, and eKYC photos are frequently years old and unflattering. The card
is not an identity document — it is a card a stranger reads in an emergency.

---

## R16 — Tracks stay separate — **REVERSED 2026-09-06**

> **This decision was reversed 19 days later.** Owner decision, 6 Sep 2026:
> **one project, built to production, no tracks.** Constitution **v3.0.0**
> deletes the binding classes entirely. The reasoning below is left as written
> — it is a decision log, and it argued its case honestly — but note what
> actually happened to its central claim: it held that merging the classes
> would let a production requirement *"quietly become a Track A obligation by
> proximity"* at the moment scope pressure is highest. The reversal does not
> dispute that risk; it removes the thing the risk was protecting. There is no
> lighter track left to protect, because every feature is now built to
> production.
>
> The one prediction that held: *"a Track B constitution still needs
> ratifying."* It never was. The split was abolished before it could be, which
> is arguably the cheaper outcome — one governing document instead of two that
> could disagree.


**Decision.** Track A completes, then Track B's foundation, and so on. The
constitution keeps its per-principle binding classes (`[A]` / `[B→A]` / `[B]`)
— they are **not** merged. The only permitted crossover is one or two
individual FRs slipping from B into A, named explicitly when it happens.

**Reasoning.** Owner direction, reversing a briefly-considered merge. The
binding classes exist precisely so a Track B requirement cannot quietly become
a Track A obligation by proximity; merging them would delete that protection
at the exact moment scope pressure is highest. Sequencing tracks also keeps
the "don't gold-plate" discipline enforceable — Track A has a definition of
done that a merged document would blur.

**Practical rule:** when an FR does slip B→A, say so in the spec and in the
`CLAUDE.md` changelog. A track boundary moves one requirement at a time, with
a reason, or not at all.

~~**Standing constraint:** a Track B constitution still needs ratifying before
compliance testing. Separate tracks does not mean an unratified one.~~
**Resolved differently (6 Sep 2026):** no second constitution was ratified and
none will be. v3.0.0 is the only one, and it binds everything.

---

## R10 — No rate limit on valid emergency-card reads

**Decision.** Reads of a valid digital card token are never throttled. Abuse
protection caps **invalid** token attempts per source instead.

**Reasoning.** The digital card is a bearer token, and the instinct is to rate
limit it. But the failure mode of rate limiting an emergency page is that a
legitimate responder is refused at the worst possible moment — the page fails
at its only job. Meanwhile the actual threat a rate limit would address is
enumeration, and enumeration is defeated by token entropy, not throttling.

Capping invalid attempts gets the enumeration protection without ever touching
a real scan. Rotation and revocation remain the primary defense, which is why
`spec.md` requires them to be obvious and one-tap rather than buried.

**Contrast with the NFC chip:** the chip has CMAC plus a monotonic counter, so
a replayed scan is detectable. The digital card has neither. It is
deliberately the weaker artifact, traded for existing on day one without
manufacturing or shipping.

---

## R11 — Digital card first, physical chip later

**Decision.** Onboarding creates a live digital card. Physical NTAG 424 cards
follow on demand or as a marketing hook. `cards` splits into card identity
(`cards`) and chip binding (`card_chips`); a card may have no chip.

**Reasoning.** "Create the card instantly" is incompatible with a physical
object that must be written and shipped. Rather than fake it or defer the
moment, the card becomes a real digital artifact that works immediately, and
the chip becomes an upgrade.

The model split is the load-bearing part. `cards.uid` as primary key conflates
the product object with the hardware, and every day that conflation survives
makes it more expensive to undo. No fulfilment state machine is built yet, but
`card_chips` must not preclude one.

---

## R12 — Splash retained

**Decision.** Keep the splash. Under 1s, no tagline, session check behind it.
The login carousel is a separate surface.

**Reasoning.** An earlier draft killed the splash on the grounds that it
delays entry. Owner overruled, correctly: `009`'s original constraint already
made it sub-second and content-free, which is not a gate — it is the window
in which the session check runs anyway. The carousel does the brand storytelling
on the login screen, where a logged-out user is already stopped. Returning
users see neither.

---

## R13 — Google sign-in is a credential, not an identity

**Decision.** Retained on the login surface with no additional screen. Phone
remains the canonical account identity; a Google-first signup still completes
phone verification before an account exists.

**Reasoning.** The concern with social sign-in was never the button, it was
the second identity path and the account-merge problem behind it. Treating
Google as a *linked credential* on a phone-canonical account removes that
entirely: there is exactly one account key, and Google is a faster way to
reach it.

It also pays for itself at the card step — a Google sign-in already supplies a
verified email, so the optional-email ask becomes a confirmation rather than
typing (FR-005).

**Still true:** a Google account is not the Aadhaar-linked mobile, so this
does nothing for ABHA. Phone verification remains mandatory for that reason
as well as for account-recovery (`docs/backlog.md` §1).

---

## R14 — Home gets both a walkthrough and a button

**Decision.** A dismissible first-run walkthrough explains family profiles and
linking and lets the user choose whether to add family. A persistent "Add
family" action stays on Home afterward. Per-profile setup chips from `001`
§4.1 are unchanged.

**Reasoning.** These do different jobs and neither substitutes for the other.
The walkthrough teaches a concept the user has never seen; the button is the
durable affordance for the moment a time-poor caregiver finally has one. A
walkthrough alone leaves nothing behind; a button alone assumes the user
already understands what family linking means.

**Principle VIII check.** Home Stays Quiet is non-negotiable and explicitly
governs "any future surface." A persistent Add-family action passes: it is an
*action affordance* attached to family context, not a content slot, not a
feed, and it does not refresh or harvest attention. The walkthrough passes
because it runs once and never returns.

The three-layer split — app-level walkthrough, app-level button,
profile-level chips — needs `001` §4.1 amended to say so, since §4.1 currently
implies empty states and chips do all the guiding.

---

## R15 — Auth provider is unresolved, and residency is why

**Status: open.** Owner is considering Firebase for auth and email, "depending
on price and compliance rules." Two facts the decision must account for.

**Data residency.** "Stored only in India" is approved copy, printed on the
Trust screen, and a Principle VI commitment. Firebase Authentication /
Identity Platform does not guarantee India-resident storage of auth records
the way the locked DigitalOcean-Bangalore stack does. Adopting it means either
proving residency or removing the claim from every surface that makes it —
and the claim is load-bearing for the trust positioning, not decoration.

FCM is unaffected and stays: it carries push tokens, not identity records.
The two being the same vendor is a coincidence, not a precedent.

**Firebase does not send transactional email.** It sends only its own
auth-template mail. Firebase relocates this dependency; it does not remove it.
Resolved separately by R17 — mail is self-hosted.

Adopting Firebase Auth also displaces Django/DRF + SimpleJWT, a **locked**
stack decision — a `CLAUDE.md` change with a changelog entry, not an
implementation detail. Note the tension: R17 puts mail on our own Bangalore
server specifically to keep data in-country, which argues against moving auth
records out of it.

---

## R17 — Transactional email is self-hosted

**Decision.** Card receipts and other transactional mail send from our own
server, not a third-party provider.

**Reasoning.** Consistency with the residency story is the main argument: mail
containing a user's emergency-card details is PHI-adjacent, and self-hosting on
the Bangalore droplet keeps it inside the same boundary as everything else the
Trust screen describes. It also avoids adding a vendor to the DPDP processor
list, where every third party is a contract to align (`compliance-baseline.md`
§7 flags vendor alignment as the highest-risk DPDP item).

**What this costs, and it is not nothing.** Self-hosted SMTP is a deliverability
problem before it is a code problem:
- **DigitalOcean blocks outbound port 25 on new droplets by default.** It takes
  a support request to unblock, and it is not always granted.
- **SPF, DKIM, and DMARC** all need correct records on the sending domain, or
  receipts go to spam.
- **IP reputation must be warmed.** A cold IP sending transactional mail to
  Gmail addresses starts in a hole.

None of this is a blocker; all of it is ops work that must be scheduled rather
than discovered. If deliverability proves unworkable, the fallback is a
provider with an India region — a residency question, not a rewrite, since the
sending interface is SMTP either way.

**Revisit if:** receipt deliverability drops below a usable rate, or the volume
outgrows a single droplet.

---

## Open threads

| Thread | Blocking | Where it lands |
|---|---|---|
| Auth provider + residency (R15) | Compliance copy, locked stack | `CLAUDE.md`, Trust screen |
| Self-hosted mail: port 25, SPF/DKIM/DMARC, IP warmup (R17) | FR-012 delivery | Ops, before FR-012 ships |
| ~~Track B constitution ratification (R16)~~ — **closed 6 Sep 2026**, split abolished instead | Governance, on the WASA path | `.specify/memory/constitution.md` (v3.0.0) |
| Carousel content — which 3–4 frames | S1 copy | Soham |
