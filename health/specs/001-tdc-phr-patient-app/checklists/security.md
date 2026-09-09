# Security checklist — 001-tdc-phr-patient-app

**Reviewed against `plan.md`/`data-model.md`/`contracts/` (2026-08-17), pre-
implementation — no code exists yet (`tasks.md` T001–T042 all unchecked).
"PASS" here means *the design commits to this*, not *it's built and tested*.
Re-run this checklist after Phase E ships, before demo rehearsal (T040).**

Aligned to what a CERT-In empanelled WASA auditor tests: OWASP Top 10 for Web
**and** OWASP API Security Top 10, authentication, authorization, encryption in
transit and at rest, session management, API security. One critical finding
stalls ABDM certification — see `docs/compliance-baseline.md` §3.

Mark each item **PASS**, **N/A** (with a reason), or **GAP** (with an owner).
**Every item blocks implementation** — the `[A]` / `[B→A]` / `[B]` binding
classes were abolished 6 Sep 2026 (constitution v3.0.0). An item may be **N/A**
only where this feature genuinely has no such surface, with the reason stated.

**Summary:** 2 blocking GAPs (Authorization #5, Audit #3), both expected
at this pre-code stage but requiring a concrete answer before the phases they
gate. 12 non-blocking GAPs tracked below, mostly standard DRF/deploy
discipline that isn't yet written into `plan.md` as an explicit requirement.
One item (PHI handling — error-response existence leak) was found **and
fixed** during this review: `contracts/core-api.md`'s 403-vs-404 split leaked
out-of-scope resource existence; collapsed to 404-only.

---

## Authorization — Principle X

- [x] **PASS** — Every new/modified DRF viewset overrides `get_queryset()` with a
      server-derived scope filter (caller's profiles + unrevoked
      `caregiver_grants`). No `Model.objects.all()` reaches a serializer.
      *Committed in `contracts/core-api.md` §Authorization Scoping, designed
      in `research.md` R9, built by `tasks.md` T005.*
- [x] **PASS** — `get_object()` resolves from the scoped queryset — no fetch-then-check.
      *Same source as above.*
- [x] **PASS** — No endpoint decides access from a client-supplied id, header, or body
      field. Ownership is derived from the JWT principal only.
      *Implied by the `get_queryset()` design deriving scope from "the
      authenticated caller," not from any request parameter.*
- [x] **PASS** — Revoking a grant denies access on the **next** request (no cached scope,
      no long-lived token embedding the scope).
      *`contracts/core-api.md`: "A revoked grant stops access on the next
      request; no cached scope."*
- [ ] **GAP** (owner: Adi,) — Tested with a second user: user B cannot read, write, or
      enumerate user A's profile, records, meds, card, or appointments — by
      id, by list, or by any filter parameter.
      *Cannot be true yet — no code exists. Expected GAP until T005/T009
      ship; close by writing this as an explicit negative-auth test run
      before Phase B starts, not discovered live on stage.*
- [x] **PASS** — IDs are non-sequential where enumeration would leak existence, or
      enumeration is provably harmless for this resource.
      *All entities in `data-model.md` use `uuid PK`; `cards.uid` is the
      NTAG chip's factory UID, not an app-assigned sequential id.*

## Authentication & session

- [ ] **GAP** (owner: Adi) — Every endpoint has an explicit permission class. No endpoint relies on a
      project-wide default to be non-public.
      *Not yet stated as a rule anywhere; add to Core API scaffold (T002) as
      an explicit requirement — every DRF viewset sets `permission_classes`
      explicitly, never relies on `DEFAULT_PERMISSION_CLASSES` alone.*
- [x] **PASS** — Endpoints intended to be public are listed here explicitly, with why.
      *`contracts/core-api.md` marks `/auth/otp/request|verify` "(no auth)";
      `contracts/fastlane-api.md` marks `GET /e/{uid}` "(no auth, public)"
      with the crown-jewel rationale (must work for a logged-out responder).*
- [ ] **GAP** (owner: Adi) — Token lifetime and refresh behaviour stated in the plan.
      *SimpleJWT is the named mechanism but no lifetime/refresh policy is
      documented anywhere. Cheap to add to `plan.md` Technical Context.*
- [ ] **GAP** (owner: Adi) — Logout / revoke path exists, is described, and is built.
      *Not described. Minimal fix: client discards JWT locally, no
      server-side blacklist yet — tag `# DEMO-MODE`, and the real path adds a
      token blacklist. **Now blocking:** auth hardening is in scope
      (constitution v3.0.0), so this cannot ship as a permanent shortcut. Cheap to close before T008.*
- [x] **PASS** — Any auth shortcut sits behind `DEMO_MODE` (off by default) and is
      tagged `# DEMO-MODE` naming its real path (Principle XIV).
      *`contracts/core-api.md` Notes + `tasks.md` T008 both commit to this
      for mock OTP.*

## PHI handling — Principle XIII

- [x] **PASS** — No PHI in URLs, query strings, or path segments that get logged.
      *`contracts/fastlane-api.md`: only `ctr`/`cmac` in the query string.
      Core API path segments are ids, not PHI content.*
- [ ] **GAP** (owner: Adi) — No PHI in application logs, error messages, or crash reports.
      *Fastlane's side is explicit (`research.md` R11: logs record uid/ctr/
      ip/ts, never blood group/allergies/etc). Core API has no equivalent
      written rule — add to `plan.md` or a Django logging-config task:
      never log `serializer.data` or a model's PHI fields on error.*
- [x] **PASS** (fixed 2026-08-17) — Error responses do not differ in a way that reveals whether a record or
      profile exists.
      *Fastlane is explicit (identical neutral-page body, HTTP 200, no
      existence confirmation). Core API's Error conventions previously
      distinguished `403` ("not a caregiver") from `404` ("unknown id"),
      leaking existence for out-of-scope ids — collapsed to 404-only in
      `contracts/core-api.md`, consistent with strict `get_queryset()`
      scoping (Principle X).*
- [ ] **GAP** (owner: Adi) — HTTPS enforced on every surface, including Fastlane.
      *Fastlane is explicit (HTTPS-only, `contracts/fastlane-api.md`). Core
      API has no equivalent explicit statement — add `SECURE_SSL_REDIRECT`
      (or reverse-proxy enforcement) to T002's scaffold requirements.*
- [x] **PASS** — Fastlane responder pages are non-indexable and carry no referrer leak.
      *`contracts/fastlane-api.md`: `X-Robots-Tag: noindex` on every
      response, PHI only in the HTML body.*
- [x] **PASS** — Payload assembly is separable from transport — nothing assumes a bundle
      is readable at handoff (non-foreclosure for Fidelius).
      *`emergency_payload` (`data-model.md`) is plain structured data with no
      transport assumption baked in.*

## Audit — Principle XI

- [x] **PASS** — Every PHI read and write in this feature emits an `access_logs` row:
      actor, subject profile, action, object type + id, timestamp, purpose,
      source service.
      *`data-model.md` §access_logs, `contracts/core-api.md` §Access Logging,
      `contracts/fastlane-api.md` step 5, built by `tasks.md` T006/T025.*
- [x] **PASS** — The log write is in the same transaction as the access; a failed log
      write fails the request.
      *`data-model.md`: "Written in the same transaction... a failed log
      write fails the request." T006 builds this as a shared writer.*
- [ ] **GAP** (owner: Adi,) — The feature does not add a PHI path that bypasses the logging layer
      (raw SQL, bulk operation, management command, direct file serve).
      *Unresolved: record files and the summary PDF (`GET
      /profiles/{id}/records/`'s `file` field, `GET
      /profiles/{id}/summary.pdf`) are the actual PHI payload, not just
      metadata. `contracts/core-api.md` doesn't yet say whether reading a
      record's file goes through the same `access_logs`-writing view or a
      separate signed-URL/static-serve path that the T006 writer never
      touches. Needs an explicit answer before T011/T015 (Phase B/C) ship —
      simplest fix: file reads are proxied through a logged DRF view, never
      served as a bare storage URL.*

## Input & API surface

- [x] **PASS** — All input validated server-side by a serializer — client validation is
      never the only validation.
      *DRF is the stated framework (`plan.md` Technical Context); every
      documented POST/PUT body in `contracts/core-api.md` follows DRF's
      serializer idiom.*
- [ ] **GAP** (owner: Adi) — File uploads: type allowlist, size cap, and a stored filename that is
      not attacker-controlled.
      *`contracts/core-api.md`'s record-upload endpoint names `file`
      (image/PDF) but no allowlist, size cap, or filename policy is
      specified. Add before T011.*
- [ ] **GAP** (owner: Adi) — Uploaded files are not served from a path that permits traversal, and
      are access-checked on read like any other PHI.
      *Same root cause as the Audit-section file-serving GAP above — close
      both together.*
- [ ] **GAP** (owner: Adi) — Mass-assignment closed: serializers use explicit `fields`, never
      `__all__`, on any model with an owner/grant/status column.
      *Standard DRF discipline, not yet written down anywhere. Enforce at
      code-review time for every serializer touching `profiles`,
      `caregiver_grants`, `cards`, `medications`.*
- [x] **PASS** — Rate limits considered for auth, OTP, and any unauthenticated endpoint;
      absence is a documented `# DEMO-MODE` decision, not an oversight.
      *T008 + Principle XIV already require the mock-OTP shortcut to carry a
      `# DEMO-MODE` tag naming what's unsafe (no rate limit) and its real path — the requirement to document is committed even though
      the literal tag text doesn't exist until T008 is coded.*

## Secrets & config

- [ ] **GAP** (owner: Adi) — No secrets, keys, tokens, or credentials in code, fixtures, seed data,
      or committed config.
      *Partially addressed: `data-model.md`'s `cards.sdm_key_ref` is
      explicitly "a reference to the per-card CMAC key (not the key
      itself)" — good. But DB credentials, the JWT signing key, and Firebase
      service-account credentials have no explicit "env-var only, never
      committed" statement. Add to T002/T003 scaffold requirements.*
- [ ] **GAP** (owner: Adi) — `DEBUG` off by default outside local; no debug endpoint reachable.
      *Not stated. Add to T002.*
- [ ] **GAP** (owner: Adi) — CORS/ALLOWED_HOSTS are allowlists, not wildcards — or tagged
      `# DEMO-MODE` naming the real path.
      *Not stated. Principle XIV names "any permissive CORS or debug
      setting" as in-scope for DEMO-MODE tagging — nothing currently
      guarantees this gets tagged if left permissive. Add to T002.*

## Copy — Principle III

- [x] **PASS** — No cipher names in UI copy ("AES-256", "military-grade").
      *Copy Guardrails (`spec.md`) + T032 lint task, covering both codebases.*
- [x] **PASS** — No banned phrases: "auto consent", "blockchain"/"Hyperledger", "ABDM
      certified", "we fetch everything", anything implying card-tap = consent.
      *Same source — FR-018, T032.*
- [x] **PASS** — Every trust claim in this feature's copy is literally true of the
      built behaviour (Principle VI) — or scoped as roadmap in the copy itself.
      *Contingent on T006/T025 (access logging) and T031 (Trust screen)
      shipping together — "every access is logged" is false until then.
      Track in Phase F, don't let T031 ship ahead of T006/T025.*

## Adversarial pass

Per `CLAUDE.md`: anything touching auth, grants, or Fastlane gets a
"how would I abuse this?" pass before merge.

- [ ] **GAP** (owner: Adi) — Abuse pass done. Findings and their resolutions recorded in the spec.
      *Not yet done — schedule once auth/grants/Fastlane code exists
      (post-Phase E), findings recorded back into this checklist or
      `spec.md`.*
- [x] **PASS** (design; behavioral test still pending code) — Card/Fastlane
      features specifically: replay (`ctr <= last_seen` rejected), CMAC
      verified, revocation honoured immediately, test scans distinguishable
      (`is_test`) and non-counter-consuming.
      *Replay/CMAC/revocation are designed (FR-012/013,
      `contracts/fastlane-api.md`). `is_test` was missing from `001`'s own
      `data-model.md`'s `scan_events` table (only in `CLAUDE.md`'s global
      data model and `007`'s spec) — **fixed 2026-08-17**, same session:
      added to `data-model.md`, `contracts/core-api.md`'s scan-log response,
      and `tasks.md` T025. All four behaviors are now designed consistently;
      what remains is running the actual replay/CMAC/revocation/test-scan
      abuse pass once Phase E code exists (tracked in the item above).*
