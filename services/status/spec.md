# Feature Specification: Platform Status Page ("down detector")

**Feature Branch:** n/a — platform-level, not on the health/ Spec Kit numbering
track (no `.specify/` tooling installed at platform root; see Open Decisions).
**Created:** 2026-08-31
**Status:** Draft
**Owner:** Adi
**Input:** "adding a system page a down detector page on our website, write
down a spec for platform that will show us all operations app wise are shown
availability."
**Decisions taken (31 Aug 2026, via clarifying questions):**
1. **Hybrid data source.** Real scheduled health checks for whatever is
   actually deployed; an explicit "Not yet launched" state for everything
   else. No component may show "Operational" without a real check behind it.
2. **Scope: six components** — TDC Core API, Emergency Fastlane, TDC Health,
   TDC Doctor, TDC Clinic (HMS), and the marketing website.
3. **Placement: the page itself lives on the marketing website** (separate
   codebase, outside this monorepo). This spec covers the checker service and
   the public API the website's `/status` route reads from — not the page's
   visual design.

**Depends on / relates to:** `README.md` (Services list — this becomes a
third entry alongside core-api and fastlane) · `health/docs/repo-structure.md`
("every app is a client" — this service is deliberately *not* a client, see
Decisions §Architecture) · `health/CLAUDE.md` locked domains list (a new
subdomain is proposed here, not yet locked).

## Why this exists

The platform is becoming multiple independently-deployed pieces (Core API,
Fastlane, three client apps, the CARE-fork HMS, the marketing site) with
**independent uptime budgets by design** — Fastlane's separation from Core
exists specifically so it survives Core being down. That design only pays off
if someone (Adi, and eventually users) can *see* which piece is down without
guessing. A public status page also does double duty as a trust signal: "you
can see when we're down" is a stronger claim than "we say we're reliable,"
and this repo already has a standing principle (Health's Principle VI —
claims must be literally true) that a status page can violate worse than
almost anything else if it fakes green.

**The one thing this spec is not allowed to produce:** a page that shows
"Operational" for a service that has never been deployed. As of this writing,
Core API and Fastlane are "not yet scaffolded" (`README.md`) and Doctor/Clinic
haven't started — a naive status page defaulting everything to green would be
actively dishonest on day one.

## What this is / is NOT

**IS:** A small, independent checker service that pings each deployed
component on a schedule, stores results, and exposes a public read-only JSON
API. A contract the marketing website's status page renders against.

**IS NOT:**
- Not the status page's UI/visual design — that's a marketing-website
  concern, out of this repo.
- Not an alerting/paging system (Slack/email/SMS on down). Worth building
  later; not in scope here — see Open Decisions.
- Not a synthetic-transaction / deep health monitor (e.g. "can a user
  actually book an appointment end to end"). Scope is reachability + basic
  liveness of each component's own `/healthz`, nothing deeper, for now.
- Not authoritative for components it doesn't check directly. Health and
  Doctor are Flutter clients with no server of their own — see FR-004.

## User Scenarios & Testing

### Primary User Story
A visitor (or Adi, mid-incident) opens `thedoorstepclinic.com/status` and
sees, at a glance, which of the six components are up, which are down, and
which haven't launched yet — each with a last-checked timestamp, and any
open incident note explaining what's happening.

### Acceptance Scenarios
1. **Given** Core API's health check has failed 3 consecutive times, **When**
   the status page loads, **Then** Core API shows "Down," Health and Doctor
   (which depend on Core API) show "Degraded," and the timestamp of the last
   check is visible.
2. **Given** a component has no `check_url` configured (not yet deployed),
   **When** the status page loads, **Then** it shows "Not yet launched," never
   "Operational" and never a fabricated uptime percentage.
3. **Given** all checks are passing, **When** the status page loads, **Then**
   every deployed component shows "Operational" with its last-checked time
   and a 90-day uptime percentage computed from real stored checks.
4. **Given** Adi manually posts an incident note ("investigating elevated
   Fastlane latency"), **When** the status page loads, **Then** the note is
   attached to Fastlane's row regardless of what the automated check currently
   says.
5. **Given** the checker service itself is unreachable, **When** the website
   fetches the status API, **Then** the page shows "Status temporarily
   unavailable" rather than silently rendering stale data as current, or
   erroring blank.

### Edge Cases
- A single failed check (transient network blip) must not flip a component to
  "Down" — see FR-005 (anti-flap threshold).
- Core API goes down: the checker itself must still be able to report this,
  because it does not depend on Core API being reachable (FR-010).
- A new component (e.g. Clinic HMS going live) must be addable via
  configuration, not a website redeploy (FR-011).
- Two components share a dependency (Health and Doctor both derive from Core
  API + Fastlane) — a single upstream outage must not read as two unrelated
  incidents.

## Requirements

### Functional Requirements
- **FR-001**: The API MUST report current status for each of the six in-scope
  components: TDC Core API, Emergency Fastlane, TDC Health, TDC Doctor, TDC
  Clinic, marketing website.
- **FR-002**: Each component MUST be in exactly one of three states at any
  time: **Operational**, **Down** (or **Degraded**, see FR-004), **Not yet
  launched**. A component with no `check_url` configured MUST always report
  "Not yet launched" — this is a config fact, not a check result.
- **FR-003**: For components with a `check_url`, status MUST be derived from
  scheduled automated health checks (see FR-005), not manually toggled,
  except for incident annotations (FR-006).
- **FR-004**: TDC Health and TDC Doctor are Flutter clients with no server of
  their own (every app is a client — `health/docs/repo-structure.md`). Their
  status MUST be **derived**, not independently checked: Operational only if
  both Core API and Fastlane are Operational; Degraded if either upstream is
  Down; Not yet launched if the app itself has no deployed build to point a
  check at yet (true today for both).
- **FR-005**: A component flips to "Down" only after **3 consecutive** failed
  checks, and back to "Operational" only after **2 consecutive** successful
  checks (anti-flap). Each check MUST use a fixed timeout (proposed: 5s).
- **FR-006**: Adi MUST be able to create, edit, and resolve an incident note
  attached to a component, shown on the status page alongside (not instead
  of) the automated state. Incident notes are the only manually-authored
  content — everything else is derived from real data (FR ties back to Why
  this exists — no fabricated status).
- **FR-007**: The public status API MUST be read-only and unauthenticated,
  and MUST NOT leak internal detail beyond: component name, state, last
  checked time, latency, 90-day uptime %, open incident notes. No internal
  URLs, no stack traces, no infra hostnames beyond the public check target.
- **FR-008**: The API MUST expose the last-checked timestamp per component,
  not just current state — a status with no timestamp is not trustworthy.
- **FR-009**: 90-day uptime percentage MUST be computed from stored check
  history (successful checks / total checks in window). A component with
  less than 90 days of history reports uptime over the history it actually
  has, labeled with the shorter window — never backfilled or estimated.
- **FR-010**: The checker service's own operation MUST NOT depend on Core API
  (or any other monitored component) being reachable — own database, own
  deploy, independent of every component it watches. Mirrors Fastlane's
  existing independence-from-Core design for the same reason.
- **FR-011**: The list of monitored components MUST be configuration (a
  table row), not hardcoded in application code — adding Clinic HMS when it
  goes live is a data change, not a deploy.
- **FR-012**: If the checker/API itself is unreachable from the website, the
  website MUST show an explicit "status temporarily unavailable" state. It
  MUST NOT render a cached snapshot without clearly marking it as stale and
  timestamped.

### Key Entities
- **Component**: id · name · kind (`service` | `client_app` | `website`) ·
  `check_url` (nullable — null means "not yet launched") · `depends_on`
  (self-referential, for derived-status components like Health/Doctor) ·
  public sort order.
- **Check**: component FK · timestamp · result (`up`/`down`/`timeout`) ·
  latency_ms · HTTP status code (nullable on timeout).
- **Incident**: component FK · title · description · started_at · resolved_at
  (nullable while open) · severity — manually authored, shown independent of
  automated state.
- **StatusSnapshot**: one row per component, the current derived state,
  updated by the checker after each check cycle — read path for the API, so
  a page load never does live aggregation over the full `Check` history.

## Architecture

```
platform/services/status/
  checker    — scheduled job, pings each component's check_url on an
               interval, writes Check rows, updates StatusSnapshot,
               applies the anti-flap threshold (FR-005) and the
               derived-status rule for client apps (FR-004)
  api        — public read-only endpoint(s), reads StatusSnapshot +
               open Incidents, never touches Check history directly
               on the hot path (FR-009's uptime % is precomputed
               alongside the snapshot, not calculated per-request)
```

Deliberately **not** a client of Core API and **not** inside `health/`,
`doctor/`, or `clinic/` — no single product owns the responsibility of
knowing whether the *others* are up. Same reasoning that put `services/`
at the platform level in the first place (`health/docs/repo-structure.md`).

**Independence from what it monitors (FR-010) is the load-bearing design
constraint.** A status checker with a foreign key into Core API's database,
or one deployed behind Core API's ingress, cannot report Core API being
down. It needs its own datastore and its own deploy, exactly like Fastlane's
existing separation from Core, for the same reason.

### API surface (proposed)
- `GET /api/v1/status` → array of components, each with current state, last
  checked timestamp, 90-day uptime %, and any open incident.
- `GET /api/v1/status/history?component=core-api&days=90` → check history for
  an uptime graph, if the website wants one.

## Not-yet-launched handling

As of 2026-08-31, real status for this platform is mostly "nothing is
deployed yet": Core API and Fastlane are not scaffolded, Doctor and Clinic
haven't started, and Health is a local Track A demo prototype with no public
build. Under FR-002, **every one of those reports "Not yet launched" today**
— which is correct, not a placeholder to fix later. The marketing website is
the only component plausibly reachable right now, and even that needs a
confirmed public `check_url` before it can show anything but "Not yet
launched" too. This spec is written so the page tells the truth on day one
and starts reporting real green as each piece actually ships, component by
component, with no code change required (FR-011).

## Open Decisions (need an explicit owner call, not assumed)

1. **Subdomain for the public API.** Proposed: `status.thedoorstepclinic.com`,
   following the existing `api.` / `e.` / `clinic.` / `console.` pattern. Not
   yet added to the locked domains list in `health/CLAUDE.md` — needs Adi's
   sign-off before it's treated as locked.
2. **Stack for the checker service.** The platform's backend stack is
   Python-only (Django/DRF for Core, FastAPI for Fastlane). Given FR-010's
   independence requirement and how small this service's job is, a
   lightweight FastAPI service (Fastlane's pattern) fits better than pulling
   in Django/DRF for what is essentially a cron job plus a read API — flagged
   as a recommendation, not a lock.
3. **Alerting.** This spec covers passive display only. Paging Adi
   (Slack/email/SMS) when a component goes Down is valuable but explicitly
   out of scope here — separate feature if wanted.
4. **Clinic HMS health endpoint.** TDC Clinic is a CARE fork
   (`thedoorstepclinic/tdc-care`); whether upstream CARE exposes a usable
   `/healthz` (or TDC needs to add one in the fork) is unconfirmed — check
   before wiring FR-001's sixth component.
5. **Check interval.** Proposed 60s per component; not fixed. Affects both
   how fast "Down" is detected (interacting with FR-005's 3-strike threshold
   — 60s interval means ~3 minutes to detect a real outage) and load on
   monitored components.

## Review & Acceptance Checklist

### Content Quality
- [x] Focused on what the status page must truthfully show and why.
- [x] Infrastructural detail included deliberately (this request is itself
      infrastructural — the spec-template's usual "no schema, no endpoints"
      guidance is waived per its own exception clause).
- [ ] Non-technical stakeholder pass — not yet reviewed by anyone but Adi.

### Requirement Completeness
- [ ] Subdomain decision (Open Decisions §1) — pending.
- [ ] Stack decision (Open Decisions §2) — pending, recommendation given.
- [x] Scope clearly bounded (six components, listed explicitly; alerting and
      synthetic transactions explicitly excluded).
- [x] Dependencies identified (Core API, Fastlane, CARE fork, marketing
      website codebase).
