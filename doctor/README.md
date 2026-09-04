# doctor

TDC Doctor — the clinician-facing app. Spec only, not started.

Self-contained product directory, same pattern as [`health/`](../health/):
own `CLAUDE.md`, specs, docs, and `.specify/` once work begins. Track B —
frozen until TDC Health's investor pitch (16 Aug 2026) and post-funding
planning.

Package ID (locked): `in.thedoorstepclinic.doctor`.

## Client rule (20 Aug 2026 decision)

TDC Doctor is a **client of TDC Core API** (`platform/services/core-api/`). It
owns no tables and no local database of record. Anything it displays — the
clinic and doctor directory included — it fetches from Core's API, the same
endpoints TDC Health calls, in the same shape.

This is worth stating before a line of code exists, because the directory is
the obvious thing to build twice: a clinician app "obviously" needs its own
doctor table, and the moment it has one there are two answers to who Dr. Kavya
is. If the doctor app needs a different projection of directory data, that is a
new Core endpoint, not a local table and not a branch inside an existing
response.

See [`health/docs/repo-structure.md`](../health/docs/repo-structure.md) —
*Every app is a client* — for the full rule and the Track B aggregation model
(Core ingests TDC Clinic + UHI/HFR/HPR and still serves one API out).
