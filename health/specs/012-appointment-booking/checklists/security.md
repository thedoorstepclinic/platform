# Security checklist — 012-appointment-booking

**Reviewed against `spec.md` (2026-09-06), pre-implementation — no code and no
`plan.md` exist yet. "PASS" here means *the spec commits to this*, not *it is
built and tested*. Re-run after implementation, before demo rehearsal.**

Supersedes `specs/011-care-discovery/checklists/security.md`. Every item from
that review is carried forward; the GAPs it left open are still open and are
marked **(carried)**. What `012` adds to the threat surface is not discovery —
it is the **appointment write path**: three endpoints that mutate a
profile-owned row (`create`, `cancel`, `reschedule`), where `011` had one.

Aligned to what a CERT-In empanelled WASA auditor tests: OWASP Top 10 for Web
**and** OWASP API Security Top 10, authentication, authorization, encryption in
transit and at rest, session management, API security. One critical finding
stalls ABDM certification — see `docs/compliance-baseline.md` §3.

Mark each item **PASS**, **N/A** (with a reason), or **GAP** (with an owner).
**Every item blocks implementation** — the binding classes were abolished
6 Sep 2026 (constitution v3.0.0). N/A only where the feature has no such
surface, with the reason stated.

> **Rescope, same day.** The spec this reviews was rewritten hours later for
> production: booking became a **request the clinic accepts or declines**,
> payments came **in scope**, and the queue feed became real. Three findings
> below are materially changed by that and are marked **[RESCOPE]**; a full
> re-review is owed once `plan.md` exists.

**Summary:** 1 blocking GAP (Audit #1 — the `access_logs` write, carried
from `011`, still designed only in `001`). 7 non-blocking GAPs. Two risks are
**new in `012`** and neither existed in `011`: the **cancel/reschedule
endpoints**, which are the app's first destructive profile-scoped writes
reachable by a caregiver, and the **slot race**, which `011` correctly called
cosmetic for a single-user demo and which stops being cosmetic the moment two
phones book at the same meetup.

---

## Authorization — Principle X

- [x] **PASS** — every `appointments` endpoint derives its queryset from the
      JWT principal's own profiles ∪ unrevoked `caregiver_grants` (FR-035).
      This now covers **reads** (`GET /profiles/{id}/appointments`) as well as
      writes; `011` only had a write to scope.
- [x] **PASS** — `get_object()` resolves from the scoped queryset on cancel and
      reschedule. An `apptId` in the URL is an addressing parameter, never an
      authorizing one.
- [x] **PASS** — clinic, doctor and slot ids in a request body are validated as
      *existing*, never as *authorizing*. They grant no access to anything
      profile-owned.
- [x] **PASS** — `POST /appointments` takes `{profile_id, slot_id}` and
      **validates `profile_id` against the derived set rather than trusting
      it** (FR-035). The server resolves clinic, doctor and `slot_ts` from the
      slot; the client cannot supply a mismatched triple.
- [x] **PASS** — no endpoint decides access from a client-supplied id. The
      `profileId` in a discovery route is a navigation parameter only; the
      directory response is identical regardless of it.
- [x] **PASS** — grant revocation denies the write on the next request; no
      cached scope. A revoked caregiver cannot cancel an appointment they could
      have cancelled a minute earlier.
- [ ] **GAP** (Adi, non-blocking, **carried from `011`**) — second-user test not
      written: user B must not create an appointment against user A's profile
      by id.
- [ ] **GAP** (Adi, non-blocking, **new in `012`**) — the same test is needed
      for the two **destructive** verbs, and they are the higher risk: user B
      must not be able to **cancel or reschedule** user A's appointment by
      `apptId`. Creating a spurious appointment is noise; cancelling someone
      else's is the appointment silently not happening. Test both, and test
      them after a grant revocation, not only before.
- [x] **N/A** — non-sequential ids: the directory is public reference data;
      enumerating clinics leaks nothing. `appointments` ids are scoped by the
      queryset above, so enumeration returns 404s, not rows.

### Intentional unscoped read — recorded per Principle X (carried from `011`)

`GET /clinics`, `/clinics/{id}`, `/clinics/{id}/nearby`, `/clinics/{id}/doctors`,
`/specialties` and `/doctors/{id}/slots` are **deliberately unscoped**. They
serve reference data with no profile linkage, no PHI and no per-user variation.

- [x] **PASS** — every one of these responses is provably profile-independent:
      no field derives from the caller, and no `profileId` reaches the queryset.
- [ ] **GAP** (Adi, non-blocking, **carried**) — needs a regression guard. The
      failure mode is not today's code, it is a later "show which clinics this
      family has visited" convenience that turns a public list into a PHI leak.
      Add a test asserting these serializers expose no profile-derived field.
- [x] **PASS** — **`012`'s amendment to `GET /doctors/{id}/slots` does not
      change its scoping class.** Returning taken slots flagged rather than
      filtered exposes *that a slot is taken*, never *by whom*. `is_booked` is a
      boolean with no profile linkage, and the serializer must not gain one —
      an "already booked by your family" convenience would end this exemption
      and is explicitly out of scope.

## Authentication & session

- [x] **PASS** — every endpoint in this feature carries an explicit
      `IsAuthenticated` permission class. None is public.
- [x] **PASS** — no endpoint here is intended to be public. In particular, no
      appointment is reachable by an unauthenticated link — unlike the
      emergency card, this feature mints **no bearer tokens and no public
      URLs**, and must not grow one (a "share your appointment" link would be
      a new public PHI surface with none of `007`'s rotation machinery).
- [x] **N/A** — token lifetime/refresh unchanged from `001`.
- [x] **N/A** — logout/revoke unchanged from `001`.
- [x] **PASS** — seven `# DEMO-MODE` tags specified (FR-029), each naming its
      real path: seeded directory · seeded wait/queue · seeded
      ratings · seeded HID chip · `slots.is_booked` · seeded consult fees ·
      derived `lapsed`.

## PHI handling — Principle XIII

- [x] **PASS** — no PHI in booking URLs. `profileId` and `apptId` are opaque
      ids, the established pattern across `008`'s route map.
- [x] **PASS** — no PHI in directory logs. Search terms are clinic/doctor/
      specialty names. **Because symptom search is rejected (FR-003), the search
      log cannot contain a health complaint** — the concrete security
      consequence of that product decision, and the main reason it stays
      rejected rather than merely deferred.
- [x] **PASS** — **`012` FR-010 removes the one field that would have made an
      appointment row clinically sensitive.** No reason-for-visit, no symptom,
      no free-text clinical note. An appointment is *who, where, with whom,
      when* — identifying and schedule-revealing, but it carries no diagnosis.
      This is a data-minimisation win worth protecting: the field returns in
      **[RESCOPE]** the clinic-side surface now exists, so the field is
      re-opened as optional PHI — scoped, logged, and excluded from
      notification bodies. Not built until specced (spec §B5).
- [x] **PASS** — error responses do not vary by existence of profile-owned
      data. A 404 on someone else's `apptId` is indistinguishable from a 404 on
      a nonexistent one.
- [x] **PASS** — HTTPS enforced; unchanged from `001`.
- [x] **N/A** — Fastlane untouched by this feature.
- [x] **N/A** — no payload handoff; no Fidelius surface.

### Local notifications as a PHI surface (new in `012`)

Appointment reminders put a name, a doctor, a clinic and a time **on a lock
screen** (FR-022). `006`'s dose reminders already established this surface;
`012` adds two more per appointment.

- [x] **PASS** — the reminder body names the profile, doctor, clinic and time.
      No diagnosis, no specialty-implying-condition phrasing, no token-only
      medical inference. *"Asha's appointment with Dr. Kavya Rane"* is a
      schedule; *"Asha's diabetes follow-up"* would be a diagnosis on a lock
      screen, and MUST NOT be used even though the specialty is known.
- [ ] **GAP** (Adi, non-blocking) — no copy review of the reminder strings yet.
      They are the only PHI-adjacent text this feature puts outside the app,
      and the specialty-name temptation above is a one-word edit away.
- [x] **PASS** — notifications are **device-local** (`flutter_local_notifications`,
      `006`'s primitive). No server push, so no appointment content transits FCM.

## Audit — Principle XI

- [ ] **GAP** (Adi, blocks the appointment-write tasks, **carried from
      `011`**) — FR-036 requires an `access_logs` row on booking confirm, in the
      same transaction, failing the request if the log write fails. The design
      for that write lives in `001` and is still open there; this feature cannot
      close it alone and must not ship its writes before `001` does.
- [x] **PASS** — **`012` widens the requirement correctly**: create,
      reschedule **and cancel** each write a log row (FR-036), where `011` only
      logged the create. A cancellation is a state change to a profile-owned
      record and is exactly the kind of event Principle XI exists to make
      answerable — *"who cancelled Aai's appointment?"* is a question the family
      will actually ask.
- [x] **PASS** — `cancelled_by` records the acting user (FR-038), so a
      caregiver cancelling a dependent's appointment is attributable in the row
      itself, not only in the log.
- [x] **PASS** — directory browsing correctly emits *no* `access_logs` row. No
      PHI is read, and logging directory reads would dilute the table Principle
      XI exists to keep meaningful.
- [x] **PASS** — no bypass path added: no raw SQL, bulk op, management command
      or direct file serve touches PHI in this feature. `seed_demo.py` writes
      reference data only and seeds zero appointments (FR-041).
- [x] **PASS** — **`lapsed` needs no log row and no job.** It is derived at read
      time (FR-017), so there is no background writer mutating profile-owned
      rows outside a request context — which would have been an unauditable
      actor in a table whose whole purpose is naming the actor.

## Input & API surface

- [x] **PASS** — `q` and `specialty` are serializer-validated server-side. `q`
      is an ORM `icontains` parameter, never interpolated SQL.
- [x] **PASS** — `slot_id` is validated as existing, unbooked, in the future,
      and **belonging to the doctor and clinic in the route** before any write.
      A slot id from another clinic paired with this clinic's route is a
      rejected request, not a cross-wired appointment.
- [x] **PASS** — the cutoff (`cancellation_window_min`) is enforced
      **server-side** for both cancel and reschedule (FR-018). A client-side-only
      cutoff is a suggestion.
- [x] **PASS** — `reschedule_count` is enforced server-side (FR-018). The cap
      exists to bound the row chain, so a client that ignores it must still be
      refused.
- [x] **N/A** — no file uploads in this feature.
- [x] **N/A** — no uploaded-file serving. Clinic/doctor imagery is static
      seeded asset references, not user content.
- [x] **PASS** — serializers use explicit `fields`, never `__all__`. Matters
      more here than usual: `__all__` on `clinics` ships internal seeding
      fields, `__all__` on `doctors` ships the raw `hid_masked` source, and
      `__all__` on `appointments` ships `cancelled_by` — an actor id — to every
      client that can read the row.
- [ ] **GAP** (Adi, non-blocking, **carried**) — no rate limit on
      `GET /clinics?q=`. Authenticated, non-PHI, read-only, so the exposure is
      cost rather than disclosure; tag `# DEMO-MODE` with the real throttle as its named path
      rather than leaving it unremarked.
- [ ] **GAP** (Adi, non-blocking, **new in `012`**) — no rate limit on
      `POST /appointments` either. One authenticated user can book every slot in
      the seeded directory in a loop and empty the demo. Not a disclosure risk;
      it is a **demo-integrity** risk, and the reset command is the mitigation
      of record. **[RESCOPE] Now blocking-adjacent:** booking spam reaches a
      real clinic, so `POST /appointments` needs a per-user rate limit before
      the request/accept flow goes live. Owner: Adi.

## Concurrency — new in `012`

- [x] **PASS** — FR-015 requires the booked check at **write** time, server
      side, returning a conflict, with the client recovering to B4 with fresh
      data. `011` marked the race cosmetic on the reasoning that this is
      single-user demo data; `012` does not inherit that assumption, because
      the meetup script hands the phone around and two people booking the same
      slot is a plausible live failure, not a theoretical one.
- [ ] **GAP** (Adi, non-blocking) — the *mechanism* is unspecified. Take a
      `select_for_update()` row lock or add a unique partial constraint on
      (`slot_id`) where the appointment is live. **A read-then-write is not
      acceptable even in a demo** — it is the exact shape auditors flag, it is
      two lines to do correctly, and the failure is visible on stage.
- [x] **PASS** — reschedule is specified as atomic (old row → `rescheduled`,
      new row created, both linked) in one transaction (§API). A partial
      reschedule would leave a profile with two live appointments or none.

## Secrets & config

- [ ] **GAP** (Adi, non-blocking, **carried — `011` decision 8**) — the maps SDK
      needs an API key in the client. A key in a demo APK is a key that leaks.
      Before any build ships: restrict by package ID
      (`in.thedoorstepclinic.health`) + signing-certificate fingerprint,
      restrict to the maps APIs actually used, set a quota cap, and inject at
      build time — never commit it.
- [x] **PASS** — no other secrets in seed data. Fictional clinics, fictional
      doctors, seeded ratings, seeded fees and a seeded masked HID string are
      invented values, not credentials.
- [ ] **GAP [RESCOPE]** (Adi, blocks the payment tasks) — this previously read
      *"no payment credentials of any kind… nothing is integrated"*. **That
      ended the same day: payments are in scope.** Now required before any
      payment code lands: Razorpay key id/secret **env-var only, never
      committed**; **webhook signature verification** on every callback (an
      unverified payment webhook is a free-order vulnerability); no card data
      touched at all (hosted flow only, keeping us out of PCI scope); and
      `payments` rows scoped from `caregiver_grants` like `appointments`,
      since they are profile-linked even though they are not PHI.
- [x] **N/A** — `DEBUG`/CORS/ALLOWED_HOSTS unchanged from `001`.

## Copy — Principle III

- [x] **PASS** — no cipher names; this feature's copy makes no security claims.
- [x] **PASS** — no banned phrases. FR-028 specifically forbids attaching
      "ABDM certified", HPR, registry or verification copy to the HID chip,
      which is where that banned phrase would most plausibly have crept in.
- [x] **PASS** — **FR-011 is a Principle VI control before it is a UX one.**
      "Booking placed" / "Booked" instead of "confirmed" / "reserved" / "held"
      keeps the app from asserting a clinic acknowledgement that never
      happened. Add these four words to the banned-copy check for this feature.
- [x] **PASS** — FR-016 keeps `completed` and `no_show` unwritten **by this app**; they are clinic-side writes.
      An app that marks an unobserved visit "completed" is fabricating a record
      in a health product; one that marks it `no_show` is fabricating an
      accusation. `lapsed` claims only what is known.
- [ ] **GAP** (Adi, non-blocking, **carried — consequences of `011` decisions 6
      and 7**) — two elements still assert something the build does not do, and
      the mitigations are copy-level, so they need a copy review rather than a
      code test: the **star rating** (a number with no measurement behind it)
      and the **HID chip** (an identifier with no registry behind it). FR-027
      and FR-028 constrain them to the narrowest honest form. Re-read both
      strings before demo; if a reviewer asks "what is 4.8 out of?", the demoer
      needs an answer that is not a bluff.
- [ ] **GAP** (Adi, non-blocking, **new in `012`**) — the **seeded consult fee**
      joins that list. A fictional clinic showing "₹400" is consistent with the
      rest of the seed, but a reviewer may read it as a real price commitment.
      The 🔒 badge is the mitigation; check that it is legible beside the
      number on both B2 and B5, not tucked under it.

## Adversarial pass

Per `CLAUDE.md`: anything touching auth, grants or Fastlane gets a "how would I
abuse this?" pass before merge.

- [x] **PASS** — abuse pass done, findings recorded here. Items 1–5 are carried
      from `011` (still valid); 6–10 are new to `012`'s write surface.

  1. **Directory as a PHI side channel.** Clean today. The abuse is a future
     convenience field ("last visited", "your doctor") turning a public
     endpoint into a leak. Guard: the serializer regression test above.
  2. **Booking against someone else's profile.** Clinic/doctor/slot ids are
     reference ids and a caller could pair any of them with any `profile_id`.
     Scope derivation is server-side (FR-035), so this fails — but it remains
     the single most likely thing to get wrong at implementation time, because
     the request body *looks* like it carries the authorization.
  3. **Slot double-booking / race.** Now tracked as its own section above, and
     no longer written off as cosmetic.
  4. **Directions hand-off.** An address string handed to an external app is an
     intent payload. Keep it a plain address, never a URL assembled from
     server-supplied text, so a seeded field cannot become an open-redirect
     into an arbitrary app.
  5. **Coordinate leak (FR-024).** Not a classic security finding but the same
     class of harm as the Ruby Hall name leak: a fictional clinic pinned on a
     real hospital implies a relationship. Verify each seeded coordinate lands
     on neutral ground before the seed merges. **Now the residual risk that
     matters most**, since the clinic *names* were accepted without a register
     check (`004`, 6 Sep) — a wrong name is fixed by editing one row, a pin on
     a real hospital was already published on a map.
  6. **Cancel as denial-of-care (new).** The genuinely new abuse in `012`. A
     caregiver grant lets one person cancel another's appointment, and the
     appointment silently does not happen. Mitigations in place: scoped
     queryset (FR-035), attributable `cancelled_by` (FR-038), an
     `access_logs` row (FR-036). **Not** in place: any notification to the
     other party. Today the caregiver *is* the account holder for
     dependent profiles, so there is no second party to notify — but the day
     invite-out / request-in from `002` lands and a family member has their own
     account, "your appointment was cancelled by Rohan" becomes required, not
     nice. Recorded here so it is not discovered after that feature ships.
  7. **Reschedule chain abuse (new).** Unbounded rescheduling grows a row chain
     and re-notifies each time. Capped at 2 (FR-018), enforced server-side.
     Check the cap is on the **chain**, not the row — otherwise each new row
     starts at zero and the cap does nothing.
  8. **Notification-id collision (new).** Appointment reminders share
     `006`'s scheduler namespace. A colliding id means moving an appointment
     silently cancels a dose reminder — a medication safety bug reached through
     a booking feature. Namespaced ids required (`006`, shared-primitive note).
  9. **Stale deep link (new).** A reminder can outlive its appointment when
     another device cancels it. `008`'s dead-link rule resolves it to B6 in a
     terminal state. The security-adjacent half: the link must resolve through
     the **scoped** queryset like any other read, so a revoked caregiver
     tapping an old notification gets a 404, not a cached row.
 10. **Seeded-fee misread (new).** Not exploitable, but the demo-integrity
     risk named in the Copy GAP above. 🔒 badge is the mitigation.

- [x] **N/A** — no card/Fastlane surface in this feature.
