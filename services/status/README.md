# status

Platform status service — powers the "down detector" page users see on the
marketing website. Checks the health of every TDC component and serves a
public read-only status API.

Currently a stub: spec written (`spec.md`), no code yet.

## What it is in the platform

```
TDC Core API      Emergency Fastlane      TDC Clinic (HMS)      Marketing website
     ▲                    ▲                      ▲                     ▲
     │  independent HTTP health checks, no shared DB, no shared uptime  │
     └──────────────────────────────┬──────────────────────────────────┘
                                     │
                          platform/services/status
                       (checker + public status API)
                                     │
                                     ▼
                    thedoorstepclinic.com/status  (lives in the
                    separate website codebase, fetches this API)
```

**Not a client** in the Core-API sense (Principle: every *app* is a client
that owns no tables — see `health/docs/repo-structure.md`). This service owns
its own small dataset (check history, incidents) and is deliberately
independent of Core, for the same reason Fastlane is independent of Core: a
monitor that depends on the thing it monitors cannot report that thing being
down. See `spec.md` FR-010.

## Notes

- Domain: **not yet locked.** `spec.md` proposes `status.thedoorstepclinic.com`
  for the JSON API; needs an explicit owner decision before it's added to the
  locked domains list in `health/CLAUDE.md`.
- The UI itself is **not** built here — it's a route on the marketing website
  (separate codebase). This directory is the checker + API only.
