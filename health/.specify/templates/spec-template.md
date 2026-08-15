# Feature Specification: [FEATURE NAME]

**Feature Branch:** `[###-feature-slug]`
**Created:** [DATE]
**Status:** Draft
**Input:** [one-line description of the feature request]

## Execution Flow (spec authoring)
```
1. Parse the feature request → identify actors, actions, data, constraints.
2. Mark every ambiguity with [NEEDS CLARIFICATION: question].
3. Fill User Scenarios (must have at least one primary story).
4. Write Functional Requirements — each MUST be testable.
5. Identify Key Entities (if data is involved).
6. Run the Review & Acceptance Checklist.
7. If any [NEEDS CLARIFICATION] remain → Status stays Draft.
```

---

## ⚡ Quick Guidelines
- ✅ Focus on WHAT users need and WHY.
- ❌ Avoid HOW to build it (no stack, no schema, no endpoints) unless the
  request is itself infrastructural.
- 👥 Written for stakeholders, not just engineers.

---

## User Scenarios & Testing *(mandatory)*

### Primary User Story
[Plain-language narrative of the main journey.]

### Acceptance Scenarios
1. **Given** [state], **When** [action], **Then** [outcome].

### Edge Cases
- What happens when [boundary / failure condition]?

## Requirements *(mandatory)*

### Functional Requirements
- **FR-001**: The system MUST [capability].

### Key Entities *(include if data is involved)*
- **[Entity]**: [what it represents, key attributes, relationships].

---

## Review & Acceptance Checklist

### Content Quality
- [ ] No unnecessary implementation detail.
- [ ] Focused on user value and business need.
- [ ] Written for non-technical stakeholders.
- [ ] All mandatory sections completed.

### Requirement Completeness
- [ ] No [NEEDS CLARIFICATION] markers remain.
- [ ] Requirements are testable and unambiguous.
- [ ] Success criteria are measurable.
- [ ] Scope is clearly bounded.
- [ ] Dependencies and assumptions identified.

---

## Execution Status
- [ ] User description parsed
- [ ] Key concepts extracted
- [ ] Ambiguities marked
- [ ] Scenarios defined
- [ ] Requirements generated
- [ ] Entities identified
- [ ] Review checklist passed
