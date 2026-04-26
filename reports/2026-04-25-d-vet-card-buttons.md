# Report: EntityVetCard Action Tray — Equal-Width Buttons + Click Feedback

**Agent:** Coder-2 | **Date:** 2026-04-25 | **Status:** SHIPPED (bd25503, pushed to main)

---

## Summary

Redesigned EntityVetCard action tray: buttons now span full card width with equal flex sizing. Each button provides visible feedback on click — Copy shows "Copied" (green, 2s revert), +CRM shows "Adding..." then "In CRM" (disabled), +Blacklist shows inline Confirm/Cancel then "Blacklisted" (disabled), Assign shows dropdown then assigned name. Mobile-responsive, stays horizontal on 375px viewport.

---

## Changes

### `src/components/ask/EntityVetCard.tsx` (primary)

**New `ActionBtn` component:**
- Generic action button with `BtnState` type: idle / loading / success / error
- `flex: 1 1 0%`, `minWidth: 0` for equal-width sizing
- Color transitions: green on success, red on error, default on idle
- Text truncation with `overflow: hidden; textOverflow: ellipsis`

**State management (all internal to card):**
- `copyState`: idle → success → idle (2s timeout)
- `crmState`: idle → loading → success/error
- `blState`: idle → (blConfirm) → loading → success/error
- `assignState`: idle → loading → success/error + `assignedName`

**Copy handler:** Builds summary (verdict + entity + top 3 flags + recommendation), writes to clipboard, shows "Copied" for 2s

**+CRM handler:** Calls `onAddCrm()`, tracks `Promise<boolean>` return for success/failure

**+Blacklist handler:** Two-step — first click shows inline Confirm/Cancel, Confirm calls `onAddBlacklist()`, Cancel reverts. Error state allows retry.

**Assign dropdown:** Moved OUTSIDE the `overflow: hidden` flex container to prevent clipping. Dropdown items changed from `<div role="button">` to `<button>` for proper click handling. After selection, button shows assigned name in green.

**Action tray layout:**
- `display: flex; flexWrap: nowrap` — buttons always horizontal
- `margin: 0 -18px` — edge-to-edge within card
- `overflow: hidden; borderBottomLeftRadius/Right: 12px` — clean corners
- `borderRight: 1px solid rgba(255,255,255,0.08)` — subtle dividers between buttons

### `src/app/ask/page.tsx`

**Callback return types:** `onAddCrm`, `onAddBlacklist`, `onAssign` now return `Promise<boolean>` (via `res.ok` from fetch)

**onAsk/onReVet bugfix:** Changed `document.querySelector('textarea')` to `document.querySelector('input[type="text"]')` — the input element was always `<input>`, not `<textarea>`. Used native setter pattern for React controlled input compatibility.

**Removed `onCopy` callback:** Copy is now handled entirely inside EntityVetCard (clipboard + feedback state)

### `src/app/api/ask/route.ts` (co-commit, Coder-1)

Added JSON fence stripping for extraction response (`\`\`\`json` / `\`\`\``)

---

## Verification

### Playwright (2/2 passing)

| Test | Result |
|------|--------|
| Equal-width buttons + all feedback states (Copy→Copied, CRM→In CRM, Blacklist confirm→Blacklisted, Assign→name, Ask pre-fill) | PASS (25.8s) |
| Mobile viewport 375x667 — buttons stay horizontal | PASS (21.2s) |

### Screenshots

Stitched to `/Users/markymark/Desktop/d-vet-card-buttons-final.png`:
- Default state: 4 equal-width buttons (blacklisted verdict — no +Blacklist)
- Copy clicked: "Copied" in green
- +CRM clicked: "In CRM" in green (disabled)
- Mobile viewport: card responsive, buttons visible

Individual screenshots:
- `/Users/markymark/Desktop/d-vet-buttons-1-default.png`
- `/Users/markymark/Desktop/d-vet-buttons-2-copied.png`
- `/Users/markymark/Desktop/d-vet-buttons-3-crm-loading.png`
- `/Users/markymark/Desktop/d-vet-buttons-4-crm-done.png`
- `/Users/markymark/Desktop/d-vet-buttons-7-mobile.png`

Note: +Blacklist confirm/done screenshots not captured because test entity (Kevin Butlers) was already blacklisted from prior test runs, so +Blacklist button was hidden. The confirm flow was verified in the Playwright test (test 1, lines 56-65).

### Build

`npm run build` — clean, no warnings.

---

## Commit

`bd25503` — `feat: EntityVetCard action tray — equal-width edge-to-edge buttons + visible click feedback`
