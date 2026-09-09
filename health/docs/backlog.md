# Backlog — real requirements without a spec yet

> **Renamed and reframed 6 Sep 2026** (was `track-b-backlog.md`, "known gaps,
> deliberately not built"). The track split is abolished (constitution
> v3.0.0), so **nothing here is "deferred" any more** — these are simply
> features that do not yet own a `specs/NNN-slug/` directory. Under workflow
> rule 7 every feature owns a complete spec-kit set, which makes this file a
> **staging area, not a parking lot**: an entry leaves by becoming a spec, not
> by being reclassified.

These are **real product requirements** identified while specifying other
features. They are recorded here so they are not rediscovered as surprises,
and so decisions taken elsewhere do not make them expensive to build.

Each entry states: what it is, why it has no spec yet, and what must not be
foreclosed in the meantime. The "why" is now a **status**, not a
justification — "not specced yet" is an honest answer; "that's Track B" is
not, because there is no Track B.

**Every item on this list is in scope.** Several are load-bearing for
production in a way they never were for a demo — **§1 account recovery most of
all**, since phone *is* the account and a lost SIM would mean a permanently
lost family health record. That was invisible while auth was mock OTP; it is
now arguably the highest-severity unspecced item in the project.

## 1. Account recovery

**What:** A way back into an account when the registered phone number is gone.

**Why no spec yet:** while auth was mock OTP (`000000`) there was nothing to
recover from. In production, phone *is* the account, and Indian users churn
SIMs constantly — a lost number would mean a permanently lost family health
record. This is the highest-severity gap on the list even though it's invisible
in the demo.

**Don't preclude:** Keep `users` separable from the phone identifier. Never let
a phone number become a primary key or the sole join path to `profiles`.

## 2. Data export

**What:** Patient-initiated export of all records, meds, and history — files
included, not just metadata.

**Why no spec yet:** The demo's export surface is the Health Summary PDF (`004`,
`009`), which is a clinical hand-off artifact, not a portability guarantee.

**Don't preclude:** It's cheap to keep this possible — records already carry a
file reference and a typed schema. Just don't build anything that stores record
content only inside a rendered artifact.

**Note:** This is a DPDP obligation in production, not a nice-to-have.

## 3. App lock (PIN / biometric)

**What:** Local re-auth to open the app.

**Why no spec yet:** Demo phones get handed around; a lock screen actively hurts
the demo. Also explicitly excluded by Principle IV ("no real auth hardening").

**Don't preclude:** nothing structural — self-contained whenever it is specced.

**Note:** For a family PHR the threat model is mostly *household*, not remote —
the person you least want reading a parent's records is often in the same room.
That makes it more important than its current lack of a spec suggests.

## 4. App-wide Marathi

**What:** Full i18n across every screen.

**Why no spec yet:** the existing item is narrower on purpose — a Marathi toggle
on the *responder page* only (`CLAUDE.md` P1 list), where a bystander or
first-responder in Pune is the reader and the stakes are highest. Translating
twelve screens is not a demo differentiator.

**Don't preclude:** Don't hardcode user-facing strings inline. Even without a
framework, keep copy in one module per screen so extraction is mechanical
later. Copy is already constrained by Principle III — an approved-phrase list
is easier to translate consistently than scattered literals.

## 5. Home vitals logging (BP, glucose, weight)

**What:** Patient- or caregiver-entered readings, trended over time.

**Why no spec yet:** Not in the golden path, and it needs a new table plus a chart
surface. Cleanly separable, which is why it was left out this long.

**Assessment: this is the highest-value item on the list.** Records and meds
are retention through *obligation*; vitals are retention through *habit* —
a daily reason to open the app that Home's quiet (Principle VIII) permits,
because a logged reading is patient-initiated, not attention-harvested. For
Asha's diabetes arc specifically, a glucose trend is the natural companion to
the HbA1c record series the demo already tells a story with.

**Don't preclude:** Nothing blocks it; it's an additive table. Worth a spec
early rather than late.

## 6. Help & Support

**What:** A visible route to a human.

**Why no spec yet:** No support org exists yet.

**Decision when built: a WhatsApp deep link, not an in-app chat.** Building
custom chat means a message store, presence, notifications, and a staffed
inbox. WhatsApp is where this user already is, costs one `wa.me` link, and
transcripts live somewhere a small team actually reads. Boring and
already-paid-for.

**Don't preclude:** Reserve a Settings row for it (`009` has the Settings
screen). No data model impact.
