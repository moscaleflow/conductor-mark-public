# V4 Shell Refinement: File-Drop Auto-Routing

> Coder-3 Spec | D152 | For Coder-1 execution
> Source: D85 lock — "File drop on any surface auto-routes by type (contract → Contracts drawer, LinkedIn screenshot → Sales lead flow, scam-chat screenshot → blacklist-or-outreach ambient nudge)"
> Build order: 6 of 6 (deferred — highest complexity, most design questions)

---

## User-observable behavior

**Before:** No drag-and-drop on `/operator`. File uploads exist on dedicated pages:
- `/` (landing page): PDF/DOCX/TXT upload → contract analysis (`/api/contract/analyze-proxy` or `/api/demo/analyze`)
- `/reports`: CSV upload → buyer report processing (`/api/buyer-reports/upload`)
- `/api/leads/import`: CSV upload → lead import

**After:** Dropping a file anywhere on the `/operator` page triggers auto-routing:

| File type | Detection | Route to | Action |
|---|---|---|---|
| PDF, DOCX, DOC, TXT | Extension match | Contract analysis | Upload to `/api/contract/analyze-proxy`, then open Milo with: "I just uploaded a contract — analyze it and walk me through the flagged clauses." |
| CSV | Extension match | Buyer report OR lead import (ambiguous) | Show a disambiguation modal: "Is this a buyer settlement report or a lead import?" → route accordingly |
| PNG, JPG, JPEG | Extension match | Milo conversation with image | Attach to Milo chat: "I dropped an image — what is this?" (Milo decides based on content: LinkedIn screenshot → sales, scam chat → blacklist, etc.) |
| Other | Fallback | Milo conversation | Attach to Milo chat: "I dropped a file — what should I do with this?" |

**Drop zone visual:**
1. When a file is dragged over the page, a full-page overlay appears: translucent backdrop + centered "Drop to upload" text with a file icon
2. Drop zone border pulses blue (#0a84ff)
3. On drop: overlay disappears, routing logic fires, toast notification confirms: "Uploading contract..." or "Opening in Milo..."
4. On drag-leave: overlay disappears

**Acceptance criteria:**
- Drag-and-drop works on desktop browsers (Chrome, Safari, Firefox, Edge)
- Full-page drop zone (not a specific target area)
- File type detection by extension, not MIME (MIME is unreliable on drag)
- PDF/DOCX: auto-route to contract analysis (no disambiguation)
- CSV: disambiguation modal (buyer report vs lead import)
- Images: send to Milo conversation with image attachment
- Multiple files: process each sequentially, or reject with "Drop one file at a time"
- Mobile: no drag-and-drop (mobile Safari doesn't support it). No change to mobile UX.
- Error handling: upload failure shows toast error, does not crash the page

---

## File-level scope

| File | Change | ~LOC |
|---|---|---|
| `src/components/operator/FileDropZone.tsx` | **New file.** Full-page drop zone wrapper. Handles `onDragEnter/Over/Leave/Drop`. Renders overlay when dragging. Routes file by extension. | ~100 |
| `src/components/operator/FileDisambiguationModal.tsx` | **New file.** Small modal for CSV disambiguation: "Buyer report" vs "Lead import" buttons. | ~50 |
| `src/app/operator/page.tsx` | Wrap page content in `<FileDropZone>`. Pass routing callbacks for each file type. | ~20 |

**Total: ~170 LOC across 3 files (2 new, 1 modified).**

---

## Data dependencies

**No new schema. No new API routes.** All upload targets already exist:
- `POST /api/contract/analyze-proxy` — accepts FormData with `file` field (PDF/DOCX/TXT)
- `POST /api/buyer-reports/upload` — accepts FormData with `file` field (CSV)
- `POST /api/leads/import` — accepts FormData with `file` field (CSV)
- Image attachment to Milo: via `milo:open-chat-with-message` custom event (would need extension to support file attachments — **see open questions**)

---

## Visual requirements

**No HTML design reference exists.** D144 never landed.

**Assumed drop zone overlay (flagged for Mark):**
```
┌──────────────────────────────────────────────────┐
│                                                  │
│              ┌──────────────────┐                │
│              │   ↓  Drop file   │                │
│              │  to auto-route   │                │
│              └──────────────────┘                │
│                                                  │
└──────────────────────────────────────────────────┘
```
- Full viewport overlay: `position: fixed; inset: 0`
- Background: `rgba(0, 0, 0, 0.7)`
- Center box: `#1c1c1e` background, `2px dashed #0a84ff` border, 16px border-radius
- Text: `#f5f5f7`, 18px, centered
- z-index: above footer (z-20), below PillDrawer (z-50)

**Disambiguation modal:**
- Same dark modal style as other modals in the app
- Two large tap targets: "Buyer settlement report" and "Lead import"
- Cancel option to dismiss

---

## Blast radius

**Medium-high.** Adds a full-page event listener for drag events. Touches the operator page layout wrapper.

**Risk 1:** Full-page `dragenter`/`dragleave` events are notoriously flaky — child elements trigger leave/enter bubbling, causing the overlay to flicker. **Mitigation:** Use a ref counter pattern: increment on `dragenter`, decrement on `dragleave`, show overlay when counter > 0.

**Risk 2:** Image-to-Milo flow requires extending the `milo:open-chat-with-message` custom event to support file attachments. The current event only carries a `message: string`. **Mitigation:** Either extend the event to `{ message: string; files?: File[] }` (changes LayoutShell/ConversationModal) or upload the image first and include a URL reference in the message. Both require changes beyond the operator page.

**Risk 3:** Auto-routing a PDF to contract analysis assumes every PDF is a contract. If someone drops a random PDF, it gets analyzed as a contract. **Mitigation:** The analysis pipeline handles non-contract PDFs gracefully (returns "no relevant clauses found"). Not a crash, just a useless result. In v4.1, Milo could inspect the first page and ask "Is this a contract?"

---

## Build order rationale

Ranked 6 of 6 (last) because:
- Highest implementation complexity of all 6 refinements
- Image-to-Milo flow requires extending the chat event system (cross-component change)
- CSV disambiguation is a UX question that needs Mark's input
- Desktop-only feature (no mobile drag-and-drop)
- The upload workflows already exist on dedicated pages — auto-routing is a convenience, not a capability gap
- Drag-and-drop event handling is brittle and needs careful cross-browser testing

---

## Open questions

**Q1 (blocking):** The `milo:open-chat-with-message` custom event currently only carries a string message. Image drop requires passing a `File` object to the conversation. Does Milo's streaming API (`/api/milo/stream`) support image attachments in the request? If not, this feature requires Milo API changes beyond the operator shell.

**Q2 (assumed answer flagged):** Should "Drop one file at a time" be enforced, or should multi-file drops be supported? **Assumed: single file only for v4.0.** Multi-file adds batch processing complexity.

**Q3 (assumed answer flagged):** D85 mentions "LinkedIn screenshot → Sales lead flow, scam-chat screenshot → blacklist-or-outreach ambient nudge." These require Milo to visually classify the image content. Is this expected to use Claude's vision API? **Assumed: yes** — the image goes to Milo chat, Milo uses vision to interpret and route. The operator page doesn't need to classify images; Milo does.
