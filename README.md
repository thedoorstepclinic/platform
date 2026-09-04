# platform

Monorepo for The Doorstep Clinic's customer-facing platform. Each product is
self-contained under its own top-level directory (own `CLAUDE.md`, specs,
docs, and — once built — app/service code).

Every app is a **client**. TDC Core API owns the data; clients consume it over
HTTP and own no tables of their own (20 Aug 2026 decision — see
[`health/docs/repo-structure.md`](health/docs/repo-structure.md)).

## Services

- `services/core-api/` — TDC Core API. Django/DRF + Postgres,
  `api.thedoorstepclinic.com`. Owns profiles, records, meds, consent, and the
  care-discovery directory. Client-agnostic: no response varies by which app
  asked. Not yet scaffolded.
- `services/fastlane/` — Emergency Fastlane. FastAPI, `e.thedoorstepclinic.com`.
  Separate uptime budget by design; reads a denormalized snapshot table rather
  than calling Core. Not a client. Not yet scaffolded.
- [`services/status/`](services/status/) — platform status ("down detector")
  checker + public API. Powers a `/status` page on the marketing website
  (separate codebase). Independent of every component it monitors, same
  reasoning as Fastlane's independence from Core. Spec only, not scaffolded;
  subdomain not yet locked.

## Products (clients)

- [`health/`](health/) — TDC Health, the family PHR app. Spec-first, Spec Kit
  format. Start there at [`health/CLAUDE.md`](health/CLAUDE.md).
- [`doctor/`](doctor/) — TDC Doctor, the clinician app. Spec only, not started.
- [`clinic/`](clinic/) — TDC Clinic, the CARE fork HMS. Specs and docs only;
  source lives in [`thedoorstepclinic/tdc-care`](https://github.com/thedoorstepclinic/tdc-care)
  (forked 30 Aug 2026) and is never copied in here, so the fork can keep
  merging from upstream CARE. Not a client — an upstream source that feeds
  Core.
