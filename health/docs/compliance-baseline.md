# Compliance baseline — ABDM M1/M2/M3, WASA, DPDP

**Verified: 10 Aug 2026. Re-verify before any Track B planning session, and
before quoting any version number or date in a pitch.**

This document holds the *facts* — external requirements, with sources and the
date they were checked. `.specify/memory/constitution.md` holds the *rules* we
derive from them. Keep them separate: the constitution must stay short and
testable, and it must not rot every time NHA publishes a new IG version.

Nothing here is binding by itself. Principles X–XV in the constitution are.

---

## 1. ABDM milestones

| Milestone | Role | What it certifies |
|---|---|---|
| **M1** | Identity | ABHA creation, verification, linking; patient discovery |
| **M2** | HIP (Health Information Provider) | Share FHIR records on consent; care-context creation & linking; facility registered in HFR |
| **M3** | HIU (Health Information User) | Fetch records from other providers via consent |
| M4 | NHCX | Digital insurance claims — not in TDC scope |

**TDC Health is an M3-shaped product** (a PHR pulls records) with M1 as its
entry requirement. TDC Clinic (the CARE fork) is the M2-shaped system. This
split matters: the two certify separately, and they are separate repos.

## 2. Sandbox → production is a four-stage gate

Sequential; each stage blocks the next.

1. **Functional testing** — third-party validation of M1/M2/M3 workflows against
   NHA's official test-case templates (ABHA creation, verification, discovery,
   consent handling, data transfer), plus non-functional/edge-case testing.
   Iterative: fix and resubmit until all cases pass.
2. **WASA** — Web Application Security Assessment by a **CERT-In empanelled**
   auditor. See §3.
3. **NHA document submission & committee review** — bundle: M1–M3 milestone
   approvals, WASA final report, **Safe-to-Host certificate**, deployment
   details, plus whatever the current ABHA certificate guidelines list.
4. **Production access & go-live** — credentials issued.

**Timeline: 6–9 months greenfield**, 4–6 months layering onto an existing HMS
with a structured integration layer. WASA alone is 3–6 weeks depending on scope
and remediation speed.

> This is the number that matters for planning. Any roadmap that shows TDC
> Health in ABDM production less than two quarters after Track B starts is
> wrong, and saying otherwise in a pitch is a claim we cannot defend
> (Principle VI).

## 3. WASA — what is actually tested

**WASA is a precondition for M1**, not a final step after M3. Only a CERT-In
empanelled auditor can issue the report and Safe-to-Host certificate NHA
accepts.

Assessed scope:

- **OWASP Top 10 for Web *and* OWASP API Security Top 10** — both. A single
  critical finding stalls certification.
- Authentication mechanisms
- Authorization controls — appropriate access levels per user
- Data encryption **at rest and in transit**
- Session management — timeout, cookie handling, token security
- API security — endpoints interacting with ABDM services

The highest-frequency killer in this class of app is broken object-level
authorization (OWASP API #1): an endpoint that trusts a client-supplied ID
instead of deriving the caller's permitted scope server-side. That is why
Principle X exists and why it binds on Track A even though Track A is a demo.

## 4. Fidelius — PHI encryption in transit

ABDM's end-to-end encryption protocol for health data moving between HIP and
HIU. Construction:

- **ECDH key exchange on Curve25519.** Both parties generate a **fresh,
  ephemeral keypair plus a 32-byte random nonce for every transaction**. The
  shared secret is never reused across sessions.
- The two nonces are combined to derive the **salt and IV**.
- **HKDF** over the shared secret + salt produces a session-specific
  **AES-256-GCM** key, which encrypts the FHIR bundle.

**Implementation warning, learned the expensive way by others:** teams that
build ECDH from a general-purpose crypto library using default
Curve25519/X25519 parameters get gateway handshake failures ("encoded key spec
not recognized"). Base the implementation on the official Fidelius CLI
reference parameters and key-generation routines. Python option:
`dimagi/pyfidelius`. Reference: `mgrmtech/fidelius-cli`.

Track A does no ABDM exchange, so Fidelius is not built now — but see
Principle XIII's non-foreclosure clause.

## 5. FHIR profile conformance

- Base: **FHIR R4.0.1**. R5 is not in scope for ABDM.
- Profile set: **NRCeS "FHIR Implementation Guide for ABDM"**.
- **Current released version: v6.5.0** — <https://nrces.in/ndhm/fhir/r4/>
- **v7.0.0 exists but is Draft as of 2026-07-15** —
  <https://www.nrces.in/preview/ndhm/fhir/r4/index.html>

Clinical and billing artifacts are profiles on the FHIR R4.0.1 `Composition`
resource.

**Pin: v6.5.0.** Do not build against the v7.0.0 draft. Re-check the released
version at the top of any Track B FHIR work — a draft becoming a release is
exactly the kind of change that invalidates a pin silently.

## 6. Consent artifacts, retention, erasure

- Consent must be **explicit, informed, granular (purpose, duration, data
  type), revocable, and auditable**.
- Consent artifacts are **cryptographically signed JSON** documents, signed
  with HIP and HIU gateway keys, describing which care contexts, which HI
  types, which date range, and how long data may be stored.
- **`dataEraseAt` is a legal obligation.** Every consent carries an expiry
  after which the HIU must purge or anonymise fetched data. As a PHR, TDC is
  the HIU — this obligation lands on us, not on someone else.
- ABDM mandates encryption, **audit logs**, role-based access, and breach
  notification; consent transactions must be logged in an auditable manner.

Note the shape mismatch to keep straight: ABDM consent is **per-patient** and
lives on the ABDM network. TDC's family layer (`caregiver_grants`) is our own
construct and is **not** ABDM delegation. Already locked in `CLAUDE.md`;
restated here because conflating them is the most likely compliance error.

## 7. DPDP Act / Rules — dates

- DPDP **Rules notified November 2025**, starting an 18-month runway.
- **13 Nov 2026 — Consent Manager framework becomes operational.** Nearest
  hard date.
- **13 May 2027 — full substantive compliance**: notice, consent, security
  safeguards, breach reporting, data-principal rights.
- Applies to any business processing digital personal data in India.
  Responsibility rests with the **Data Fiduciary** even where a Data Processor
  does the processing; processor agreements must carry security provisions.
- Healthcare is flagged high-risk because of sensitive data plus frequent
  third-party lab/diagnostic integrations — **vendor contract alignment is the
  highest-risk item**, which is a commercial task, not an engineering one.

TDC is a Data Fiduciary. Track A is pre-launch with seeded fictional data and
no real data principals, so no obligation is live yet; the obligations attach
at first real user.

---

## Sources

Checked 10 Aug 2026. Vendor blogs are secondary — treat NRCeS and official NHA
material as authoritative and re-derive anything load-bearing before it goes in
a submission.

- [NRCeS — FHIR Implementation Guide for ABDM v6.5.0 (released)](https://nrces.in/ndhm/fhir/r4/)
- [NRCeS — FHIR IG for ABDM v7.0.0 (Draft, 2026-07-15)](https://www.nrces.in/preview/ndhm/fhir/r4/index.html)
- [NRCeS — Implementation Guide for Adoption of FHIR in ABDM and NHCX (PDF)](https://www.nrces.in/download/files/pdf/Implementation_Guide_for_Adoption_of_FHIR_in_ABDM_and_NHCX.pdf)
- [mgrmtech/fidelius-cli — encryption & decryption guidelines for FHIR data in ABDM](https://github.com/mgrmtech/fidelius-cli/blob/main/abdm/Encryption%20and%20Decryption%20Implementation%20Guidelines%20for%20FHIR%20data%20in%20ABDM.md)
- [dimagi/pyfidelius — Python ECDH/AES-GCM port](https://github.com/dimagi/pyfidelius)
- [ISECURION — ABDM M1 WASA testing guide & checklist](https://isecurion.com/abdm-m1-wasa-testing-complete-guide.html)
- [QRC Solutionz — Understanding WASA audits](https://www.qrcsolutionz.com/blog/understanding-wasa-audits-abdm-compliance-simplified)
- [Nirmitee — ABDM certification process, sandbox to production](https://nirmitee.io/blog/abdm-certification-process-sandbox-to-production-guide/)
- [Nirmitee — Building an ABDM HIU: M3 flow & reference architecture](https://nirmitee.io/blog/building-abdm-hiu-from-scratch-m3-flow-reference-architecture/)
- [Caladrius Health — How clinical data actually flows under ABDM](https://caladriushealth.ai/blog/2026/06/19/How-Records-Move/)
- [Scrut — DPDP Rules 2025 practical guide & checklist](https://www.scrut.io/post/dpdp-rules)
- [Vinsys — DPDP compliance deadlines, Consent Manager Nov 2026](https://www.vinsys.com/blog/dpdp-act-compliance-deadline-nov-2026-for-consent-manager)
