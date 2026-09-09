# Security checklist — [FEATURE NNN-slug]

**Copy to `specs/NNN-slug/checklists/security.md`. Review against `plan.md`
BEFORE implementation, and re-check after.**

Aligned to what a CERT-In empanelled WASA auditor tests: OWASP Top 10 for Web
**and** OWASP API Security Top 10, authentication, authorization, encryption in
transit and at rest, session management, API security. One critical finding
stalls ABDM certification — see `docs/compliance-baseline.md` §3.

Mark each item **PASS**, **N/A** (with a reason), or **GAP** (with an owner).
**Every item blocks implementation** — the `[A]` / `[B→A]` / `[B]` binding
classes were abolished 6 Sep 2026 (constitution v3.0.0), so there is no longer
a non-blocking class of compliance gap. An item may be **N/A** only when this
feature genuinely has no such surface, with the reason stated. "For later" is
not a reason.

*(Repealed 6 Sep 2026: this previously read "a `GAP` on `[B→A]` is allowed
only if the non-foreclosure clause still holds". No gap is allowed on that
basis any more.)*

---

## Authorization — Principle X

- [ ] Every new/modified DRF viewset overrides `get_queryset()` with a
      server-derived scope filter (caller's profiles + unrevoked
      `caregiver_grants`). No `Model.objects.all()` reaches a serializer.
- [ ] `get_object()` resolves from the scoped queryset — no fetch-then-check.
- [ ] No endpoint decides access from a client-supplied id, header, or body
      field. Ownership is derived from the JWT principal only.
- [ ] Revoking a grant denies access on the **next** request (no cached scope,
      no long-lived token embedding the scope).
- [ ] Tested with a second user: user B cannot read, write, or enumerate user
      A's profile, records, meds, card, or appointments — by id, by list, or by
      any filter parameter.
- [ ] IDs are non-sequential where enumeration would leak existence, or
      enumeration is provably harmless for this resource.

## Authentication & session

- [ ] Every endpoint has an explicit permission class. No endpoint relies on a
      project-wide default to be non-public.
- [ ] Endpoints intended to be public are listed here explicitly, with why.
- [ ] Token lifetime and refresh behaviour stated in the plan.
- [ ] Logout / revoke path exists, is described, and is built.
- [ ] Any auth shortcut is behind `DEMO_MODE`, off by default, tagged
      `# DEMO-MODE` with its real path named — and its blocking gate + gate
      owner if that real path is unbuilt (Principle XIV). A shortcut that
      cannot be switched off is a defect, not a shortcut.
- [ ] Rate limiting, lockout, session and token handling are specified and
      built for the surfaces this feature owns. "Auth hardening is Track B"
      is repealed (constitution v3.0.0).

## PHI handling — Principle XIII

- [ ] No PHI in URLs, query strings, or path segments that get logged.
- [ ] No PHI in application logs, error messages, or crash reports.
- [ ] Error responses do not differ in a way that reveals whether a record or
      profile exists.
- [ ] HTTPS enforced on every surface, including Fastlane.
- [ ] Fastlane responder pages are non-indexable and carry no referrer leak.
- [ ] Payload assembly is separable from transport — nothing assumes a bundle
      is readable at handoff (non-foreclosure for Fidelius).

## Audit — Principle XI

- [ ] Every PHI read and write in this feature emits an `access_logs` row:
      actor, subject profile, action, object type + id, timestamp, purpose,
      source service.
- [ ] The log write is in the same transaction as the access; a failed log
      write fails the request.
- [ ] The feature does not add a PHI path that bypasses the logging layer
      (raw SQL, bulk operation, management command, direct file serve).

## Input & API surface

- [ ] All input validated server-side by a serializer — client validation is
      never the only validation.
- [ ] File uploads: type allowlist, size cap, and a stored filename that is
      not attacker-controlled.
- [ ] Uploaded files are not served from a path that permits traversal, and
      are access-checked on read like any other PHI.
- [ ] Mass-assignment closed: serializers use explicit `fields`, never
      `__all__`, on any model with an owner/grant/status column.
- [ ] Rate limits considered for auth, OTP, and any unauthenticated endpoint;
      absence is a documented `# DEMO-MODE` decision, not an oversight.

## Secrets & config

- [ ] No secrets, keys, tokens, or credentials in code, fixtures, seed data,
      or committed config.
- [ ] `DEBUG` off by default outside local; no debug endpoint reachable.
- [ ] CORS/ALLOWED_HOSTS are allowlists, not wildcards — or tagged
      `# DEMO-MODE` with its real path and, if unbuilt, its gate + owner.

## Copy — Principle III

- [ ] No cipher names in UI copy ("AES-256", "military-grade").
- [ ] No banned phrases: "auto consent", "blockchain"/"Hyperledger", "ABDM
      certified", "we fetch everything", anything implying card-tap = consent.
- [ ] Every trust claim in this feature's copy is literally true of the
      built behaviour (Principle VI) — or scoped as roadmap in the copy itself.

## Adversarial pass

Per `CLAUDE.md`: anything touching auth, grants, or Fastlane gets a
"how would I abuse this?" pass before merge.

- [ ] Abuse pass done. Findings and their resolutions recorded in the spec.
- [ ] Card/Fastlane features specifically: replay (`ctr <= last_seen`
      rejected), CMAC verified, revocation honoured immediately, test scans
      distinguishable (`is_test`) and non-counter-consuming.
