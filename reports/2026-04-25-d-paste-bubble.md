# Report: /ask Long-Paste Collapse to Chip

**Agent:** Coder-2 | **Date:** 2026-04-25 | **Status:** SHIPPED (b2162a0 page.tsx, pushed to main)

---

## Summary

Text paste >500 chars now collapses to a "Pasted text (N chars)" chip instead of flooding the input box. Clipboard icon distinguishes from file/image chips. Click chip to expand full text in scrollable modal. X to remove. Multiple pastes stack independently.

---

## Changes

### `src/app/ask/page.tsx` (single file — Pattern 23)

**New state:**
- `pastedTexts: Array<{ id: string; content: string }>` — stackable paste chips
- `pastedTextPreview: string | null` — id of paste shown in modal

**New constant:** `CLIPBOARD_SVG` — small clipboard icon for paste chips

**Paste handler extended** (useEffect on window 'paste'):
- After image check (unchanged), checks `e.clipboardData.getData('text/plain')`
- If text.length > 500: `preventDefault()`, adds to `pastedTexts` array with `Date.now()` id
- If text.length <= 500: falls through to browser default (inline insert)
- Image paste behavior completely unchanged

**Chip rendering** (both pre-chat and active-chat locations):
- `pastedTexts.map()` renders chips alongside file/image chip
- Each chip: clipboard icon + "Pasted text (N chars)" label + X button
- Click label opens modal, X removes that specific paste
- `marginRight: 6` for horizontal stacking

**Text preview modal:**
- Fixed overlay, z-index 200, `rgba(0,0,0,0.88)` backdrop
- Inner container: dark panel, 80vw/80vh max, scrollable, 12px border-radius
- `<pre>` with `white-space: pre-wrap` preserves formatting
- Click inner panel: no-op (stopPropagation). Click backdrop: dismisses.

**Submit flow extended:**
- `handleSubmit` combines: `input.trim()` + all `pastedTexts` content (joined with `\n\n`)
- Submit button enabled when pastedTexts exist (even without typed input)
- `doSubmit` clears `pastedTexts` state alongside `attachedFile`

**Reset flow:** `handleReset` clears `pastedTexts` and `pastedTextPreview`

---

## Verification

### Playwright (6/6 passing)

| Test | Result |
|------|--------|
| Short text paste (<500 chars) stays inline | PASS |
| Long text paste (>500 chars) collapses to chip | PASS |
| Click chip opens text modal | PASS |
| X button removes chip | PASS |
| Multiple pastes stack | PASS |
| Image + pasted text coexist | PASS |

### Screenshot

Stitched to `/Users/markymark/Desktop/d-paste-bubble.png`:
- Chip with clipboard icon and char count
- Modal open with full pasted text
- Multiple chips stacked
- Image chip + text chip side by side

### Build

`npm run build` — clean, no warnings.

---

## Co-commit Note

Commit `b2162a0` includes Coder-1's concurrent EntityVetCard changes in page.tsx (same pattern as clipboard paste co-commit with D200 Part 1.5). All paste-bubble code is additive and does not interact with vet card rendering.

---

## Commit

`b2162a0` — `feat: /ask long-paste collapse — text >500 chars becomes "Pasted text" chip with click-to-expand modal`
