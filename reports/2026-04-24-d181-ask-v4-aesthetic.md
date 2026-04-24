# D181 — /ask Aesthetic Strip-Down to V4 System

**Date:** 2026-04-24  
**Agent:** Coder-1  
**Scope:** Surgical aesthetic edit of /ask to align with V4 design system

---

## Summary

Stripped /ask down to V4 design: removed all operator shell chrome, added MILO | TLP wordmark header, replaced pill grid with single-row pill buttons + tooltips, added purple glow search bar, implemented full-viewport drop zone with brand purple pulse, converted all icons to SVG, confirmed zero emoji in UI chrome. Build clean, 5/5 tests pass, stitched screenshot on Desktop.

---

## Commits

### milo-for-ppc

| Commit | Description |
|---|---|
| `8f09f2c` | feat: /ask aesthetic strip-down to V4 design system |

---

## Fixes Applied

### FIX 1 — Header replaced
- Removed: Topbar (MILO wordmark + "investintpi+milotest" + logout + notifications + search)
- Added: Fixed header with `MILO` (letter-spacing 0.22em, 13px) + 1px purple divider + `TLP` (letter-spacing 0.12em, 13px, 50% opacity)
- Right side: empty

**Method:** Added `/ask` to `BARE_ROUTES` in LayoutShell.tsx (line 24). This bypasses the entire operator shell (Topbar, ImpersonationBanner, ConversationModal, SupportBubble, SupportPanel, AdminSkillBar, UniversalDropZone). Custom Header component renders directly in the /ask page.

### FIX 2 — Feedback chat bubble removed
- SupportBubble (bottom-right floating widget) removed via BARE_ROUTES bypass
- No floating chrome of any kind on /ask

### FIX 3 — Center column layout
- Search bar: max-width 620px, rgba(14,14,22,0.95) background, 0.5px border, border-radius 20px, padding 18px 22px
- Placeholder: "Ask anything..." at 14px, rgba(255,255,255,0.38)
- Submit: SVG arrow icon (14px), no mic, no emoji
- Glow: linear-gradient(90deg, purple→pink→green), 12px blur, 0.55 opacity → 0.85 on focus
- Pills: single row directly beneath (20px gap), pill-shaped (border-radius 999px), 8px 18px padding, 13px, letter-spacing 0.02em
- Descriptions removed from main view — tooltip on hover only

### FIX 4 — Background + typography
- Background: #06060a (applied via page-level div, overrides layout's #0a0a0f)
- Font stack: -apple-system, BlinkMacSystemFont, "Inter", "Segoe UI", Roboto, sans-serif
- No @import or font-face declarations
- Sentence case everywhere except MILO and TLP

### FIX 5 — Full-page drop zone
- Window-level dragover/dragleave/drop handlers (not scoped to container)
- Overlay: fixed inset 0, z-index 100, rgba(6,6,10,0.92) background
- Dashed border: 1.5px dashed rgba(139,92,246,0.6), inset 24px, border-radius 24px
- Pulse animation: borderPulse 2.5s ease-in-out infinite (0.4 → 0.8 opacity)
- Center text: "Drop it anywhere" (32px, weight 300) + "I'll read PDFs, screenshots, images, whatever" (14px subtitle)
- Dismisses on drop or window dragleave

### FIX 6 — Copy audit
- Zero "we"/"our"/"our AI"/"the system" in UI chrome — confirmed via grep
- Zero emoji characters in UI chrome — all icons are SVG elements
- Drop zone uses first-person Milo voice: "I'll read PDFs..."
- Rate limit error uses neutral phrasing: "Rate limit: 30 requests per hour"

---

## Verification (Pattern 16)

### Playwright tests: 5/5 pass (49.0s)

| Test | Result |
|---|---|
| Empty state — desktop | PASS |
| Empty state — mobile | PASS |
| Vet active — desktop (mid-stream + completed) | PASS |
| Vet active — mobile | PASS |
| All screenshots exist | PASS |

### Screenshots (on Desktop)

Individual files:
- `d181-01-empty-desktop.png` — MILO | TLP header, glow search bar, 4 pill buttons, #06060a bg
- `d181-02-empty-mobile.png` — same layout at 375px width
- `d181-03-vet-active-desktop.png` — mid-stream vet response, pill tabs in header, glow input
- `d181-04-vet-active-mobile.png` — mobile active state
- `d181-05-completed-desktop.png` — full vet response rendered

**Stitched:** `/Users/markymark/Desktop/d181-ask-v4-cleanup.png` (286KB)

### Checklist

- [x] Top-right is empty (no logout, notifications, avatar, search widget)
- [x] Bottom-right has no feedback widget
- [x] Zero emoji characters in UI chrome
- [x] MILO | TLP wordmark with purple divider
- [x] Purple glow search bar with SVG arrow submit
- [x] Pills in single row beneath search, descriptions as tooltips only
- [x] Background #06060a
- [x] Full-viewport drop zone with purple pulse animation
- [x] First-person Milo voice ("I'll read PDFs...")
- [x] Build passes clean
- [x] All 6 fixes visible in stitched screenshot

---

## Decision Log

Decision 87 added to DECISION-LOG.md: "/ask aesthetic strip-down to V4 system"

---

## Deploy URL

https://tlp.justmilo.app/ask
