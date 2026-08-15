# Implementation Plan: [FEATURE NAME]

**Branch:** `[###-feature-slug]` · **Date:** [DATE] · **Spec:** [link to spec.md]
**Input:** Feature specification from `specs/[###-feature-slug]/spec.md`

## Summary
[One paragraph: what is being built and the chosen technical approach.]

## Technical Context
- **Language/Runtime:** [...]
- **Primary dependencies:** [...]
- **Storage:** [...]
- **Target platforms:** [...]
- **Constraints (perf/uptime/UX):** [...]
- **NEEDS CLARIFICATION:** [any remaining unknowns]

## Constitution Check
*Gate: MUST pass before Phase 0. Re-check after Phase 1.*

**Core principles — bind unconditionally.**
- [ ] I — Emergency Flow Is Sacred (NON-NEGOTIABLE)
- [ ] II — Consent Is a Standing Grant, Not a Card Tap (NON-NEGOTIABLE)
- [ ] III — Copy Discipline
- [ ] IV — Demo-Grade, Not Production-Grade — Stated Honestly
- [ ] V — Reset-in-One-Command
- [ ] VI — Data Sovereignty Framing
- [ ] VII — Elderly-First Accessibility
- [ ] VIII — Home Stays Quiet (NON-NEGOTIABLE once a new surface is proposed)
- [ ] IX — One Snapshot, Never a Parallel Template

**Compliance principles — check the binding class before applying.**
`[A]` binds Track A now · `[B→A]` binds Track B, Track A must not foreclose ·
`[B]` Track B only.
- [ ] X — Authorization Is Derived, Never Accepted `[A]`
- [ ] XI — Every PHI Access Leaves a Log `[A]`
- [ ] XII — FHIR Is Pinned `[A]` when any FHIR appears
- [ ] XIII — PHI Does Not Cross a Boundary in Plaintext `[B→A]`
      (Fastlane's public-URL clause binds Track A)
- [ ] XIV — Demo Shortcuts Are the Audit Remediation List `[A]`
- [ ] XV — Consent Artifacts, Retention, and Erasure `[B→A]`

**Compliance checklists** (Development Workflow §6)
- [ ] Does this feature touch PHI, auth, grants, Fastlane, or an ABDM surface?
      → **Yes:** `specs/[###-slug]/checklists/security.md` created from
      `.specify/templates/checklist-security.md` and reviewed against this
      plan. **No:** state why here, and skip.
- [ ] ABDM surface (ABHA, FHIR, consent artifacts, care contexts, HIP/HIU)?
      → also `checklists/abdm.md` from `.specify/templates/checklist-abdm.md`.

Violations (if any) go in **Complexity Tracking** with justification. A `GAP`
on any `[A]` item blocks implementation and is not a Complexity Tracking row.

## Project Structure
```
[directory layout produced by this plan]
```

## Phase 0 — Research (`research.md`)
[Unknowns resolved, decisions + rationale + alternatives considered.]

## Phase 1 — Design
- `data-model.md` — entities, fields, relationships, state transitions.
- `contracts/` — API contracts.
- `quickstart.md` — how to stand it up and run the golden path.

## Phase 2 — Task Planning Approach
[How `tasks.md` will be derived. Do not enumerate tasks here.]

## Complexity Tracking
| Violation | Why needed | Simpler alternative rejected because |
|-----------|-----------|--------------------------------------|
| | | |

## Progress Tracking
- [ ] Phase 0 complete
- [ ] Phase 1 complete
- [ ] Constitution re-check passed
- [ ] Compliance checklist(s) reviewed against this plan — no open `[A]` gaps
- [ ] Ready for `/tasks`
