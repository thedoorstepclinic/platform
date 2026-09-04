# clinic

TDC Clinic — the hospital management system, a fork of
[CARE](https://github.com/ohcnetwork/care) (a Digital Public Good for
decentralized healthcare administration).

## Source lives elsewhere — deliberately

**Repo:** [`thedoorstepclinic/tdc-care`](https://github.com/thedoorstepclinic/tdc-care)
· forked 30 Aug 2026 · default branch `develop` (CARE's, kept for upstream
merges).

The CARE-fork source is **never copied into this monorepo** (locked decision,
`health/CLAUDE.md`). A vendored fork cannot cleanly pull upstream, and CARE is
an actively developed project we want to keep merging from. This directory
holds TDC Clinic's specs and docs only — the code stays in its own repo, on
its own branch structure, tracking upstream.

Currently a stub: no specs written yet. Track B.

## What it is in the platform

```
TDC Health (Flutter)   TDC Doctor (Flutter)   ← clients, own no tables
            └──────────┬──────────┘
                       ▼
              TDC Core API  ←—(P1 webhook)— TDC Clinic (this, CARE fork)
                       │
                       ▼
              Emergency Fastlane
```

TDC Clinic is **not** a client of Core — it is an upstream *source*. Two
integration points are already specced from the Core side:

- **P1 HMS→app slice** (`health/specs/001-tdc-phr-patient-app/spec.md`) — an Rx
  created in the fork lands in the patient's timeline. Shared-DB or webhook
  fake is acceptable for the demo.
- **Track B directory + queue** (`health/specs/011-care-discovery/spec.md`) —
  TDC Clinic becomes the source for TDC's own clinics, doctors and real slot
  availability, ingested by Core (`source = tdc_clinic`). Core still serves one
  directory API out; clients never learn where a row came from.

Track A depends on neither: the directory is seeded and the queue is a
client-side simulation.

## Notes

- **The fork is public.** CARE is open source, so that is normal and fine for
  the fork itself. It stops being automatically fine once TDC-specific
  configuration, deployment manifests, or facility data land in it — keep
  secrets out of the repo regardless, and revisit visibility before TDC config
  is committed.
- Domain (locked): `clinic.thedoorstepclinic.com`.
