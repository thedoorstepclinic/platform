# Feature Specification: Records — Capture, Multi-Page Scan & Timeline

**Feature Branch:** `005-records-capture-timeline`
**Created:** 2026-07-16
**Status:** Draft
**Owner:** Adi (dev)
**Depends on:** [`001-tdc-phr-patient-app`](../001-tdc-phr-patient-app/spec.md) —
implements that spec's FR-005/FR-006 (record upload + timeline) at UX
level. Covers screens 3–5 (Profile timeline, Upload, Record detail).

## What this is / is NOT

**IS:** The exact capture flow for "digitize the plastic bag" — camera scan
with automatic edge-detection and multi-page assembly, or importing existing
files — down to what happens on every path, including the platform gap
between Android and web.

**IS NOT:** OCR or any form of content extraction from the document (title,
date, or anything else). **Hard-prohibited regardless of technique**, per
`CLAUDE.md` — even a narrow "just read the date off the page" extraction is
out of scope. Every field is user-entered, always.

---

## Goal

Capture quality that feels like a dedicated scanning app (auto edge-detect,
perspective-correct, multi-page), built entirely from an off-the-shelf
on-device capability — not custom computer-vision work. Zero ambiguity in
the flow: two clearly-labeled entry points, one shared tagging step,
nothing left half-done if the user backs out.

### Primary User Story
As a caregiver, I tap "Add record" on Asha's timeline, choose to scan a
multi-page prescription with my camera, capture each page with the document
auto-detected and cropped for me, tag it once (title, type, date), and see
it appear at the top of her timeline immediately.

### Acceptance Scenarios
1. **Given** a profile timeline, **When** I tap the FAB, **Then** I see
   exactly two entry points: **"Scan with camera"** and **"Choose from
   files."**
2. **Given** "Scan with camera" on Android, **When** I point the camera at a
   document, **Then** its edges are auto-detected and it's auto-cropped and
   perspective-corrected without me drawing a crop box.
3. **Given** a captured page, **When** I tap **"Add another page,"**
   **Then** the camera reopens for the next page; **When** I tap
   **"Done,"** **Then** all captured pages are assembled into one
   multi-page PDF.
4. **Given** "Choose from files," **When** I select multiple images,
   **Then** they combine into one multi-page PDF in selection order
   (reorderable before save); **When** I select a single PDF, **Then** it's
   used as-is with no reassembly step.
5. **Given** either capture path completes, **When** I land on the tagging
   screen, **Then** I set title (required), type (Rx/Lab/Discharge/Other,
   required), and date — **pre-filled to today, always editable** — and
   tap Save.
6. **Given** I tap Save, **When** the save completes, **Then** the record
   appears at the top of the timeline (newest-first) immediately.
7. **Given** a saved record, **When** I open Record detail, **Then** I can
   edit title/type/date, or delete with an explicit confirmation step.
8. **Given** the web build, **When** I tap the FAB, **Then** I see only
   **"Choose from files"** — no live-camera auto-scan button is shown (see
   Platform Scoping below).

### Edge Cases
- User backs out mid-capture (any page, any path) → nothing is persisted;
  no partial or orphan `records` row is created. Only the final Save tap on
  the tagging screen writes anything.
- Edge-detection fails (poor lighting, no clear document boundary) → falls
  back to the scanner library's own manual corner-adjustment UI; this is
  the library's built-in behavior, not something built here.
- User cancels the file picker or selects nothing → returns to the timeline
  unchanged.
- User picks 0 pages after opening the camera flow (backs out before any
  capture) → same as above, nothing persisted.

---

## Platform Scoping (Android vs. web)

Google's ML Kit Document Scanner (the auto edge-detect/crop/multi-page
capability) is **Android-native only** — there is no equivalent
off-the-shelf, "boring" web API for live-camera document scanning. Rather
than fake a degraded version or silently omit the camera button, this is a
deliberate, disclosed platform difference:

| Platform | Entry points |
|---|---|
| Android | "Scan with camera" (ML Kit auto-scan) + "Choose from files" |
| Web | "Choose from files" only |

This is not a golden-path risk — `001`'s demo script performs the live
camera-upload beat on the physical Android demo phone, never on web.

---

## All Records — cross-profile list & search (added 9 Sep 2026)

From a supplied reference. The timeline this spec owns is **per profile** and
stays that way. What it lacked is the surface for the question a caregiver
actually asks: *"where is that prescription?"* — asked about a family, not
about a person.

**Route:** `/records` (app-shell level, not profile-scoped).

| Element | Behaviour |
|---|---|
| **Header** | *"Medical Records — manage and review your family's health documents."* |
| **Search** | Free text over record **title and type**. Debounced, server-side. **Not** over document contents — there is no OCR and no extraction (this spec's hard prohibition). |
| **Filter** | The existing type chips: Rx · Lab · Discharge · Other. |
| **Record card** | Type icon, **type chip**, title, **`Person · date`**, and an action naming what opens (`View Report` / `View Image`). |
| **Empty / loading / error** | `008`'s four states, with an empty-search state naming the recovery action. |

### Scope is the existing derivation, not a new one

Owner decision, 9 Sep 2026: *"if the member is family owner, then he should be
able to — or has been given access to."* So this list shows **the caller's own
profiles ∪ profiles reachable through an unrevoked `caregiver_grants` row** —
the same queryset derivation every other profile-scoped endpoint already uses
(`001` FR-022, Principle X). It introduces **no new authorization concept**: an
account owner sees their family because they hold the grants, and a
grant-holder sees exactly what they were granted.

Consequences that must be built, not assumed:

- **Revoking a grant removes those rows on the very next request.** No cached
  scope, same as everywhere else.
- **Every row names its person.** A record card without attribution is the
  wrong-patient error waiting to happen, and this is the one screen in the app
  where records from different people sit side by side. Person is **not**
  optional metadata here — it is part of the identity of the row.
- **`access_logs` on read**, like every other PHI surface (Principle XI).

**Principle VIII is not in tension here.** VIII prohibits content feeds and
attention-harvesting surfaces — a browse destination for *inventory* the user
does not own. This lists the user's own family's documents, in response to a
search they typed. It has nothing to refresh for, nothing recommended, and no
one else's data in it. The profile-first rule that `012` FR-001 enforces for
*discovery* is about not browsing clinics without a patient in context; it was
never a rule that a caregiver cannot see their own family's files in one place.

---

## Requirements

### Functional Requirements
- **FR-001**: The FAB on Profile timeline MUST present exactly two capture
  entry points on Android ("Scan with camera," "Choose from files") and
  exactly one on web ("Choose from files").
- **FR-002**: "Scan with camera" MUST use on-device automatic edge
  detection and perspective correction via a native document-scanner
  capability (e.g. Google ML Kit Document Scanner) — no hand-built
  computer-vision pipeline.
- **FR-003**: "Scan with camera" MUST support capturing multiple pages in
  one continuous session (add-another-page / done), assembling all
  captured pages into a single multi-page PDF before the tagging step.
- **FR-004**: "Choose from files" MUST allow selecting multiple images
  (combined into one multi-page PDF, page order = selection order,
  reorderable before save) or a single PDF (used as-is, no reassembly).
- **FR-005**: Regardless of capture path, a record MUST resolve to exactly
  one file. No change to `001`'s `records` schema — multi-page capture
  always produces one multi-page PDF, never multiple `records` rows.
- **FR-006**: Both capture paths MUST converge on one shared tagging
  screen: title (free text, required), type (Rx/Lab/Discharge/Other,
  required), date (date picker, defaults to today, always user-editable).
- **FR-007**: The system MUST NOT auto-extract or infer any field
  (including date) from document content. All fields are manually entered;
  the date default is a convenience, not an inference.
- **FR-008**: Nothing is written to `records` until the user explicitly
  taps Save on the tagging screen. Backing out at any prior point MUST
  leave no persisted trace.
- **FR-009**: From Record detail, title/type/date MUST be editable after
  save. Delete MUST require explicit confirmation; delete is permanent
  (no trash/undo) — see Open Decisions; a production undo window is worth revisiting now that deletion is permanent for real users, not demo data.
- **FR-010**: A saved record MUST appear in the profile timeline
  immediately, newest-first (unchanged from `001` FR-005/006).

- **FR-012**: An app-level **All Records** list MUST exist at `/records`,
  MUST derive its queryset from the caller's own profiles ∪ unrevoked
  `caregiver_grants` (Principle X, `001` FR-022), MUST reflect a revoked grant
  on the next request, and MUST write an `access_logs` row on read.
- **FR-013**: Every row in the All Records list MUST name the person the record
  belongs to. Attribution is not optional metadata on this surface — it is what
  prevents the wrong-patient error that a mixed list otherwise invites.
- **FR-014**: Record search MUST operate over **title and type only**. It MUST
  NOT search, index, or extract document contents — no OCR, no content
  extraction, by any technique (this spec's standing prohibition).

### Non-Functional Requirements
- **NFR-001 (On-device capture):** Document detection/cropping happens
  entirely on-device with no server round-trip — capture must not depend
  on network availability (upload of the finished file still requires
  connectivity, same as any other save).

---

## Build Note (not a requirement — flagged for D3 planning)

**Updated 2026-08-15 for the Flutter stack** (was written against Expo RN).
The Android scan capability is a native integration — a Flutter plugin or
platform channel wrapping Google ML Kit's Document Scanner — not a pure-Dart
package. Flutter has no Expo-Go/custom-dev-client split (`flutter run`
always builds against the full native project), so the old dev-client
caveat no longer applies; the only real D3 setup item is confirming the
chosen plugin/channel builds cleanly on the target Android SDK version
before D3 work starts, not discovered mid-sprint.

---

## Open Decisions
| Decision | Owner | Notes |
|---|---|---|
| Max pages per multi-page record | Adi | **Needs a cap.** Uncapped was acceptable when the happy path was the only path; at production an uncapped multi-page scan is a memory and upload-size failure on a low-end device. |
| Exact native scanner library/version | Adi | A Flutter plugin or platform channel wrapping Google ML Kit Document Scanner (e.g. `google_mlkit_document_scanner` or an equivalent) is the leading candidate; confirm during a short D3 spike. |
| Page-reorder interaction (drag vs. up/down buttons) in "Choose from files" multi-image combine | Adi | Implementation detail — doesn't change the behavior contract above. |

---

## Review & Acceptance Checklist

### Content Quality
- [x] Platform gap (Android vs. web) stated explicitly, not glossed over.
- [x] No OCR/extraction anywhere, consistent with `CLAUDE.md` hard-prohibited list.
- [x] All mandatory sections completed.

### Requirement Completeness
- [x] Requirements testable (two vs. one entry point by platform, one-file-per-record invariant, nothing persists before explicit Save).
- [x] Native-module build implication flagged ahead of the sprint it affects.
- [x] Dependencies/assumptions identified (ML Kit Document Scanner availability, target Android SDK version for the plugin/channel).

## Execution Status
- [x] Scenarios defined
- [x] Requirements generated
- [x] Platform scoping resolved and documented
- [x] Review checklist passed
