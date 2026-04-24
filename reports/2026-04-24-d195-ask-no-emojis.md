# D195 — Strip Emojis from Pill System Prompts

**Date:** 2026-04-24  
**Agent:** Coder-1  
**Scope:** System prompt edits to 4 pill files + Playwright verification

---

## Summary

Added explicit no-emoji instructions to all 4 pill system prompts. Locked 🚩 and ✅ into the Vet pill's RED FLAGS / GREEN FLAGS output format headers as the only allowed emoji markers. All other emojis prohibited.

---

## Commits

### milo-for-ppc

| Commit | Description |
|---|---|
| `97052d0` | fix: strip emojis from pill system prompts — keep Vet 🚩/✅ markers |
| `e7758a3` | test: D195 Playwright — emoji strip verification across all 4 pills |

---

## Changes

### buyer.ts
Added: `Never use emojis in your responses. Not in headers, not in bullet points, not anywhere. Plain text and markdown only.`

### sales.ts
Added: same instruction

### publisher.ts
Added: same instruction

### vet.ts
- Changed `Header: "RED FLAGS"` → `Header: "RED FLAGS 🚩"`
- Changed `Header: "GREEN FLAGS"` → `Header: "GREEN FLAGS ✅"`
- Added: `Never use emojis in your responses except for 🚩 in the RED FLAGS header and ✅ in the GREEN FLAGS header. Those two markers are part of the locked output format. No other emojis anywhere.`

### Classifier prompt (route.ts)
Audited — already emoji-free, no changes needed.

---

## Playwright Results

5/5 passing (1.3m)

| Test | Result | Detail |
|---|---|---|
| Buyer — no emojis | PASS | 0 emojis found |
| Sales — no emojis | PASS | 0 emojis found |
| Publisher — no emojis | PASS | 0 emojis found |
| Vet — 🚩✅ present, no others | PASS | 2 allowed emojis, 0 other |
| Verify screenshots | PASS | 4/4 exist |

---

## Visual Verification

Stitched screenshot: `/Users/markymark/Desktop/d195-ask-no-emojis.png` (1280x3260)

| Panel | Content |
|---|---|
| d195-01 | Buyer response — clean, no emoji |
| d195-02 | Sales response — clean, no emoji |
| d195-03 | Publisher response — clean, no emoji |
| d195-04 | Vet response — 🚩 RED FLAGS and ✅ GREEN FLAGS present |

---

## Decision

Logged as **Decision 96** in DECISION-LOG.md.
