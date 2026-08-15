# ABDM conformance checklist — [FEATURE NNN-slug]

**Copy to `specs/NNN-slug/checklists/abdm.md`. Required for any feature
touching an ABDM surface: ABHA, FHIR, consent artifacts, care contexts, HIP/HIU
flows. Review against `plan.md` BEFORE implementation.**

Facts, versions, and dates: `docs/compliance-baseline.md` (verified 10 Aug
2026 — re-verify before relying on a version pin).

Mark **PASS**, **N/A** (with reason), or **GAP** (with owner). Most items are
`[B→A]`: on Track A the question is not "did we build it" but **"did we make
it expensive to build later."** Answer that question, not the other one.

---

## Scope declaration — answer first

- [ ] Which milestone does this feature touch? **M1** (ABHA identity) / **M2**
      (HIP — share) / **M3** (HIU — fetch) / **none**.
- [ ] Track A or Track B? If Track A, the ABDM surface MUST be a labelled mock
      — Track A performs no real ABDM exchange (Principle IV, `CLAUDE.md`).
- [ ] If Track A: every mock is tagged `# DEMO-MODE` with its Track B
      replacement named (Principle XIV `[A]`).

> TDC Health is an **M3-shaped product** (a PHR pulls records) with M1 as its
> entry requirement. TDC Clinic (CARE fork) is the M2-shaped system, in its own
> repo, certified separately. If this feature seems to need M2, check that it
> belongs in this repo at all.

## FHIR conformance — Principle XII `[A]` when FHIR is present

- [ ] Base version is **FHIR R4.0.1**. R5 appears nowhere.
- [ ] Profiles target **NRCeS FHIR IG for ABDM v6.5.0** (released). The
      v7.0.0 IG is **draft** — not built against.
- [ ] Released IG version re-verified at <https://nrces.in/ndhm/fhir/r4/> this
      session; if it has moved past 6.5.0, the pin in
      `docs/compliance-baseline.md` and Principle XII are updated before
      proceeding.
- [ ] Clinical/billing artifacts are profiles on the R4.0.1 `Composition`
      resource, per the IG.
- [ ] Resource mapping documented in the feature's `data-model.md` — which TDC
      table maps to which FHIR profile and field.
- [ ] Non-foreclosure: nothing in the internal model makes a required IG field
      unrecoverable (dropped provenance, lossy merge, free-text where the IG
      requires a coded value).

## Consent — Principles II and XV

- [ ] **The two consent systems are not conflated.** ABDM consent is
      per-patient, on the ABDM network. `caregiver_grants` is TDC's own family
      layer and is **not** ABDM delegation. Which one does this feature touch?
      State it.
- [ ] No screen, copy, log line, or comment describes a card tap as consent
      (Principle II, NON-NEGOTIABLE).
- [ ] Consent is created once, explicitly, in-app, logged, and revocable.
- [ ] `[B→A]` Consent artifact fields are not foreclosed: purpose, HI types,
      date range, care contexts, expiry can all be represented later without a
      migration that rewrites history.
- [ ] `[B]` Consent artifacts are cryptographically signed and signature is
      verified on receipt.
- [ ] Consent transactions are logged auditably (ties to `access_logs`,
      Principle XI `[A]`).

## Retention & erasure — Principle XV

- [ ] `[B]` `dataEraseAt` is honoured: fetched data is purged or anonymised at
      consent expiry. **This duty is ours** — as a PHR, TDC is the HIU.
- [ ] `[B]` Purge on consent revocation is implemented and testable.
- [ ] `[B→A]` Non-foreclosure: records carry provenance (`source`) and remain
      individually deletable. No aggregate makes a single record's origin
      unrecoverable or its deletion a rewrite. Regenerable denormalized
      snapshots (`emergency_payload`) are fine; lossy ones are not.

## Encryption in transit — Principle XIII

- [ ] `[B]` Data exchanged with the ABDM network uses **Fidelius**: ephemeral
      Curve25519 ECDH keypair + fresh 32-byte nonce per transaction, HKDF →
      AES-256-GCM session key, no shared-secret reuse.
- [ ] `[B]` Implemented from the **official Fidelius CLI reference
      parameters**, not default X25519 settings in a general-purpose library
      (known cause of gateway handshake failures — baseline §4).
- [ ] `[B→A]` Non-foreclosure: payload assembly is separable from transport;
      no persisted or contract-level shape assumes plaintext at handoff.

## Registry & identity

- [ ] `[B]` M1: ABHA creation, verification, and linking follow current NHA
      API contracts; patient discovery implemented per NHA test cases.
- [ ] `[B]` M2 only: facility registered in HFR; care contexts created and
      linked. **Not this repo** — see scope note above.
- [ ] `[A]` Track A's ABHA field is a labelled manual/mock input and is never
      presented as a verified ABHA.

## Certification readiness

- [ ] `[B]` Feature is covered by NHA's official functional test-case
      templates for its milestone; gaps named.
- [ ] `[B]` Feature's endpoints are in WASA scope and have passed the security
      checklist (`checklists/security.md`) — WASA is a **precondition for M1**,
      not a final step.
- [ ] `[A]` No copy anywhere claims or implies "ABDM certified". Approved
      framing only: "built on India's ABDM framework (integration in
      development)".
- [ ] No timeline is quoted publicly that contradicts baseline §2 (6–9 months
      greenfield sandbox→production; WASA alone 3–6 weeks).
