# clinic

TDC Clinic — the HMS (CARE fork). Spec only, not started.

Self-contained product directory, same pattern as [`health/`](../health/):
own `CLAUDE.md`, specs, docs, and `.specify/` once work begins.

**This directory never holds CARE-fork source.** The fork (`ohcnetwork/care`)
stays in its own separate repository to preserve upstream rebase capability;
`clinic/` holds TDC's own specs, ADRs, and the `care_tdc` plugin integration
contract, not copied CARE code.
