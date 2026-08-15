# Repo structure — health/ product directory

Status: **decided, not yet built.** `health/` currently holds specs only; this
document is the target layout D1 scaffolds into. It records decisions made
before any code existed, because directory renames are free now and migrations
are not.

`health/` is a self-contained product directory inside the `platform`
monorepo (see `platform/README.md`) — patient-app-scoped only. **TDC Doctor is
not nested here**; it is its own top-level product directory,
`platform/doctor/`, built independently when it actually starts (15 Aug 2026
decision, see `CLAUDE.md` changelog). TDC Clinic is likewise its own top-level
`platform/clinic/`.

Authority: `CLAUDE.md` for names/IDs/domains, `.specify/memory/constitution.md`
for principles. This file only covers layout, CI, and what may be shared.

## Layout

```
apps/
  health/            # Flutter app (Android + web) — in.thedoorstepclinic.health
services/
  core-api/          # Django/DRF + Postgres — api.thedoorstepclinic.com
  fastlane/          # FastAPI responder service — e.thedoorstepclinic.com
docs/
specs/
.specify/
```

Three top-level units, two of them Python (`core-api`, `fastlane`), one of
them Dart/Flutter (`apps/health`). `apps/` stays plural for consistency with
`services/`, even though `health/` scopes to a single app — there is no
second app landing inside it.

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

For Track A this is two or three small workflows, not a platform. Resist
anything that needs a build orchestrator.

## Tooling: none, for now

> **STALE — stack changed from Expo React Native to Flutter (15 Aug 2026,
> see `CLAUDE.md` changelog).** This section's reasoning (npm workspaces,
> Metro bundler, pnpm hoisting) is JS-ecosystem-specific and no longer
> applies. Needs a Flutter-tooling pass (e.g. Melos vs. plain per-package
> `pub`) before TDC Doctor lands — not yet decided, left here for reference
> until then.

No npm workspaces, no Turborepo, no Nx for Track A. `apps/health` runs its own
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
