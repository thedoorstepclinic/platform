# Feature Specification: Navigation, App Shell & Screen-State Conventions

**Feature Branch:** `008-navigation-app-shell`
**Created:** 2026-07-16
**Status:** Draft
**Owner:** Adi (dev)
**Depends on:** all feature specs (`001`–`007`) — this spec is the frame
they mount into. Design values come from
[`/docs/design-tokens.md`](../../docs/design-tokens.md).
**Purpose:** these specs feed production code generation — this file is the
single answer to "where does every screen live and how do you get there,"
so no navigation decisions are improvised at build time.

## What this is / is NOT

**IS:** The route map (`go_router` route tree — see routing-library note
below), the navigation model (what's a tab, what's a stack, what's a sheet),
the deep-link contract for every notification, and the app-wide screen-state
conventions (loading / error / empty).

> **Updated 2026-08-15 for the Flutter stack** (was written against Expo RN
> and `expo-router`). Routing library: **`go_router`** — Flutter's
> team-maintained router, chosen because it's declarative and supports
> nested/shell routes and path params the same way `expo-router` did, so the
> navigation *model* below (two-level, no tab bar, segmented profile context)
> carries over unchanged. This is a new technical decision this pass is
> introducing, not a previously-locked one — flag if a different router is
> preferred before D1 scaffolding.

**IS NOT:** Per-screen content (owned by `001`–`007`) or visual design
values (owned by `docs/design-tokens.md`).

---

## Navigation model

> **Re-affirmed twice, 9 Sep 2026.** Two separate reference sets carried a bottom
tab bar — `012`'s Queue Status (Status · Clinics · History · Profile) and a
records set (Dashboard · Records · Visits · Pets). **Both rejected**, same
reasoning, plus: `Pets` is out of scope entirely (`001`), `Visits` is a
profile-context screen (`012` B8), and `Records` is reachable from the app
shell without needing a permanent slot. The first re-affirmation, in full:

**Re-affirmed 9 Sep 2026.** A supplied reference design for `012`'s Queue
> Status screen carried a bottom tab bar (Status · Clinics · History · Profile).
> **Rejected, owner decision:** the tab bar is not adopted and the **Clinics tab
> is not built** — a browse destination reachable without choosing a patient is
> the surface Principle VIII (NON-NEGOTIABLE) exists to prevent. The re-entry
> cost this creates is paid by `012` FR-030a's day-of Home entry, not by a
> permanent shell. See `012` §B6 *Divergences from the reference*.

**Two-level architecture — no bottom tab bar.** TDC Health is Home-centric:
Home is the hub, profiles are the spokes. A global bottom tab bar would
imply app-level sections that don't exist ("Home stays quiet — no feed,
ever"), and elderly-first means fewer persistent controls, not more.

1. **Root stack** — auth flow, then the app.
2. **Home** — the hub (family cards + alerts strip, `001` §4.1). Header
   carries two quiet entry points: **Family & consent** and **Settings**
   (Trust screen lives under Settings, and is also linked from consent
   surfaces).
3. **Profile context** — tapping a family card enters that person's space:
   a profile screen with a **segmented control** (not tabs):
   **Records · Meds · Emergency**. Everything about one person happens
   inside their context; the back gesture always returns to Home.
4. **Modals/sheets** — capture flow (`005`), dose checklist (`006`), med
   add/edit, refill, upload tagging. Sheets for quick actions, full-screen
   modals for multi-step flows.

## Route map (`go_router`)

```
/login                              # S0+S1 Splash/Login (002): phone → OTP → in

ShellRoute (authenticated app shell — persistent header, no tab bar)
├── /                               # S2 Home: family cards + alerts strip
├── /family                         # S11 Family & consent (007)
├── /records                        #     All Records: cross-profile search + list (005)
├── /settings                       #     Settings (utility, 009)
├── /settings/trust                 # S12 Trust screen (009)
└── /profile/:profileId             # Profile context: header + segmented control
    ├── (index)                     # S3 Timeline (records segment, default) (005)
    ├── /upload                     # S4 Capture/tagging — full-screen modal (005)
    ├── /record/:recordId           # S5 Record detail (005)
    ├── /summary                    # S6 Health Summary (009 + 004 field map)
    ├── /meds                       # S7 Meds segment: today's checklist + stock (006)
    ├── /meds/:medId                # S8 Med add/edit ('new' = create) — modal (006)
    ├── /meds/slot/:time            #     Dose-occasion checklist — sheet (006)
    ├── /emergency                  # S9 Emergency editor + preview (007)
    ├── /card                       # S10 Card manager + scan log (007)
    ├── /visits                     # B8 Visit History — completed appointments (012)
    ├── /alert/:scanEventId         # S13 Emergency Alert — family blast target (007)
    ├── /book                       # B1 Find care: search + specialty + clinic list (012)
    ├── /book/:clinicId             # B2 Clinic page + fee line (012)
    ├── /book/:clinicId/doctors     # B3 Choose doctor — sheet (012)
    ├── /book/:clinicId/:docId/confirm # B4 Confirm Booking — date + period + profile + pay (012)
    ├── /appointment/:apptId        # B6 Appointment detail + queue position (012)
    └── /appointment/:apptId/reschedule # B7 Reschedule — B4 rebound to an existing appt (012)
```

`ShellRoute` keeps Home's header persistent across the profile-context
routes without introducing a bottom tab bar; modals/sheets (`upload`, med
add/edit, dose checklist) are pushed as full-screen or sheet routes per the
Navigation model above, not separate shell branches.

All 12 numbered screens (`001`) have exactly one route. No screen is
reachable two ways with two different back behaviors.

**Booking sub-tree note (`012`, 6 Sep 2026 — supersedes the `010`+`011` note):**
the `/book` sub-tree was the one place where a single feature spanned two
specs. That seam is gone: `012-appointment-booking` owns B1–B7 end to end.

It stays a linear push stack: every screen's back goes one step up the funnel,
and the profile context (`profileId`) is carried the whole way so the "who is
this for" header never disappears. The doctor sheet is a sheet, not a shell
branch, per the Navigation model above. There is **no** route into this
sub-tree that does not start at a profile — `012` FR-001, and it is what keeps
discovery out of Home (Principle VIII).

**Route change, 9 Sep 2026:** `/book/:clinicId/:docId/slots` is **removed**.
`012`'s B4 and B5 merged into one "Confirm Booking" screen at `.../confirm`,
which carries date, period, profile and payment in one scroll. B5's number is
retired rather than reused, so B6/B7 references stay correct.

**`/appointment/:apptId/reschedule` is B4 rebound, not a second slot screen.**
It renders the same date + period picker against an existing appointment, so it lives beside
the appointment rather than back inside `/book` — a reschedule that pushed the
user back through the booking funnel would put Clinic and Doctor in their back
stack as if they were still choosable, which they are not (`012` §B7: same
doctor only). Its back goes to B6, never into the booking funnel.

## Deep-link contract

Every notification the app emits maps to exactly one route. This table is
the contract (`001` NFR-006: notification → action <10s, ≤1 intermediate
screen — deep links are how that's met):

| Trigger | Link target | Notes |
|---|---|---|
| Dose-slot reminder (`006`) | `/profile/[id]/meds/slot/[time]` | Straight into the bundled checklist |
| Low-stock alert | `/profile/[id]/meds` | Stock bar visible on landing |
| Missed-dose nudge | `/profile/[id]/meds/slot/[time]` | Same checklist, past slot |
| Family blast — card scanned (`001` FR-014) | `/profile/[id]/alert/[scanEventId]` | **Changed 9 Sep 2026** — was the card manager's scan log, a maintenance screen. Now the Emergency Alert screen (`007` S13, FR-011): what happened, where if known, who if the responder said, and one tap to call them |
| Test-scan blast (`007`) | `/profile/[id]/alert/[scanEventId]` | Same screen, banner and entry carry the Test label — a test must exercise the real path (`007` FR-005) |
| ABDM record arrived (`003`, gated on ABDM certification) | `/profile/[id]` | Timeline, new record on top |
| Upcoming appointment (alerts strip, `012`) | `/profile/[id]/appointment/[apptId]` | Queue view visible on landing |
| Appointment reminder T−24h (`012`) | `/profile/[id]/appointment/[apptId]` | Queue block absent — it appears only on the day |
| Appointment reminder T−2h (`012`) | `/profile/[id]/appointment/[apptId]` | Queue block present; token visible without scrolling |

Cold-start rule: a deep link into a logged-out app goes to login and then
**continues to the target** — never dumps the user on Home after auth.

**Dead-link rule (`012`).** An appointment reminder can outlive its
appointment — the row may have been cancelled or rescheduled on another
device. Both reminders are cancelled locally when that happens (`012` FR-021),
but a link that arrives anyway MUST resolve to B6 showing the terminal state
(`Cancelled`, or `Moved to …` with a link forward), never to an error screen
and never to a 404. A notification that leads nowhere is worse than one that
was never sent.

## Screen-state conventions (app-wide)

Every data-bearing screen implements exactly these four states — specs
`001`–`007` define the *content* of each; this defines the *pattern*:

- **Loading:** skeleton placeholders (cards/rows), never a full-screen
  spinner. 250ms delay before showing (no flash on fast loads).
- **Empty:** an instructive empty state that names the action (the `001`
  §4.1 checklist prompts are these). Never a bare "No data."
- **Error:** plain-language message + Retry button. Copy pattern:
  *"Couldn't load [thing]. Check your connection and try again."* Never
  error codes, never blame ("you did X wrong").
- **No offline mode** (hard-prohibited, `CLAUDE.md`): no cached-data
  pretense, no sync queues. A network failure is an honest error state
  with retry. The single exception is already-scheduled **local** dose
  reminders (`006`), which fire without network by nature.

Demo-mode banner: when `DEMO_MODE` is on, a thin, dismissible "Demo" tag
shows on the login screen only — never in the main app shell (it would
poison every investor screenshot).

---

## Requirements

### Functional Requirements
- **FR-001**: Navigation MUST follow the two-level model (Home hub →
  profile context) with no global bottom tab bar.
- **FR-002**: The profile context MUST present Records / Meds / Emergency
  as a segmented control inside one layout; back from anywhere in a
  profile returns to Home.
- **FR-003**: Every screen MUST be reachable by exactly one route per the
  route map; route additions require updating this spec.
- **FR-004**: Every notification type MUST deep-link per the contract
  table; cold-start deep links MUST survive the auth flow and land on
  their target.
- **FR-005**: Every data-bearing screen MUST implement the four screen
  states per the conventions above.
- **FR-006**: All colors, type sizes, spacing, radii, and tap-target
  minimums MUST come from `docs/design-tokens.md` tokens — no literal
  values in component code.
- **FR-007**: The Card manager (`007`) and Emergency editor (`007`) MUST
  remain reachable within two taps from Home (family card → Emergency
  segment) — emergency surfaces are never buried deeper than that.

### Non-Functional Requirements
- **NFR-001 (Elderly-first):** back behavior is always the OS-standard
  gesture/button; no custom gestures anywhere.
- **NFR-002 (Web parity):** the same route map serves the web build;
  routes are the URLs. Screens with Android-only capabilities follow their
  owning spec's platform scoping (e.g. `005` capture).

---

## Open Decisions
| Decision | Owner | Notes |
|---|---|---|
| Segmented-control labels: "Records · Meds · Emergency" vs adding a 4th "Card" segment | Adi/Soham | Lean 3 segments with Card reached from the Emergency segment (card is part of the emergency story, and 3 segments fit elderly-first type sizes better). |
| ~~Settings screen contents~~ | — | **RESOLVED in `009`:** phone display · Trust link · logout · version. **Re-open (6 Sep):** production Settings also needs app lock, data export, and account deletion — all staged in `docs/backlog.md`. |

---

## Review & Acceptance Checklist
- [x] Every `001` screen mapped to exactly one route.
- [x] Every notification in `006`/`007`/`001` covered by the deep-link contract.
- [x] State conventions defined once, referenced everywhere.
- [x] No design literals — tokens file is the single source.
- [x] Requirements testable (route uniqueness, 2-tap emergency rule, cold-start deep links).

## Execution Status
- [x] Navigation model decided (hub-and-spoke, no tab bar)
- [x] Route map complete
- [x] Deep-link contract complete
- [x] Review checklist passed
