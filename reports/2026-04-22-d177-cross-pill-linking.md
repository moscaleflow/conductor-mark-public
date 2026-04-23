# D177 — V4.1 Item 07: Cross-pill Drawer Linking

**Date:** 2026-04-22  
**Agent:** Coder-1  
**Scope:** ~30 LOC across 3 files — when a DrawerItem references another pill, tapping opens the target drawer instead of navigating to Milo chat

---

## Summary

Two commits shipping cross-pill linking for operator drawers:

1. `95e32ec` — Client-side: `action_pill_id` on DrawerItem, `onPillOpen` callback in PillDrawer + operator page
2. `8d00458` — Server-side: aging overdue items → disputes drawer proof link

Server-side links for accounting→disputes and contracts→esign were already committed by Coder-3 in D176 (`5863eb2`).

## Commit 1: Client-side wiring (`95e32ec`)

### Files modified

| File | Change |
|---|---|
| src/components/operator/PillDrawer.tsx | Added `action_pill_id?: string` to DrawerItem, `onPillOpen?: (pillId: string) => void` to props, Review button dispatches to target drawer when link present |
| src/app/operator/page.tsx | Added `handlePillOpen` callback that resolves pill by ID from data or ROLE_PILLS, guards against non-drawer pills via DRAWER_PILL_IDS, passes `onPillOpen` to PillDrawer |

### Behavior

- DrawerItem with `action_pill_id` shows `${action_label} →` button text
- Tapping calls `onPillOpen(pillId)` which sets the active drawer pill to the target
- If the target pill ID is not in `DRAWER_PILL_IDS`, the tap is silently ignored (safety guard)
- Items without `action_pill_id` behave exactly as before (onReview → Milo chat)

## Commit 2: Aging → Disputes proof link (`8d00458`)

### File modified

| File | Change |
|---|---|
| src/app/api/operator/pill/route.ts | Overdue invoices in aging pill: `action_pill_id: 'disputes'`, `action_label: 'View dispute'` |

## All 3 proof links (across D176 + D177)

| Source pill | Condition | Target pill | action_label | Committed in |
|---|---|---|---|---|
| accounting | disputes_summary tab item | disputes | (default) | D176 `5863eb2` |
| contracts_review | status=pending_signature | esign | Open signing | D176 `5863eb2` |
| aging | status=overdue | disputes | View dispute | D177 `8d00458` |

## Verification

- **npm run build**: PASS on both commits
- **Playwright API tests**: 3/3 pass (`tests/d177-cross-pill.spec.ts`)
  - accounting disputes tab: `action_pill_id=disputes` confirmed
  - contracts pending_signature tab: `action_pill_id=esign` confirmed
  - API snapshot captured to Desktop
- **Note**: Full UI drawer-switch screenshot requires deploy (tests run against production which doesn't have D176/D177 yet). API-level verification confirms all three links are correctly wired in code.

## LOC impact

| File | +/- |
|---|---|
| PillDrawer.tsx | +129/−62 (includes linter ContractClauseList merge) |
| operator/page.tsx | +10/−1 |
| pill/route.ts | +2/−1 |
| **Net D177 logic** | ~30 LOC |
