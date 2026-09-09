# TDC Health — PHR (Personal Health Record) App

Part of the TDC (The Doorstep Clinic) ecosystem. This repository is
**spec-first**: features are hardened as specifications (GitHub Spec Kit format)
before and alongside the visual-first prototypes.

**Start here:** [`CLAUDE.md`](CLAUDE.md) — the locked-decisions context pack.
If any other doc, chat, or generated code conflicts with it, `CLAUDE.md` wins.

## What's here

```
CLAUDE.md                     # locked decisions — source of truth, read first
.specify/
  memory/constitution.md      # v3.0.0 — principles I-XV, ALL binding, no classes
  templates/                  # spec / plan / tasks + security & ABDM checklists
specs/
  001-tdc-phr-patient-app/          # core app spec — the only complete spec-kit set
    spec.md                         # WHAT & WHY — golden path, scope, requirements
    plan.md                         # HOW — architecture, constitution check
    research.md                     # Phase 0 decisions + rationale
    data-model.md                   # entities, fields, relationships, snapshot rule
    contracts/                      # Core API (DRF) + Fastlane (FastAPI) contracts
    quickstart.md                   # stand-up + golden-path walkthrough
    tasks.md                        # build tasks, dependency-ordered
    checklists/security.md
  002-onboarding-router-activation/ # onboarding, emergency card, ABHA activation
    spec.md · research.md · flowchart.md
  003-abdm-sync-subscription/       # ABDM continuous sync — GATE: ABDM certification
    spec.md
  004-seed-data-and-summary/        # seed/fixture dataset + Health Summary PDF fields
    spec.md
  005-records-capture-timeline/     # multi-page scan, upload, timeline
    spec.md
  006-medications-reminders-adherence/ # bundled dose checklist, reminders, stock
    spec.md
  007-emergency-profile-card-consent/  # screens 9-11 + abuse pass
    spec.md
  008-navigation-app-shell/         # route map, deep links, screen states
    spec.md
  009-utility-screens/              # Health Summary, Trust, Settings (login ceded to 002)
    spec.md
  010-consult-booking-queue/        # SUPERSEDED by 012 — kept as decision record
    spec.md
  011-care-discovery/               # SUPERSEDED by 012 — kept as decision record
    spec.md · checklists/security.md
  012-appointment-booking/          # the live booking spec — discovery -> lifecycle
    spec.md · checklists/security.md
docs/
  design-tokens.md                  # color/type/spacing/touch tokens — code reads these
  repo-structure.md                 # target monorepo layout, CI/deploy split, sharing rules
  backlog.md                        # real requirements without a spec yet — all in scope
  compliance-baseline.md            # ABDM M1-M3 / WASA / DPDP facts + sources (dated)
```

> **No application code yet** — the tree above is the whole repo. `docs/repo-structure.md`
> defines the `apps/` + `services/` layout that D1 scaffolds into.

## Spec Kit workflow

1. **Constitution** — `.specify/memory/constitution.md` sets non-negotiable
   principles. Every plan gates against it.
2. **Specify** — `specs/NNN-slug/spec.md` captures WHAT and WHY (no stack).
3. **Plan** — `plan.md` chooses the HOW and runs the Constitution Check.
4. **Design** — `data-model.md`, `contracts/`, `quickstart.md`.
5. **Tasks** — `tasks.md` derives ordered, parallelisable work.

## Current specs

**[001 — TDC PHR Patient App](specs/001-tdc-phr-patient-app/spec.md)** — the
core app (display name: **TDC Health**): profiles, records timeline, Health
Summary, medications, emergency card, Trust screen. Android + web from one
Flutter codebase. Currently the **only feature with a complete spec-kit set**.
The story: *"A family's health, handled — even when the worst happens."*

**[002 — Onboarding, Emergency Card & ABHA Activation](specs/002-onboarding-router-activation/spec.md)**
— the first five minutes: splash, login carousel, phone + OTP, and the
**emergency-card builder as the activation moment**, with ABHA offered after
rather than as a gate. *(The router question the directory is named for was
deleted in the 18 Aug rewrite; the directory name is kept for link stability.)*

**[003 — ABHA Sync (Subscription-Based Record Inflow)](specs/003-abdm-sync-subscription/spec.md)**
— continuous ABDM record sync after a single merged consent screen.
**GATE: ABDM certification** (WASA audit is a precondition for M1).

**[004 — Seed Data & Health Summary Contract](specs/004-seed-data-and-summary/spec.md)**
— the `seed_demo.py` fixture dataset and the Health Summary PDF's field
mapping. Fixtures are a development and test asset, never shipped content.

**[005](specs/005-records-capture-timeline/spec.md) · [006](specs/006-medications-reminders-adherence/spec.md) · [007](specs/007-emergency-profile-card-consent/spec.md) · [008](specs/008-navigation-app-shell/spec.md) · [009](specs/009-utility-screens/spec.md)**
— per-screen specs: records capture & timeline; medications, reminders &
adherence; emergency profile, card manager & family consent; navigation shell
& deep links; Health Summary / Trust / Settings. Every screen has an owning
spec.

**[012 — Appointment Booking](specs/012-appointment-booking/spec.md)** — the
whole booking loop end to end: find care, clinic page, doctor, slot, confirm,
reschedule, cancel, reminders. Supersedes `010` (booking + queue) and `011`
(care discovery), which are retained as decision records only.

> **One project, built to production** (owner decision, 6 Sep 2026). The Track
> A / Track B split is **abolished** — constitution v3.0.0 §One Standard. There
> is no demo track and no deferral target: every feature is a priority, owns a
> complete spec-kit set, and is built to production quality. Where an external
> integration is genuinely unreachable (ABDM, UHI, payment rails), the spec
> names **the gate and its owner** — not a track. The prototype survives as a
> `DEMO_MODE` switch that is off by default and substitutes data, never
> behaviour. Requirements that don't yet own a spec are staged in
> [`docs/backlog.md`](docs/backlog.md) — all of it in scope.

## Non-negotiables (see constitution)
- The **emergency flow** (Fastlane responder + family blast) is sacred — never cut.
- **Consent is a standing grant**, never a card-tap.
- **Copy discipline** — approved terms only; banned terms never ship.
- **One-command reset** via `seed_demo.py`.
