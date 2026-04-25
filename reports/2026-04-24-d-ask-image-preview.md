# Report: /ask Input — Image Thumbnail Preview in Attachment Chip

**Agent:** Coder-2 | **Date:** 2026-04-24 | **Status:** SHIPPED (ed4236b page.tsx, pushed to main)

---

## Summary

Pasted or dropped images now show a 40x40 thumbnail in the attachment chip instead of filename-only. Click the thumbnail to open a full-screen preview modal. PDFs and DOCX keep filename-only chips.

---

## Changes

### `src/app/ask/page.tsx` (+47 LOC net, single file — Pattern 23)

**State:** Added `previewOpen` boolean for modal toggle.

**Thumbnail in chip (both pre-chat and active-chat locations):**
- When `attachedFile.imageData` exists, renders `<img>` from `data:${mimeType};base64,${imageData}`
- 40x40px, `object-fit: cover`, 4px border-radius, subtle border
- Click opens modal (`setPreviewOpen(true)`)
- Chip border-radius and padding adjust for image vs non-image (8px rounded vs 6px)

**Preview modal:**
- Fixed overlay, `z-index: 200`, `rgba(0,0,0,0.88)` background
- Image rendered at up to 90vw / 85vh with 8px border-radius and box shadow
- Click anywhere on overlay dismisses

**No behavioral changes:** Paste, drop, submit, remove all work identically.

---

## Verification

### Playwright (3/3 passing)

| Test | Result |
|------|--------|
| Image paste shows thumbnail in file chip | PASS |
| PDF drop shows filename-only chip (no thumbnail) | PASS |
| Click thumbnail opens modal, overlay click dismisses | PASS |

### Screenshot

Stitched to `/Users/markymark/Desktop/d-ask-image-preview.png`:
- Chip with 40x40 thumbnail visible
- Full-screen modal with enlarged image
- Modal dismissed, chip still present

### Build

`npm run build` — clean, no warnings.

---

## Scope Notes

- **Multi-file deferred:** Current architecture is single-attachment (`useState` with single object). Multi-file stacking would require array state + append-on-paste behavior, which is its own directive.
- **Pattern 23 verified:** Only `src/app/ask/page.tsx` modified.
- **No behavior changes:** Paste/drop/submit/remove unchanged per non-negotiable.

---

## Commit

`ed4236b` — `feat: /ask image thumbnail preview in attachment chip with click-to-enlarge modal`
