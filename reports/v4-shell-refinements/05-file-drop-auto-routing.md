# V4 Shell Refinement: File-Drop Auto-Routing

> Coder-3 Spec | D152 (updated D161) | For Coder-1 execution
> Source: D85 lock — "File drop on any surface auto-routes by type (contract → Contracts drawer, LinkedIn screenshot → Sales lead flow, scam-chat screenshot → blacklist-or-outreach ambient nudge)"
> Build order: **ALREADY SHIPPED** — UniversalDropZone already does this globally

---

## CRITICAL FINDING: This refinement already exists

`src/components/UniversalDropZone.tsx` (985 lines) is a fully built file-drop auto-router that:

1. **Wraps the entire app** via `LayoutShell.tsx` (lines 99-159) — every page including `/operator` already has file drop
2. **Handles 8 file categories** with smart detection:

| Category | Detection | Route |
|---|---|---|
| `IMAGE` | Extension: .png, .jpg, .jpeg, .gif, .webp | `/api/capture` — OCR via `callClaudeVision`, entity extraction, scam screening, pipeline add |
| `LEAD_CSV` | CSV headers: "name" + "email"/"company" | `/api/leads/import` — match existing, create new, flag dupes |
| `BANK_CSV` | CSV headers: "date" + "description" + "amount"/"balance" | `/api/bank-transactions/import` — auto-match to invoices |
| `BUYER_REPORT` | Extension: .xlsx, .xls, .xlsm OR CSV with call/CID/disposition headers | `/api/buyer-reports/upload` — reconciliation engine |
| `CONTRACT` | PDF/DOCX with contract keywords (msa, agreement, io, w9, nda, redline, rider, addendum) | `/api/contract/process` → base64 upload → analysis → opens Milo chat with document ID |
| `PUBLISHER_INVOICE` | PDF with invoice/statement/bill keywords | `/api/buyer-reports/upload` — cross-check against call records |
| `TRANSFER_SCRIPT` | .txt/.doc/.docx with script/transfer/creative/ivr/pitch keywords | `/api/creative/analyze` — compliance scoring + qualifier matching |
| `ASK` | Fallback for unrecognized files | Opens Milo chat: "I dropped a file: {name}. What should I do?" |

3. **CSV disambiguation is automatic** — inspects first 4KB of CSV headers to distinguish lead CSVs, bank CSVs, and buyer reports. No manual disambiguation modal needed.

4. **Image handling is fully built** — images go to `/api/capture` which calls `callClaudeVision` for OCR, then extracts companies, runs scam screening, adds to pipeline, and dispatches `milo:entity-open` events. This is exactly D85's "LinkedIn screenshot → Sales lead flow, scam-chat screenshot → blacklist-or-outreach ambient nudge."

5. **Full-page overlay exists** — drag enter shows a backdrop-blurred overlay with file-type-specific messaging and icons. Processing state shows spinner. Success/error shows toast with "View" link.

6. **Clipboard paste supported** — pasting images triggers the same flow. Pasting text that looks like a transfer script triggers script analysis.

---

## What D85 requested vs what exists

| D85 requirement | Status |
|---|---|
| Drop PDF anywhere on /operator → contract analyzer | **DONE** — `CONTRACT` category, keyword detection, `/api/contract/process` |
| Drop CSV → appropriate importer | **DONE** — 3-way auto-disambiguation via header inspection |
| LinkedIn screenshot → Sales lead flow | **DONE** — `IMAGE` category → `/api/capture` → entity extraction → pipeline add |
| Scam-chat screenshot → blacklist-or-outreach | **DONE** — `/api/capture` runs `screenEntity` for blacklist check + scam screening |
| File drop on any surface | **DONE** — `LayoutShell` wraps all pages with `UniversalDropZone` |

---

## Q1 Resolution: Does Milo streaming accept image attachments?

**Answer: No, but it doesn't need to.**

`/api/milo/stream` accepts `{ messages: UIMessage[] }` as JSON (line 100-101). UIMessage parts are filtered to `type: 'text'` only (line 120-122). No image parts, no FormData, no base64 image support in the streaming endpoint.

**This is not a gap** because images are handled by a separate pipeline:
- `UniversalDropZone.processImage()` → `/api/capture` (line 564) → `callClaudeVision` (OCR) → entity extraction → scam screening → pipeline add → `milo:entity-open` event
- The image never needs to reach Milo's streaming endpoint. Claude Vision does the image understanding; Milo chat gets a text message about the extracted entity.

**If image-in-Milo-chat were ever needed** (e.g., "Milo, what is this screenshot?"), the changes would be:
1. Extend `UIMessage` parts to include `{ type: 'image'; image: string }` (~5 LOC in stream route)
2. Pass image parts to `streamMiloResponse` which forwards them to Claude as image content blocks (~10 LOC in milo-agent)
3. Extend `sendMessage` in `ConversationModal` to accept `File` alongside text (~15 LOC)
**Total: ~30 LOC across 3 files.** But this is not needed for D85 scope — the capture pipeline is the correct architecture for screenshots.

---

## Remaining work (if any)

**The only possible gap:** D85 mentions "file drop auto-routing" as a V4 shell refinement. Since `UniversalDropZone` already does this globally, spec 05 should be **marked as ALREADY SHIPPED** and removed from Coder-1's backlog.

**Potential v4.1 enhancements (not in scope):**
- Add `disputes` category for dispute evidence documents
- Add `timesheet` category for VA time tracker imports
- Improve PDF disambiguation (currently PDF falls to `ASK` if no keyword match — could use first-page Vision inspection)
- Add multi-file drop support (currently takes `files[0]` only, line 375)

---

## Build order update

**Removed from build order.** Refinement 05 is complete. The updated build order for remaining refinements is:

1. 06: Decommission /dashboard-v2
2. 02: Footer sync status
3. 04: Curated picker
4. 03: Long-press jiggle
5. 01: Mic placement
