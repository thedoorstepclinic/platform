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
  memory/constitution.md      # binding principles I-IX (Track A) + X-XV (compliance)
  templates/                  # spec / plan / tasks + security & ABDM checklists
specs/
  001-tdc-phr-patient-app/          # core prototype spec (Track A)
    spec.md                         # WHAT & WHY — golden path, scope, requirements
    plan.md                         # HOW — architecture, constitution check
    research.md                     # Phase 0 decisions + rationale
    data-model.md                   # entities, fields, relationships, snapshot rule
    contracts/                      # Core API (DRF) + Fastlane (FastAPI) contracts
    quickstart.md                   # stand-up + golden-path walkthrough
    tasks.md                        # build tasks in D1–D12 order
  002-onboarding-router-activation/ # router, family add, ABHA linking (Track B)
    spec.md
    flowchart.md                    # mermaid diagrams: account→sync path, states, revoke
  003-abdm-sync-subscription/       # ABDM continuous sync (Track B)
    spec.md
  004-seed-data-and-summary/        # exact seed dataset + Health Summary PDF fields (Track A)
    spec.md
  005-records-capture-timeline/     # multi-page scan, upload, timeline (Track A)
    spec.md
  006-medications-reminders-adherence/ # bundled dose checklist, reminders, stock (Track A)
    spec.md
  007-emergency-profile-card-consent/  # screens 9–11 + abuse pass (Track A)
    spec.md
  008-navigation-app-shell/         # route map, deep links, screen states (Track A)
    spec.md
  009-utility-screens/              # Login, Health Summary, Trust, Settings (Track A)
    spec.md
  010-consult-booking-queue/        # seeded consult booking + queue sim — meetup loop (Track A, P1)
    spec.md
docs/
  design-tokens.md                  # color/type/spacing/touch tokens — code reads these
  repo-structure.md                 # target monorepo layout, CI/deploy split, sharing rules
  track-b-backlog.md                # known gaps deliberately not built in Track A
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

**[001 — TDC PHR Patient App, Prototype](specs/001-tdc-phr-patient-app/spec.md)**
(Track A) — the investor-demo prototype (display name: **TDC Health**).
**Investor pitch: 16 Aug 2026**; meetup/social demos ahead of it. Seeded,
demo-grade, Android + web from one Flutter codebase.
The story: *"A family's health, handled — even when the worst happens."*

**[002 — Onboarding, Router & ABHA Linking](specs/002-onboarding-router-activation/spec.md)**
(Track B) — the first five minutes: one router question, persona-weighted
Home, assisted vs. self-serve family add, Aadhaar-first ABHA detection.

**[003 — ABHA Sync (Subscription-Based Record Inflow)](specs/003-abdm-sync-subscription/spec.md)**
(Track B) — continuous ABDM record sync after a single merged consent screen.

**[004 — Seed Data & Health Summary Contract](specs/004-seed-data-and-summary/spec.md)**
(Track A) — the exact `seed_demo.py` dataset and the Health Summary PDF's
field mapping.

**[005](specs/005-records-capture-timeline/spec.md) · [006](specs/006-medications-reminders-adherence/spec.md) · [007](specs/007-emergency-profile-card-consent/spec.md) · [008](specs/008-navigation-app-shell/spec.md) · [009](specs/009-utility-screens/spec.md)**
(Track A) — per-screen specs: records capture & timeline; medications,
reminders & adherence; emergency profile, card manager & family consent;
navigation shell & deep links; Login / Health Summary / Trust / Settings.
Every screen in the app has an owning spec.

**[010 — Consult Booking & Queue](specs/010-consult-booking-queue/spec.md)**
(Track A, P1) — seeded booking + client-side queue simulation. The *meetup*
demo's beat; the `001` investor golden path is unchanged by it.

> **Track A vs Track B:** `001` and `004`–`010` are the demo prototype. `002`
> and `003` are production MMP work (payments, real ABDM exchange, DPDP
> hardening), re-specced post-funding. Deferred-but-real gaps are listed in
> [`docs/track-b-backlog.md`](docs/track-b-backlog.md).

## Non-negotiables (see constitution)
- The **emergency flow** (Fastlane responder + family blast) is sacred — never cut.
- **Consent is a standing grant**, never a card-tap.
- **Copy discipline** — approved terms only; banned terms never ship.
- **One-command reset** via `seed_demo.py`.
