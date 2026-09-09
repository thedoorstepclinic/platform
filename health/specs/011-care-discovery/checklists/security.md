# Security checklist — 011-care-discovery

> **SUPERSEDED 6 Sep 2026** by
> [`specs/012-appointment-booking/checklists/security.md`](../../012-appointment-booking/checklists/security.md),
> which carries every item here forward. Retained as the decision record.
>
> **Vocabulary note:** this file predates the abolition of the Track A / Track B
> split (constitution v3.0.0) and its `[A]` / `[B→A]` markers and "Track B
> replacement" language are **historical**, left as written. There are no
> binding classes any more: **every item blocks implementation**, and a demo
> path names its real path plus any blocking gate and gate owner (Principle XIV).

**Reviewed against `spec.md` (2026-08-20), pre-implementation — no code and no
`plan.md` exist yet. "PASS" here means *the spec commits to this*, not *it is
built and tested*. Re-run after implementation, before demo rehearsal.**

Aligned to what a CERT-In empanelled WASA auditor tests: OWASP Top 10 for Web
**and** OWASP API Security Top 10, authentication, authorization, encryption in
transit and at rest, session management, API security. One critical finding
stalls ABDM certification — see `docs/compliance-baseline.md` §3.

Mark each item **PASS**, **N/A** (with a reason), or **GAP** (with an owner).
A `GAP` on any item marked `[A]` blocks implementation. A `GAP` on `[B→A]` is
allowed only if the non-foreclosure clause still holds — say why.

**Summary:** 1 `[A]`-blocking GAP (Audit #1 — the `access_logs` write on booking
is specced as FR-016 but has no design behind it yet, same open question as
`001`'s). 5 non-blocking GAPs, four of which are consequences of the three owner
overrides recorded in `spec.md` *Decisions taken* (ratings, HID chip, maps SDK).
The one item this feature genuinely changes about the app's threat surface is
**an unscoped, non-PHI read path** — the directory — which did not previously
exist. It is intentional; the risk is that a later change quietly attaches
profile data to it.

---

## Authorization — Principle X `[A]`

- [x] **PASS** — the booking write is profile-scoped. `appointments` is created
      through `010`'s endpoint, whose queryset derives the profile from the JWT
      principal plus unrevoked `caregiver_grants` (FR-015).
- [x] **PASS** — `get_object()` resolves from the scoped queryset. Clinic,
      doctor and slot ids in the request body are validated as *existing*, never
      as *authorizing*: they grant no access to anything profile-owned.
- [x] **PASS** — no endpoint decides access from a client-supplied id. The
      `profileId` in the discovery route is a navigation parameter only; the
      directory response is identical regardless of it, and the booking write
      re-derives scope server-side.
- [x] **PASS** — grant revocation denies the booking write on the next request;
      no cached scope.
- [ ] **GAP** (Adi, non-blocking) — second-user test not yet written: user B must
      not be able to create an appointment against user A's profile by id.
- [x] **N/A** — non-sequential ids: the directory is public reference data;
      enumerating clinics leaks nothing. `appointments` ids inherit `010`'s
      treatment.

### Intentional unscoped read — recorded per Principle X

`GET /clinics`, `/clinics/{id}`, `/clinics/{id}/nearby`, `/clinics/{id}/doctors`,
`/specialties` and `/doctors/{id}/slots` are **deliberately unscoped**. They serve
reference data with no profile linkage, no PHI and no per-user variation.

- [x] **PASS** — every one of these responses is provably profile-independent:
      no field derives from the caller, and no `profileId` reaches the queryset.
- [ ] **GAP** (Adi, non-blocking) — needs a regression guard. The failure mode is
      not today's code, it is a later "show which clinics this family has visited"
      convenience that turns a public list into a PHI leak. Add a test asserting
      these serializers expose no profile-derived field.

## Authentication & session

- [x] **PASS** — every directory endpoint carries an explicit
      `IsAuthenticated` permission class. None is public.
- [x] **PASS** — no endpoint in this feature is intended to be public.
- [x] **N/A** — token lifetime/refresh unchanged from `001`.
- [x] **N/A** — logout/revoke unchanged from `001`.
- [x] **PASS** — the seeded directory, wait/queue values, ratings and HID chip
      each carry a `# DEMO-MODE` tag naming their Track B replacement (FR-017).

## PHI handling — Principle XIII

- [x] **PASS** — no PHI in discovery URLs. `profileId` is an opaque id, the
      established pattern across `008`'s route map; clinic/doctor/slot ids are
      reference ids.
- [x] **PASS** — no PHI in directory logs. Search queries are clinic/doctor/
      specialty terms. **Because symptom search was rejected (decision 4), the
      search log cannot contain a health complaint** — this is the concrete
      security consequence of that product decision and the main reason to keep
      it rejected in Track A.
- [x] **PASS** — error responses do not vary by existence of profile-owned data.
- [x] **PASS** — HTTPS enforced; unchanged from `001`.
- [x] **N/A** — Fastlane untouched by this feature.
- [x] **N/A** — no payload handoff; no Fidelius surface.

## Audit — Principle XI `[A]`

- [ ] **GAP `[A]`** (Adi, blocks the booking-write task) — FR-016 requires an
      `access_logs` row on booking confirm, in the same transaction, failing the
      request if the log write fails. The design for that write lives in `001`
      and is itself still open there; this feature cannot close it alone but
      must not ship its write before `001` does.
- [x] **PASS** — directory browsing correctly emits *no* `access_logs` row. No
      PHI is read, and logging directory reads would dilute the table that
      Principle XI exists to keep meaningful.
- [x] **PASS** — no bypass path added: no raw SQL, bulk op, management command
      or direct file serve touches PHI in this feature. `seed_demo.py` writes
      reference data only.

## Input & API surface

- [x] **PASS** — `q` and `specialty` are serializer-validated server-side.
      `q` is used as an ORM `icontains` parameter, never interpolated SQL.
- [x] **N/A** — no file uploads in this feature.
- [x] **N/A** — no uploaded-file serving. Clinic/doctor imagery is static seeded
      asset references, not user content.
- [x] **PASS** — directory serializers use explicit `fields`, never `__all__`.
      This matters more here than usual: `__all__` on `clinics` would ship
      internal seeding fields, and `__all__` on `doctors` would ship the raw
      `hid_masked` source if one is ever added.
- [ ] **GAP** (Adi, non-blocking) — no rate limit on `GET /clinics?q=`. It is an
      authenticated, non-PHI, read-only endpoint, so the exposure is cost rather
      than disclosure; tag as `# DEMO-MODE` with "throttle in Track B" rather
      than leaving it unremarked.

## Secrets & config

- [ ] **GAP** (Adi, non-blocking, **new risk introduced by decision 8**) — the
      maps SDK needs an API key in the client. A key in a demo APK is a key that
      leaks. Before any build ships: restrict by package ID
      (`in.thedoorstepclinic.health`) + signing-certificate fingerprint, restrict
      to the maps APIs actually used, and set a quota cap. The key must not be
      committed — inject at build time.
- [x] **PASS** — no other secrets in seed data. Fictional clinics, fictional
      doctors, seeded ratings and a seeded masked HID string are all invented
      values, not credentials.
- [x] **N/A** — `DEBUG`/CORS/ALLOWED_HOSTS unchanged from `001`.

## Copy — Principle III `[A]`

- [x] **PASS** — no cipher names; this feature's copy has no security claims.
- [x] **PASS** — no banned phrases. FR-014 specifically forbids attaching
      "ABDM certified", HPR, registry or verification copy to the HID chip,
      which is where that banned phrase would most plausibly have crept in.
- [ ] **GAP** (Adi, non-blocking, **consequence of decisions 6 and 7**) — two
      elements assert something the build does not do, and the mitigations are
      copy-level, so they need a copy review pass rather than a code test:
      the **star rating** (a number with no measurement behind it) and the
      **HID chip** (an identifier with no registry behind it). FR-013 and FR-014
      constrain them to the narrowest honest form — rating with no corpus, chip
      with no claim. Re-read both strings before demo; if a reviewer asks "what
      is 4.8 out of?" the demoer needs an answer that is not a bluff.

## Adversarial pass

Per `CLAUDE.md`: anything touching auth, grants or Fastlane gets a "how would I
abuse this?" pass before merge.

- [x] **PASS** — abuse pass done, findings recorded here:
  1. **Directory as a PHI side channel.** Today it is clean. The abuse is a
     future convenience field ("last visited", "your doctor") turning a public
     endpoint into a leak. Guard: the serializer regression test above.
  2. **Booking against someone else's profile.** Clinic/doctor/slot ids are
     unauthenticated-ish reference ids and a caller could pair any of them with
     any `profile_id`. Scope derivation is server-side (FR-015), so this fails —
     but it is the single most likely thing to be got wrong at implementation
     time, because the request body *looks* like it carries the authorization.
  3. **Slot double-booking / race.** Two confirms on one `slots` row. Track A is
     single-user demo data, so impact is cosmetic; still, take the row lock or
     a unique constraint rather than a read-then-write.
  4. **Directions hand-off.** An address string handed to an external app is an
     intent payload. Keep it a plain address, never a URL assembled from
     server-supplied text, so a seeded field cannot become an open-redirect into
     an arbitrary app.
  5. **Coordinate leak (FR-011).** Not a classic security finding but the same
     class of harm as the Ruby Hall name leak: a fictional clinic pinned on a
     real hospital implies a relationship. Verify each seeded coordinate lands
     on neutral ground before the seed merges.
- [x] **N/A** — no card/Fastlane surface in this feature.
