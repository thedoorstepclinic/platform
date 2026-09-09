# Contract: Emergency Fastlane (FastAPI)

Separate service · subdomain `e.thedoorstepclinic.com` · independent uptime
budget · reads only the `emergency_payload` snapshot (never live Core joins).

## Public-page hardening (Principle XIII, public-page clause)
HTTPS-only. PHI appears only in the rendered HTML body — never in the query
string (`ctr`/`cmac` are the only params), never in access/error logs, never
in a `Referer`-leaking link. Every response (success and neutral page alike)
sends `X-Robots-Tag: noindex`. See `research.md` R11.

## `GET /e/{uid}`  *(no auth, public)*

### Query params (SDM-mirrored by NTAG 424 DNA)
| Param | Meaning |
|-------|---------|
| `ctr` | monotonic scan counter from the chip |
| `cmac` | CMAC computed by the chip over the mirrored data |

### Processing
1. Look up `cards[uid]`. If missing or `active = false` → **neutral page**
   (see below). No data, no confirmation the card exists (FR-015).
2. Verify `cmac` using the per-card key (`sdm_key_ref`). Invalid → neutral page.
3. Reject replay: if `ctr ≤ last_ctr` → neutral page. Else set `last_ctr = ctr`
   (FR-013).
4. Load `emergency_payload[uid]` (snapshot only).
5. Log `scan_event(uid, ctr, ts, ip, coarse_geo?)` (card mechanics) **and**,
   in the same transaction, write one `access_logs` row (`actor=null`,
   subject profile = the card's bound profile, `action=read`,
   `object_type=emergency_payload`, `source_service=fastlane`) — Principle
   XI. Neutral-page responses (steps 1–3) write neither: no PHI was accessed.
6. FCM blast to all family device tokens (coarse geo if present, omitted
   gracefully if not) (FR-014).
7. Render server-side HTML, **zero JS required to read**, target **<2s on 4G**,
   with `X-Robots-Tag: noindex` (Principle XIII).

### Success response — responder HTML render order
1. **BLOOD GROUP** — largest element on the page.
2. **ALLERGIES** — red / high-contrast.
3. Conditions.
4. Current meds (name + dose).
5. Emergency contacts — **tap-to-call** (`tel:` links), ordered by priority.
6. ABHA number — captioned "for hospital records access".
7. Banner: **"Access logged & family notified."**
8. EN/MR toggle *(P1)*.

> Consent framing (Principle II): the page reflects a **standing grant**. Copy
> never states or implies the scan itself is consent.

### Neutral page (invalid / revoked / replay / bad CMAC)
Fixed copy, no data:
> **"This card is inactive. If this is an emergency, call 108."**

HTTP 200 (do not leak validity via status codes); identical body for all
invalid reasons (no existence confirmation).

## Snapshot dependency
Fastlane's only write-time dependency is the `emergency_payload` table populated
by Core API on profile/emergency/med/contact save (FR-010). If Core API is down,
`GET /e/{uid}` still renders from the last snapshot (Principle I / NFR-002).

## Non-functional
- **NFR-001:** <2s on 4G (server-rendered, snapshot-backed, no JS to read).
- **NFR-002:** survives Core API downtime.
- **NFR-003:** CMAC + monotonic counter replay protection (prototype-grade).
