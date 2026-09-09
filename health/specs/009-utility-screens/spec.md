# Feature Specification: Utility Screens — Health Summary, Trust & Settings

**Feature Branch:** `009-utility-screens`
**Created:** 2026-07-16
**Status:** Draft
**Owner:** Adi (dev) / Soham (Trust copy sign-off)
**Depends on:** [`001`](../001-tdc-phr-patient-app/spec.md) (screens 6,
12) · [`004`](../004-seed-data-and-summary/spec.md) (summary field map) ·
[`008`](../008-navigation-app-shell/spec.md) (routes, states, tokens).
**Cedes to:** [`002-onboarding-router-activation`](../002-onboarding-router-activation/spec.md)
— **Splash/Login (S1) moved to `002`** in its 18 Aug 2026 rewrite. `002`'s
downstream-edits table has asked for this since; applied 6 Sep 2026.

## What this is / is NOT

**IS:** The remaining screens no feature spec owns: Health Summary (S6),
Trust (S12), and the minimal Settings screen `008` introduced. "Utility" in
navigation terms only — Trust is the positioning and gets full care.

**IS NOT:** **Splash/Login — ceded to `002`** (see below), the summary PDF's
field mapping (owned by `004`), or onboarding (`002`).

---

## Screen 1 — Splash / Login — **CEDED to `002`**

> **Removed 6 Sep 2026.** `002`'s 18 Aug rewrite absorbed Splash and Login
> (its S0 and S1) and its downstream-edits table has listed this cession as
> required ever since. Keeping a second description here was a live
> contradiction: this spec described a `+91`-fixed field, `000000` accepted
> for any number, and *"no account-lockout logic (no real auth hardening, per
> scope)"* — all three of which `002` and constitution v3.0.0 now override.
>
> **The authority is [`002` §S0/S1](../002-onboarding-router-activation/spec.md).**
> There: splash under 1s with the session check behind it · a 3–4 frame brand
> carousel with the entry field in the first frame · Google sign-in on the
> same surface · **6-digit OTP with an enforced resend cooldown and attempt
> cap**, whose failure copy must not reveal whether a number is registered.
>
> `009`'s former FR-001 and FR-002 are deleted with the screen; FR-003 and
> FR-004 move to `002` as well. FR-005 onward (Health Summary, Trust,
> Settings) are unaffected and keep their numbers.

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

### Summary header and family cards (added 9 Sep 2026)

From a supplied "Medical Summary" reference. Two additions:

**Action header.** `My Summary` alongside **PDF** and **Share** as peer
actions. `004`'s field mapping and Principle IX are untouched — both actions
render the same mapping the screen shows; Share is `share_plus` over the PDF
that PDF produces, not a second document.

**Family record cards.** Below the summary, one card per profile in scope:
initial avatar, name, **age**, **blood group**, and a one-line latest entry.

- **Blood group on the card is deliberate.** It is the first thing anyone needs
  in an emergency and the field most likely to be stale, so putting it where a
  caregiver sees it routinely is how it stays correct. It is the user's own
  family's data, already entered by them (`007` emergency profile).
- **The latest-entry line MUST come from a clinic, or be omitted.** The
  reference shows *"Latest: Annual Checkup — Normal."* **"Normal" is a clinical
  interpretation**, and TDC does not make those: if a clinic wrote an outcome,
  render it and attribute it; otherwise show the visit or record without a
  verdict, or show nothing. Deriving "Normal" from our own data is medical
  inference — the same rule that keeps symptom search rejected.
- Cards are scoped by the standard derivation (own profiles ∪ unrevoked
  grants), like every other multi-profile surface.

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

Deliberately minimal, resisting growth (`008` open decision):
account phone number (display only) · **Trust & privacy** (→ S12) ·
**Log out** (one confirm) · app version + build tag. Nothing else — no
preferences, no theme toggle, no notification settings
(reminders configure per-med in `006`).

---

## Requirements

### Functional Requirements
- ~~**FR-001**~~ · ~~**FR-002**~~ · ~~**FR-003**~~ · ~~**FR-004**~~ —
  **deleted 6 Sep 2026, with Splash/Login ceded to `002`.** They covered
  splash auto-advance, the two-step phone→OTP flow with `000000` accepted,
  inline OTP errors with a 30s resend, and profile auto-creation on first
  auth. All four are now owned by [`002` §S0–S3](../002-onboarding-router-activation/spec.md),
  which supersedes them on substance as well as ownership: OTP is 6 digits
  with an enforced cooldown **and an attempt cap**, and failure copy must not
  reveal whether a number is registered. Numbers are retired rather than
  reused, so existing references to `009` FR-005…FR-008 stay valid.
- **FR-005**: The Health Summary screen MUST render the same fields in
  the same order as the `004` PDF mapping, derived entirely from
  structured data — no on-screen editing, no divergence from the PDF.
- **FR-009**: The Health Summary screen MUST offer **PDF** and **Share** as
  peer actions rendering the same `004` field mapping the screen displays —
  never a second hand-maintained document (Principle IX).
- **FR-010**: Family cards MUST show name, age and blood group, scoped to the
  caller's own profiles ∪ unrevoked `caregiver_grants`.
- **FR-011**: A latest-entry outcome (e.g. *"Normal"*) MUST be rendered only
  when a clinic wrote it, and MUST be attributed. The system MUST NOT derive,
  infer, or summarise a clinical outcome from its own data.
- **FR-006**: Share-as-PDF MUST be the screen's single primary action,
  producing the `004`-mapped PDF via the OS share sheet.
- **FR-007**: The Trust screen MUST contain only the five sections above,
  using approved copy, with working links to Family & consent and the
  scan log.
- **FR-008**: Settings MUST contain exactly: phone display, Trust link,
  logout (with confirm), version. **Production additions pending specs:** app lock, data export, account deletion — all staged in `docs/backlog.md`, all in scope.

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
