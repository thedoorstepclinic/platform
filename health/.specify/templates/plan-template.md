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

**All fifteen principles bind unconditionally** (constitution v3.0.0 — the
`[A]` / `[B→A]` / `[B]` binding classes were abolished 6 Sep 2026). A
principle may be marked N/A only when this feature genuinely has no such
surface, and the reason must be stated. "For later" is not a reason: there is
no later track.

**Product principles.**
- [ ] I — Emergency Flow Is Sacred (NON-NEGOTIABLE)
- [ ] II — Consent Is a Standing Grant, Not a Card Tap (NON-NEGOTIABLE)
- [ ] III — Copy Discipline
- [ ] IV — Production-Grade, With Demo Data Switchable
- [ ] V — Reset-in-One-Command
- [ ] VI — Data Sovereignty Framing
- [ ] VII — Elderly-First Accessibility
- [ ] VIII — Home Stays Quiet (NON-NEGOTIABLE once a new surface is proposed)
- [ ] IX — One Snapshot, Never a Parallel Template

**Compliance principles — same binding force as the above.**
- [ ] X — Authorization Is Derived, Never Accepted
- [ ] XI — Every PHI Access Leaves a Log
- [ ] XII — FHIR Is Pinned (binds the moment any FHIR appears, mocks included)
- [ ] XIII — PHI Does Not Cross a Boundary in Plaintext
      (includes Fastlane's public-URL clause: HTTPS-only, no PHI in query
      strings or logs, `X-Robots-Tag: noindex`)
- [ ] XIV — Every Demo Path Is Inventoried, and the Inventory Shrinks
- [ ] XV — Consent Artifacts, Retention, and Erasure

**Compliance checklists** (Development Workflow §6)
- [ ] Does this feature touch PHI, auth, grants, Fastlane, or an ABDM surface?
      → **Yes:** `specs/[###-slug]/checklists/security.md` created from
      `.specify/templates/checklist-security.md` and reviewed against this
      plan. **No:** state why here, and skip.
- [ ] ABDM surface (ABHA, FHIR, consent artifacts, care contexts, HIP/HIU)?
      → also `checklists/abdm.md` from `.specify/templates/checklist-abdm.md`.

Violations (if any) go in **Complexity Tracking** with justification. **Any
open compliance `GAP` blocks implementation** and is not a Complexity Tracking
row — there is no longer a non-blocking class of compliance gap.

**Demo-path inventory** (Principle IV/XIV) — required if this feature has any:
- [ ] Every demo path is behind the single `DEMO_MODE` switch, off by default.
- [ ] Each substitutes **data**, never **behaviour** — no skipped
      authorization, no skipped logging, no weakened validation, no state the
      real system cannot produce.
- [ ] Removing `DEMO_MODE` leaves the feature working. If it does not, the
      feature is unfinished.
- [ ] Each `# DEMO-MODE` tag names its real path, and any unbuilt real path
      names **its blocking gate and that gate's owner**.

**External gates** — list any integration this feature needs that is not
reachable yet (ABDM, UHI, HFR/HPR, payment rails), with the gate owner. Build
everything on our side of it; claiming it exists is a Principle VI violation.

| Gate | Blocks | Owner | Ships without it? |
|---|---|---|---|

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
- [ ] Compliance checklist(s) reviewed against this plan — no open gaps
- [ ] Ready for `/tasks`
