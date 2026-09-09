# Repo structure — health/ product directory

Status: **decided, not yet built.** `health/` currently holds specs only; this
document is the target layout D1 scaffolds into. It records decisions made
before any code existed, because directory renames are free now and migrations
are not.

`health/` is a self-contained **client product** directory inside the
`platform` monorepo (see `platform/README.md`). **TDC Doctor is not nested
here**; it is its own top-level product directory, `platform/doctor/`, built
independently when it actually starts (15 Aug 2026 decision). TDC Clinic is
likewise its own top-level `platform/clinic/`.

**Amended 20 Aug 2026 — services moved out of `health/`.** `health/` was
"patient-app-scoped only" and held `services/core-api/` and
`services/fastlane/` inside it. That stopped being true the moment TDC Core
API became the directory owner for every client (`011-care-discovery`, whose
booking flow is now owned by `012-appointment-booking`): a
service the doctor app calls cannot live inside the patient app's product
directory without giving `doctor/` an upward dependency into `health/`.
Services now sit at `platform/services/`. `health/` keeps its app, specs and
docs, and is now scoped to **the patient client**, not to the patient stack.

Authority: `CLAUDE.md` for names/IDs/domains, `.specify/memory/constitution.md`
for principles. This file only covers layout, CI, and what may be shared.

## Layout

```
platform/
  services/                # shared, client-agnostic
    core-api/              # Django/DRF + Postgres — api.thedoorstepclinic.com
    fastlane/              # FastAPI responder — e.thedoorstepclinic.com
  health/                  # TDC Health (this product directory)
    apps/
      health/              # Flutter app — in.thedoorstepclinic.health
    docs/  specs/  .specify/
  doctor/                  # TDC Doctor (Flutter client, not started)
  clinic/                  # TDC Clinic — specs/docs only; source in its own repo
```

**TDC Clinic's source repo is [`thedoorstepclinic/tdc-care`](https://github.com/thedoorstepclinic/tdc-care)**
(forked from CARE 30 Aug 2026, default branch `develop`). `platform/clinic/`
is the specs-and-docs stub this document promised; it is now created. The fork
is never vendored in — a copied fork cannot cleanly merge from an actively
developed upstream, which is the whole reason for the exception.

`health/` is now one unit: the Flutter client plus its specs and docs. `apps/`
stays plural even though it scopes to a single app — there is no second app
landing inside it.

## Every app is a client (20 Aug 2026 decision)

**TDC Core API owns the data. TDC Health, TDC Doctor, and any future client
consume it over HTTP and own no tables of their own.** This was already true
in practice for records, meds and profiles; it became a stated rule when the
care-discovery directory (`011`) needed to be readable by both the patient app
and the eventual doctor app.

Concretely:
- **No client-local directory tables, no client-local caches presented as
  data.** If TDC Doctor needs the clinic list, it calls
  `GET /api/v1/clinics` — the same endpoint TDC Health calls, with the same
  shape. A second implementation is a second source of truth.
- **No client talks to another client's backend**, and no client reads
  Postgres directly. Fastlane is the single exception to the "one API" shape,
  and it is a *reader* of a snapshot table, not a client — that separation is
  Principle I and it stands.
- **Directory endpoints are client-agnostic**: no response field may vary by
  which app asked. If the doctor app needs a different projection, that is a
  new endpoint, not a branch inside an existing one.

### core-api aggregates, one API out

The directory is currently seeded locally (a `DEMO_MODE` fixture). core-api becomes the
**aggregation point**, not the sole author: it ingests TDC's own facilities
from TDC Clinic (the CARE fork) and external providers from UHI / HFR / HPR,
and still serves one directory API to every client. Rows carry `source`
(`seed` | `tdc_clinic` | `uhi`) and `external_ref` (HFR facility id / HPR
professional id). Clients never learn where a row came from — that opacity is
what keeps them clients.

`source = seed` rows are `DEMO_MODE` fixtures, `# DEMO-MODE` tagged per Principle XIV; `tdc_clinic` has no external gate (we own the fork), `uhi` is gated on UHI onboarding.

## Why a monorepo at all

One developer, ~4h/day, and a demo where a data-model change ripples through
Core → Fastlane snapshot → seed script → app screen in a single sitting.
Cross-repo PRs for that ripple would cost more than the isolation buys. The
constitution's same-day seed-ripple rule (Development Workflow §5) is only
practical inside one working tree.

The thing a monorepo must **not** cost us is Principle I.

## CI and deploy: path-filtered, per-service

**Non-negotiable: Fastlane deploys independently of Core API.**

Principle I says Fastlane must survive Core being down. A shared pipeline that
builds, tests, and ships both services together silently repeals that — a bad
Core migration would take the responder page with it, and the separate-service
architecture becomes decoration. The physical split (`001` research.md R1) is
only real if the *release* path is split too.

Concretely:

- Each of `apps/health`, `services/core-api`, `services/fastlane` gets its own
  workflow, triggered on `paths:` for its own directory (plus shared config).
- No job in the Fastlane workflow depends on a Core job.
- Fastlane can be redeployed, rolled back, or left frozen while Core ships.
- A red Core test suite must never block a Fastlane hotfix.

Today this is two or three small workflows, not a platform. Resist
anything that needs a build orchestrator.

## Tooling: none, for now

> **STALE — stack changed from Expo React Native to Flutter (15 Aug 2026,
> see `CLAUDE.md` changelog).** This section's reasoning (npm workspaces,
> Metro bundler, pnpm hoisting) is JS-ecosystem-specific and no longer
> applies. Needs a Flutter-tooling pass (e.g. Melos vs. plain per-package
> `pub`) before TDC Doctor lands — not yet decided, left here for reference
> until then.

No npm workspaces, no Turborepo, no Nx. `apps/health` runs its own
`npm install` and Expo toolchain; each service has its own venv and
`requirements.txt`. There is exactly one JS package — a workspace manager would
be pure ceremony.

When TDC Doctor arrives and there is real shared TypeScript, adopt **npm
workspaces**, not pnpm. Reason: React Native's Metro bundler and native module
resolution are well-trodden with npm/yarn hoisting; pnpm's symlinked store
regularly needs `node-linker=hoisted` and per-package patches to work with RN.
Boring and already-paid-for beats fast-and-fiddly.

## What may be shared, and what may not

**Do not share Python code between `core-api` and `fastlane`.** They share a
*table* — the denormalized `emergency_payload` snapshot — not a codebase. A
common `models.py` or a `tdc-shared` package would reintroduce exactly the
coupling the two-service split exists to prevent: an import graph that makes a
Core refactor a Fastlane deploy. Fastlane defines its own thin read model over
the snapshot columns it needs. Duplicating a handful of field names is the
cheaper mistake.

The contract between them is the snapshot's shape, and that is specified in
`001/data-model.md`, not enforced by a shared import.

**TDC Clinic (the CARE fork) stays in its own repository.** You cannot
usefully monorepo a fork you continue to pull upstream into; vendoring it would
turn every upstream merge into a repo-wide conflict. It integrates over the P1
webhook, as an external system.

**When TDC Doctor lands**, share:

- design tokens' *palette* (`docs/design-tokens.md` colors)
- API client and generated request/response types
- domain types shared across both apps

Do **not** share the *type scale, spacing, or tap-target minimums*. TDC Health
is elderly-first — large type, large targets, readable at arm's length
(Principle VII). A clinician app is density-first: a doctor scanning a queue
wants more rows per screen, not fewer. Sharing the scale would force one of the
two apps to be wrong. Share the palette so they look related; let the rhythm
differ because the users differ.
