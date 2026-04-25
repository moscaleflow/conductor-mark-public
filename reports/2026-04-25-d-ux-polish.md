# Report: /ask UX Polish — Animated Thinking Dots + Paste Card Restyle

**Agent:** Coder-2 | **Date:** 2026-04-25 | **Status:** SHIPPED (co-committed in 9587c37 page.tsx, pushed to main)

---

## Summary

Two UX polish items: (A) Static "..." thinking indicator replaced with animated pulsing purple dots matching the /ask accent palette. (B) Paste chips restyled from horizontal pills to 104x104 square cards with "PASTED" badge, text preview with fade mask, and char count footer — matching Claude.ai attachment card pattern and consistent with image thumbnail chips.

---

## Changes

### `src/app/ask/page.tsx` (single file — Pattern 23)

**A. Animated Thinking Dots**

CSS keyframes added (~line 720):
- `@keyframes thinkingDot` — 1.4s cycle, opacity oscillates 0.3→1.0 at 40% keyframe
- Three 6px purple circles (`rgba(139,92,246,0.8)`) with 0.2s staggered delays
- Replaces static `"..."` text in empty assistant message during streaming

Rendering (~line 1331):
- Inline-flex container with 3 animated dot `<span>` elements
- Renders only when `streaming && i === messages.length - 1 && !msg.content`

**B. Paste Card Restyle**

Card design (applied in 3 locations — pre-chat input, active-chat input, ExpandableText collapsed state):
- 104x104px fixed dimensions, dark background (`rgba(14,14,22,0.95)`)
- Purple border (`rgba(139,92,246,0.3)`), 8px border-radius
- "PASTED" badge: 9px uppercase, cyan accent (`rgba(103,194,222,0.9)`)
- Text preview: first 200 chars, 10px font, `maskImage` linear-gradient fade-out
- Char count footer: bottom-aligned, 9px, muted text
- X button: top-right, stops propagation to prevent modal open
- Click card body: opens full text preview modal

---

## Verification

### Playwright (6/6 passing)

| Test | Result |
|------|--------|
| Animated thinking dots render during streaming | PASS |
| Paste card renders at 104x104 with PASTED badge | PASS |
| Paste card click opens text modal | PASS |
| X button removes paste card | PASS |
| Image chip + paste card coexist | PASS |
| Mobile viewport paste card sizing | PASS |

### Screenshots

Stitched to `/Users/markymark/Desktop/d-ux-polish-stitched.png`:
- Animated thinking dots (purple pulsing)
- 104x104 paste card with PASTED badge + text preview
- Modal open from paste card click
- Mixed image + paste card side by side
- Mobile viewport

Individual screenshots also at:
- `/Users/markymark/Desktop/d-thinking-dots.png`
- `/Users/markymark/Desktop/d-paste-card.png`
- `/Users/markymark/Desktop/d-paste-card-modal.png`
- `/Users/markymark/Desktop/d-paste-card-mixed.png`
- `/Users/markymark/Desktop/d-paste-card-mobile.png`

### Build

`npm run build` — clean, no warnings.

---

## Co-commit Note

Commit `9587c37` includes Coder-1's concurrent vet cards rendering fix (clear prose on card arrival, skip dedup for multi-entity). UX polish code is additive and does not interact with vet card rendering.

---

## Commit

`9587c37` — co-committed with Coder-1's `fix: vet cards rendering — clear prose on card arrival, skip dedup for multi-entity`
