# Track B backlog — known gaps, deliberately not built

These are gaps identified during Track A specification that are **real product
requirements** but correctly out of scope for a seeded investor/meetup demo
(Principle IV). They are parked here so they are not rediscovered as surprises
post-funding, and so Track A avoids decisions that would make them expensive.

Not a roadmap — no ordering, no estimates. Each entry: what, why it's deferred,
and what Track A must not preclude.

## 1. Account recovery

**What:** A way back into an account when the registered phone number is gone.

**Why deferred:** Track A auth is mock OTP (`000000`); there is nothing to
recover from. But in production, phone *is* the account, and Indian users churn
SIMs constantly — a lost number would mean a permanently lost family health
record. This is the highest-severity gap on the list even though it's invisible
in the demo.

**Don't preclude:** Keep `users` separable from the phone identifier. Never let
a phone number become a primary key or the sole join path to `profiles`.

## 2. Data export

**What:** Patient-initiated export of all records, meds, and history — files
included, not just metadata.

**Why deferred:** The demo's export surface is the Health Summary PDF (`004`,
`009`), which is a clinical hand-off artifact, not a portability guarantee.

**Don't preclude:** It's cheap to keep this possible — records already carry a
file reference and a typed schema. Just don't build anything that stores record
content only inside a rendered artifact.

**Note:** This is a DPDP obligation in production, not a nice-to-have.

## 3. App lock (PIN / biometric)

**What:** Local re-auth to open the app.

**Why deferred:** Demo phones get handed around; a lock screen actively hurts
the demo. Also explicitly excluded by Principle IV ("no real auth hardening").

**Don't preclude:** Nothing structural. Track B addition, self-contained.

**Note:** For a family PHR the threat model is mostly *household*, not remote —
the person you least want reading a parent's records is often in the same room.
That makes this more important than its Track A priority suggests.

## 4. App-wide Marathi

**What:** Full i18n across every screen.

**Why deferred:** Track A's P1 item is narrower on purpose — a Marathi toggle
on the *responder page* only (`CLAUDE.md` P1 list), where a bystander or
first-responder in Pune is the reader and the stakes are highest. Translating
twelve screens is not a demo differentiator.

**Don't preclude:** Don't hardcode user-facing strings inline. Even without a
framework, keep copy in one module per screen so extraction is mechanical
later. Copy is already constrained by Principle III — an approved-phrase list
is easier to translate consistently than scattered literals.

## 5. Home vitals logging (BP, glucose, weight)

**What:** Patient- or caregiver-entered readings, trended over time.

**Why deferred:** Not in the golden path, and it needs a new table plus a chart
surface. Cut cleanly for Track A.

**Assessment: this is the highest-value item on the list.** Records and meds
are retention through *obligation*; vitals are retention through *habit* —
a daily reason to open the app that Home's quiet (Principle VIII) permits,
because a logged reading is patient-initiated, not attention-harvested. For
Asha's diabetes arc specifically, a glucose trend is the natural companion to
the HbA1c record series the demo already tells a story with.

**Don't preclude:** Nothing blocks it; it's an additive table. Worth a spec
early in Track B rather than late.

## 6. Help & Support

**What:** A visible route to a human.

**Why deferred:** No support org exists yet.

**Decision when built: a WhatsApp deep link, not an in-app chat.** Building
custom chat means a message store, presence, notifications, and a staffed
inbox. WhatsApp is where this user already is, costs one `wa.me` link, and
transcripts live somewhere a small team actually reads. Boring and
already-paid-for.

**Don't preclude:** Reserve a Settings row for it (`009` has the Settings
screen). No data model impact.
