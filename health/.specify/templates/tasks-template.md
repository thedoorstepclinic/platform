# Tasks: [FEATURE NAME]

**Input:** Design docs from `specs/[###-feature-slug]/`
**Prerequisites:** plan.md (required), research.md, data-model.md, contracts/

## Conventions
- **[P]** = can run in parallel (different files, no dependency).
- Each task names exact file paths.
- Tasks are ordered by the spec's build order; the emergency flow is never
  deprioritised below P1 polish.

## Phase Layout
- Setup → Data/Contracts → Core features (P0) → Integration → P1 → Polish.

## Tasks
- [ ] T001 [Setup] ...
- [ ] T002 [P] [Data] ...

## Dependency Notes
[Which tasks block which.]

## Parallel Execution Example
[Commands or groupings that can run together.]
