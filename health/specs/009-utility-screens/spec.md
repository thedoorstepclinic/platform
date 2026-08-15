# Feature Specification: Utility Screens — Login, Health Summary, Trust, Settings (Track A)

**Feature Branch:** `009-utility-screens`
**Created:** 2026-07-16
**Status:** Draft
**Owner:** Adi (dev) / Soham (Trust copy sign-off)
**Depends on:** [`001`](../001-tdc-phr-patient-app/spec.md) (screens 1, 6,
12) · [`004`](../004-seed-data-and-summary/spec.md) (summary field map) ·
[`008`](../008-navigation-app-shell/spec.md) (routes, states, tokens).

## What this is / is NOT

**IS:** The remaining screens no feature spec owns: Splash/Login (S1),
Health Summary (S6), Trust (S12), and the minimal Settings screen `008`
introduced. These are "utility" in navigation terms only — Login is the
first impression and Trust is the positioning, so both get full care.

**IS NOT:** Real OTP infrastructure (Track B), the summary PDF's field
mapping (owned by `004`), or the onboarding router (`002`, Track B).

---

## Screen 1 — Splash / Login

Purpose: get a returning user in instantly and a demo in reliably. Two
steps, one decision each (`002`'s one-decision-per-screen rule applies
even in Track A).

**Flow:**
1. **Splash** — logo + wordmark on `bg`, auto-advances (<1s). If a valid
   session exists → straight to Home. No splash-screen taglines.
2. **Phone entry** — one field (+91 prefix fixed for Track A), one primary
   button ("Get OTP"). Below, quiet reassurance line from approved copy:
   *"Your records are encrypted and stored only in India."*
3. **OTP entry** — 6-digit code boxes, auto-submit on 6th digit, resend
   after 30s countdown. Demo mode: `000000` accepted for any number
   (`001` FR-001, tagged `# DEMO-MODE`); thin dismissible "Demo" tag shows
   here and only here (`008`).

**States:** wrong OTP → inline *"That code didn't match. Try again."*
(field shakes once, code clears — never a modal). Network failure →
standard error state with retry (`008`). No account-lockout logic in
Track A (no real auth hardening, per scope).

**After auth:** returning user → Home. First-ever login → own profile
auto-created, then Home with the first-session featured action (`001`
§4.1; full router deferred to `002`/Track B).

## Screen 6 — Health Summary

Purpose: the "show this to any doctor in 60 seconds" one-pager, on screen.

- **Content = `004`'s PDF field mapping, order and all** (header →
  allergies → conditions → current meds → last-5 records → contacts →
  footer). The screen and the PDF are the same information — the screen
  is a preview of exactly what will be shared, never a richer or poorer
  variant. Same single-source rule as `007`'s responder preview.
- **Share as PDF** is the one primary button (56dp, thumb zone), rendering
  via `001` `GET /profiles/{id}/summary.pdf` → OS share sheet (WhatsApp
  surfaces naturally; no WhatsApp-specific integration).
- Meds section derives from active `medications` (`006`); records section
  is the profile's last 5 by date. Nothing hand-entered on this screen —
  it has no edit mode. Gaps prompt via their owning screens (e.g. empty
  allergies row shows "Add in Emergency profile →").
- Loading: skeleton of the section layout; generation failure: standard
  error + retry (`008`).

## Screen 12 — Trust

Purpose: the positioning, in the app. Static content, approved copy only —
this screen is where drift would be most expensive, so its full copy is
locked here (Soham may polish wording within the approved list; structure
is fixed):

1. **"Your family's records, in your control"** — you decide who sees
   what; caregiver access is granted once, logged, and revocable any time
   in Family & consent. *(links → Family & consent)*
2. **"Every access is logged"** — record views, card scans, all of it;
   card scans also notify the family instantly. *(links → Card manager
   scan log)*
3. **"Stored only in India, encrypted"** — full stop. No cipher names, no
   certifications claimed.
4. **"Built on India's ABDM framework (integration in development)"** —
   the one forward-looking line, exactly this phrasing.
5. Footer: The Doorstep Clinic (TDC) · support contact · privacy policy
   link.

Layout: plain scrolling page, `heading` + `body` tokens, no icon
theatrics, no lock imagery beyond a single quiet glyph per section.
Reached from Settings and linked from consent surfaces (`008`).

**Banned here above all** (`CLAUDE.md`): cipher names, "military-grade",
"ABDM certified", any tap-equals-consent implication, "auto consent".

## Settings (utility, unnumbered)

Track A minimum, resisting growth (`008` open decision, now resolved):
account phone number (display only) · **Trust & privacy** (→ S12) ·
**Log out** (one confirm) · app version + build tag. Nothing else — no
preferences, no theme toggle, no notification settings in Track A
(reminders configure per-med in `006`).

---

## Requirements

### Functional Requirements
- **FR-001**: Splash MUST auto-advance in under 1s; a valid session skips
  login entirely.
- **FR-002**: Login MUST be exactly two steps (phone → OTP), with demo
  OTP `000000` accepted in demo mode and the Demo tag confined to this
  screen.
- **FR-003**: OTP errors MUST resolve inline (no modals); resend gated by
  a 30s countdown.
- **FR-004**: First-ever auth MUST auto-create the user's own profile
  before landing on Home.
- **FR-005**: The Health Summary screen MUST render the same fields in
  the same order as the `004` PDF mapping, derived entirely from
  structured data — no on-screen editing, no divergence from the PDF.
- **FR-006**: Share-as-PDF MUST be the screen's single primary action,
  producing the `004`-mapped PDF via the OS share sheet.
- **FR-007**: The Trust screen MUST contain only the five sections above,
  using approved copy, with working links to Family & consent and the
  scan log.
- **FR-008**: Settings MUST contain exactly: phone display, Trust link,
  logout (with confirm), version — nothing more in Track A.

### Non-Functional Requirements
- **NFR-001:** Login flow completable one-handed; OTP boxes ≥48dp.
- **NFR-002:** Summary screen renders from local/API data in one request;
  PDF generation may take longer but shows progress on the button itself
  (spinner-in-button, screen stays interactive).

---

## Open Decisions
| Decision | Owner | Notes |
|---|---|---|
| Trust-screen wording polish within approved list | Soham | Structure and claims locked above; words may be warmed. |
| Support contact on Trust footer (email vs phone) | Soham | Whatever is actually staffed — an unanswered channel is worse than none. |

---

## Review & Acceptance Checklist
- [x] All remaining `001` screens (1, 6, 12) + Settings covered; every
      screen in the app now has an owning spec.
- [x] Summary screen locked to `004`'s mapping (single-source, like `007`'s preview).
- [x] Trust copy structure locked with approved-list-only language.
- [x] Requirements testable (two-step login, inline OTP errors, exact
      Settings contents, five Trust sections).

## Execution Status
- [x] Flows defined
- [x] Requirements generated
- [x] Copy structure locked (polish delegated within approved list)
- [x] Review checklist passed
