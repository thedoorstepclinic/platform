# Contract: TDC Core API (DRF)

Base: `/api/v1/` · Auth: SimpleJWT (Bearer) except where noted · Format: JSON.
Prototype-grade; error bodies are indicative.

## Authorization Scoping (Principle X)
Every endpoint below that touches `profiles`, `records`, `medications`,
`emergency_profiles`, `cards`, or `scan_events` MUST derive its queryset from
the authenticated caller: **own profiles ∪ profiles reachable through an
unrevoked `caregiver_grants` row.** A viewset whose `get_queryset()` returns
an unfiltered `Model.objects.all()` is a build failure, not a review comment.
`get_object()` always resolves out of that scoped queryset — never
`Model.objects.get(pk=...)` followed by a permission check. A revoked grant
stops access on the very next request; no cached scope. FR-022, `research.md` R9.

## Access Logging (Principle XI)
Every endpoint below marked **PHI** writes one `access_logs` row (actor,
subject profile, action, object type + id, purpose, `source_service =
"core"`) in the same transaction as the request. A failed log write fails
the request. FR-021, `data-model.md` §access_logs, `research.md` R10.

## Auth
### `POST /api/v1/auth/otp/request`  *(no auth)*
Request `{ "phone": "+9198..." }` → `200 { "sent": true }`.
Demo mode: any phone accepted; code is always `000000`.

### `POST /api/v1/auth/otp/verify`  *(no auth)*
Request `{ "phone": "+9198...", "code": "000000" }` →
`200 { "access": "<jwt>", "refresh": "<jwt>", "user": { "id", "phone" } }`.
Wrong code → `400 { "error": "invalid_otp" }`.

## Profiles **[PHI]**
### `GET /api/v1/profiles/`
→ `200 [ { id, name, relation, dob, alerts: { low_stock, missed_dose } } ]`
(Home cards + alerts strip.)

### `POST /api/v1/profiles/`
Assisted add. Request `{ name, relation, dob }` →
`201 { profile }`. Side effect: writes a `caregiver_grants` row
(`ts = now`) — **the consent moment** (FR-003).

### `GET|PATCH|DELETE /api/v1/profiles/{id}/`
Standard CRUD.

## Records **[PHI]**
### `GET /api/v1/profiles/{id}/records/?type=rx|lab|discharge|other`
→ `200 [ { id, type, title, file, record_date, created_at } ]` newest-first.

### `POST /api/v1/profiles/{id}/records/`  *(multipart)*
Fields: `type`, `title`, `record_date`, `file` (image/PDF) → `201 { record }`.
No OCR.

### `GET|DELETE /api/v1/records/{id}/`
Record detail / delete.

## Health Summary **[PHI]**
### `GET /api/v1/profiles/{id}/summary.pdf`
→ `200 application/pdf`. Template render from emergency profile + meds + record
list. Not AI-generated. Client shares via WhatsApp.

## Medications **[PHI]**
### `GET|POST /api/v1/profiles/{id}/medications/`
POST `{ name, dose, times: ["08:00","20:00"], stock_count, threshold }` →
`201 { medication }`. Client schedules local notifications from `times`.

### `GET|PATCH|DELETE /api/v1/medications/{id}/`
Edit/delete. `days_left = stock_count / len(times)`; low-stock when ≤ 5.

## Emergency profile **[PHI]**
### `GET|PUT /api/v1/profiles/{id}/emergency/`
Body `{ blood_group, allergies, conditions, abha_no,
contacts: [{ name, phone, relation, priority }] }`.
**Side effect (FR-010):** rebuilds `emergency_payload` for each active card
bound to the profile.

## Caregiver grants
### `GET /api/v1/profiles/{id}/grants/`
→ `200 [ { grantee, ts, revoked_ts } ]` (Family & consent screen).

### `POST /api/v1/grants/{id}/revoke`
Sets `revoked_ts = now` (one toggle) → `200 { grant }`.

## Cards
### `POST /api/v1/cards/link`
`{ uid, profile_id, sdm_key_ref }` → `201 { card }` (active=false until confirmed).

### `POST /api/v1/cards/{uid}/activate` / `POST /api/v1/cards/{uid}/revoke`
Toggle `active`. Revoke ⇒ Fastlane serves neutral page (kill-switch, FR-011).

### `GET /api/v1/cards/{uid}/scans`
→ `200 [ { ctr, ts, ip, geo, is_test } ]` (Card manager scan log; test scans
carry `is_test=true` and a "Test" chip per `007`).

## Device tokens (for family blast)
### `POST /api/v1/devices/register`
`{ token }` → `201`. Fastlane blasts these on scan.

## Error conventions
- `401` missing/expired JWT · `404` unknown id **or** the caller isn't a
  scoped caregiver for it — identical response either way, no existence
  confirmation for a resource outside the caller's `get_queryset()` scope
  (Principle X; corrected 2026-08-17 — a prior draft distinguished `403` for
  "not a caregiver" from `404` for "unknown id," which leaked whether an
  out-of-scope id existed) · `400 { error, detail }` validation.

## Notes
- All `# DEMO-MODE` shortcuts (mock OTP, permissive grants) are flagged in
  code with what it substitutes and its real path on the same/next line
  (Principle XIV format) — see `research.md` R7.
- HMS→timeline (P1) lands records with `source = "hms"` via shared DB or a fake
  webhook `POST /api/v1/hms/webhook` (guarded, P1 only).
