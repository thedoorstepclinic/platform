# Contract: TDC Core API (DRF)

Base: `/api/v1/` · Auth: SimpleJWT (Bearer) except where noted · Format: JSON.
Prototype-grade; error bodies are indicative.

## Auth
### `POST /api/v1/auth/otp/request`  *(no auth)*
Request `{ "phone": "+9198..." }` → `200 { "sent": true }`.
Demo mode: any phone accepted; code is always `000000`.

### `POST /api/v1/auth/otp/verify`  *(no auth)*
Request `{ "phone": "+9198...", "code": "000000" }` →
`200 { "access": "<jwt>", "refresh": "<jwt>", "user": { "id", "phone" } }`.
Wrong code → `400 { "error": "invalid_otp" }`.

## Profiles
### `GET /api/v1/profiles/`
→ `200 [ { id, name, relation, dob, alerts: { low_stock, missed_dose } } ]`
(Home cards + alerts strip.)

### `POST /api/v1/profiles/`
Assisted add. Request `{ name, relation, dob }` →
`201 { profile }`. Side effect: writes a `caregiver_grants` row
(`ts = now`) — **the consent moment** (FR-003).

### `GET|PATCH|DELETE /api/v1/profiles/{id}/`
Standard CRUD.

## Records
### `GET /api/v1/profiles/{id}/records/?type=rx|lab|discharge|other`
→ `200 [ { id, type, title, file, record_date, created_at } ]` newest-first.

### `POST /api/v1/profiles/{id}/records/`  *(multipart)*
Fields: `type`, `title`, `record_date`, `file` (image/PDF) → `201 { record }`.
No OCR.

### `GET|DELETE /api/v1/records/{id}/`
Record detail / delete.

## Health Summary
### `GET /api/v1/profiles/{id}/summary.pdf`
→ `200 application/pdf`. Template render from emergency profile + meds + record
list. Not AI-generated. Client shares via WhatsApp.

## Medications
### `GET|POST /api/v1/profiles/{id}/medications/`
POST `{ name, dose, times: ["08:00","20:00"], stock_count, threshold }` →
`201 { medication }`. Client schedules local notifications from `times`.

### `GET|PATCH|DELETE /api/v1/medications/{id}/`
Edit/delete. `days_left = stock_count / len(times)`; low-stock when ≤ 5.

## Emergency profile
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
→ `200 [ { ctr, ts, ip, geo } ]` (Card manager scan log).

## Device tokens (for family blast)
### `POST /api/v1/devices/register`
`{ token }` → `201`. Fastlane blasts these on scan.

## Error conventions
- `401` missing/expired JWT · `403` not a caregiver for the profile ·
  `404` unknown id · `400 { error, detail }` validation.

## Notes
- All `# DEMO-MODE` shortcuts (mock OTP, permissive grants) are flagged in code.
- HMS→timeline (P1) lands records with `source = "hms"` via shared DB or a fake
  webhook `POST /api/v1/hms/webhook` (guarded, P1 only).
