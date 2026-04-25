# Clipboard Paste Report: /ask Input Accepts Image Paste

**Agent:** Coder-2 | **Date:** 2026-04-24 | **Status:** SHIPPED (10f6e8d page.tsx + dbbe210 route.ts, pushed to main)

---

## Summary

Copy image from Teams/Slack/screenshot → Cmd+V into /ask → image attaches → Milo sees it via Claude vision. Text paste unchanged.

---

## Changes

### 1. `src/app/ask/page.tsx`

**Paste handler** (useEffect on window 'paste'):
- Iterates `e.clipboardData.items`
- Finds first `image/*` MIME type
- Calls `e.preventDefault()` to suppress browser default
- Calls `handleFileContent(file)` — same function as drag-drop

**Image file handling** (extended `handleFileContent`):
- `isImage = file.type.startsWith('image/')`
- Image size limit: 10MB (vs 5MB contracts, 100KB text)
- Binary → base64 encoding (same ArrayBuffer path as contracts)
- Stored in `attachedFile.imageData` + `attachedFile.mimeType`

**Submit flow** (extended `doSubmit`):
- Sends `imageFile: { name, data, mimeType }` in fetch body when image attached

**Note:** Frontend changes landed in Coder-1's commit 10f6e8d (D200 Part 1.5) because they committed while paste changes were in the working tree.

### 2. `src/app/api/ask/route.ts` (+11 LOC)

**Body parsing:** Added `imageFile?: { name: string; data: string; mimeType: string }` to body type.

**Vision content blocks:** When `imageFile` present, constructs messages array with image + text content blocks:
```
[{ type: 'image', source: { type: 'base64', media_type, data } },
 { type: 'text', text: userMessage }]
```

Passed to `callClaudeStream` via `messages` option (type assertion needed — `CallClaudeOptions.messages` types `content` as `string`, but SDK accepts content blocks).

---

## Verification

### Playwright (2/2 passing)

| Test | Result |
|------|--------|
| Image paste creates file badge on Vet pill | PASS |
| Text paste/type still works normally | PASS |

### Screenshots

Stitched to `/Users/markymark/Desktop/d-ask-paste-image.png`:
- Before: Vet pill selected, input empty
- After: "screenshot.png" file badge visible, Milo responding

---

## Scope Deviation

Directive specified "Pattern 23: file scope is the input component file only." Implementation required 2 files because the directive also requires "Milo responds" to pasted images — that requires backend vision content blocks in route.ts. Without the backend change, images attach visually but Claude can't see them.

---

## Edge Cases Handled

| Case | Behavior |
|------|----------|
| Text-only paste | Normal text paste — handler skips, no preventDefault |
| Image paste | Image captured, base64 encoded, attached |
| Mixed (text + image) | Image wins — first image/* item found, preventDefault stops text |
| Large image (>10MB) | Error: "File too large (max 10MB)" |
| Vet pill + image paste | Auto-submits (same as contract drag-drop) |
| Multiple pastes | Second paste replaces first attachment |
