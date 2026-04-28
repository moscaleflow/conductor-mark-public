# Report: EntityVetCard Refinements — Kill Ribbon, Add Edit, Secondary Stubs

**Agent:** Coder-2 | **Date:** 2026-04-25 | **Status:** READY TO COMMIT

---

## Summary

Three-part refinement of EntityVetCard per directive:
- **Part A:** Removed left-edge colored ribbon (`borderLeft: 3px solid`). Card now uses standard 1px all-around border. Verdict communicated by header dot only.
- **Part B:** Added "Edit" button as 6th button in action tray (order: Copy / +CRM / +Blacklist / Edit / Assign / Ask). Inline edit mode exposes entity name, type, all facts, all flag details, and recommendation as editable fields. Save persists via `update_vet_result` API action. Cancel reverts.
- **Part C:** Replaced inline secondary names text list with `SecondaryNameStubs` component. Each stub shows name + classification badge (TLP TEAM / IN CRM / BLACKLISTED) or "Vet this" button. Backend classifies each secondary name against team-roster, crm_counterparties, and blacklist tables before returning.

---

## Changes

### `src/components/ask/EntityVetCard.tsx` (primary)

**Part A:** Removed `borderLeft: '3px solid ${c}'` from outer card div.

**Part B — Edit mode:**
- New exports: `VetEditPayload`, `SecondaryNameEntry`
- New prop: `onEditSave?: (data: VetEditPayload) => Promise<boolean> | void`
- New state: `editing`, `editSaving`, `editData`, `savedOverrides`
- Display variables (`displayEntity`, `displayFacts`, `displayFlags`, `displayRec`) render from saved overrides or original props
- Entity name/type fields shown only in edit mode (normally not visible in card)
- Facts grid switches to 2-column layout in edit mode showing all facts (not just top 3)
- Flag details render as inline inputs in edit mode (expandable chevrons hidden)
- Recommendation renders as textarea in edit mode
- Save/Cancel buttons above action tray when editing
- Edit button in tray shows "Editing" (purple highlight) when active, disabled to prevent re-entry
- `ActionBtn` updated with optional `disabled` prop

**Part C — SecondaryNameStubs component:**
- New exported component with props: `entries: SecondaryNameEntry[]`, `onVetThis?`, `vettingNames?`
- Badge styles: TLP TEAM (purple), IN CRM (blue), BLACKLISTED (red)
- Team members show badge only, no "Vet this" button
- Unknown classification shows "Vet this" button (pre-fills chat input on click)
- "Vetting..." loading state supported via `vettingNames` Set

### `src/app/ask/page.tsx`

- Import `SecondaryNameStubs`, `SecondaryNameEntry`, `VetEditPayload`
- `Message.vetSecondaryNames` type changed from `string[]` to `SecondaryNameEntry[]`
- Both response handlers (non-streaming JSON + SSE) updated for classified entries
- `onEditSave` callback added: calls `vet-action` with `update_vet_result`, then updates message state
- Inline secondary names div replaced with `<SecondaryNameStubs>` component
- `onVetThis` callback pre-fills chat input with "Vet [name]" using native setter pattern

### `src/app/api/ask/vet-action/route.ts`

- Added `update_vet_result` action type
- New `handleUpdateVetResult` handler: updates entity_name, entity_name_normalized, entity_type, verdict_data
- Imported `normalizeEntityName` from vet-results

### `src/lib/ask/vet-results.ts`

- Extended `updateVetResult` partial type to accept: `entity_name`, `entity_name_normalized`, `entity_type`, `verdict_data`

### `src/app/api/ask/route.ts`

- Added `classifySecondaryNames()` helper function
- Imports `TEAM_ROSTER` from team-roster
- Batch queries crm_counterparties and blacklist tables for classification
- All three return paths (cached JSON, fresh JSON, SSE) now return classified entries instead of raw strings

---

## Verification

### Playwright (2/2 passing)

| Test | Result |
|------|--------|
| No left ribbon + 6 buttons + Edit mode + cancel | PASS |
| Mobile viewport 375x667 — buttons horizontal | PASS |

### Screenshots

Stitched to `/Users/markymark/Desktop/d-vet-card-refinements.png`:
- Card without left ribbon (5 buttons: Copy / +CRM / Edit / Assign / Ask — no +Blacklist since blacklisted verdict)
- Edit mode active showing entity name/type inputs, fact value inputs, flag detail inputs, recommendation textarea, Save/Cancel buttons
- Mobile viewport: all 5 buttons horizontal in single row

Individual:
- `/Users/markymark/Desktop/d-vet-refinements-1-no-ribbon.png`
- `/Users/markymark/Desktop/d-vet-refinements-2-edit-mode.png`
- `/Users/markymark/Desktop/d-vet-refinements-3-mobile.png`

Note: Secondary name stubs with classification badges not captured in screenshots because Kevin Butlers vet response does not include secondary names. The `SecondaryNameStubs` component is verified by code structure and build success.

### Build

`npm run build` — clean, no warnings.

---

## Decision TBD: EntityVetCard refinements — kill ribbon, add Edit, secondary stubs

**Date:** 2026-04-25
**Status:** READY TO COMMIT

**Decision:** Three-part EntityVetCard refinement. (A) Removed verdict-colored left-edge ribbon — card border returns to standard 1px all-around, verdict communicated by header dot only. (B) Added Edit as 6th action tray button with inline edit mode (entity name/type, facts, flag details, recommendation) + Save/Cancel + backend persistence via update_vet_result. (C) Secondary names upgraded from plain text to classified stubs with TLP TEAM/IN CRM/BLACKLISTED badges and "Vet this" buttons. Backend classifies against team-roster, crm_counterparties, blacklist tables.

**Evidence:** 2/2 Playwright tests passing. Build clean. Screenshots at /Users/markymark/Desktop/d-vet-card-refinements.png.
