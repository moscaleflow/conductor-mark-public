# V4 Shell Refinements: Visual Assumptions — Approve or Redline

> Coder-3 | D161 → D164a | 2026-04-22
> No HTML design reference exists (D144 never landed).
> This is the minimum-viable design spec. Coder-1 builds to whatever Mark approves.
>
> **How to review:** Scan each numbered item. If it's fine, leave the checkbox empty.
> If something needs to change, check the box and write the correction.
> If everything is fine: "approve all." Expected review time: 5 minutes.

---

## 1. Mic Icon (Spec 01 — SearchBar)

**What it looks like:**
```
┌──────────────────────────────────────────────────────────┐
│ Ask Milo anything…  e.g. "Show me flagged calls"    🎤  │
└──────────────────────────────────────────────────────────┘
                                                       ↑
                                              18px outline mic
                                              #636366 idle
                                              #ff453a when listening
```

- [ ] **1a. Icon style:** SF Symbols-style outline mic, 1.5px stroke, 18px, inline SVG. No icon library.
  - Change to: ___

- [ ] **1b. Colors:** Idle `#636366` (gray) → Listening `#ff453a` (red) → Unavailable `#48484a` (dark gray, hidden if browser lacks Web Speech API)
  - Change to: ___

- [ ] **1c. Position:** Right-aligned inside input, 14px from right edge. Input gets 44px right padding.
  - Change to: ___

- [ ] **1d. Listening animation:** Opacity pulse (1 → 0.4 → 1) over 1s, infinite, ease-in-out.
  - Change to: ___

- [ ] **1e. Behavior:** Tap starts listening. Speech end (1.5s silence) auto-submits to Milo. Tap again cancels.
  - Change to: ___

---

## 2. Footer Sync Status (Spec 02 — new component)

**What it looks like (3 states):**
```
Healthy:
┌────────────────────────────────────────────────────────────┐
│ ● Live · synced                             Checked 3m ago │
└────────────────────────────────────────────────────────────┘
  green dot, #636366 text                     #48484a text

Degraded:
┌────────────────────────────────────────────────────────────┐
│ ● Degraded · TrackDrive slow                  Troubleshoot │
└────────────────────────────────────────────────────────────┘
  amber dot, #ff9f0a text                     #0a84ff link

Critical:
┌────────────────────────────────────────────────────────────┐
│ ● Offline · MOP API down                      Troubleshoot │
└────────────────────────────────────────────────────────────┘
  red dot, #ff453a text                       #f5f5f7 white underlined
```

- [ ] **2a. Position:** Fixed bottom of viewport, full width, 36px height, `#000` background, `0.5px solid #1c1c1e` top border.
  - Change to: ___

- [ ] **2b. Dot colors:** Green `#30d158` / Amber `#ff9f0a` / Red `#ff453a`, 8px circle.
  - Change to: ___

- [ ] **2c. Text styling:** 12px throughout. Status label inherits dot color (gray/amber/red). Timestamp `#48484a`. "Troubleshoot" link `#0a84ff` (degraded) or `#f5f5f7` underlined (critical).
  - Change to: ___

- [ ] **2d. "Troubleshoot" action:** Opens Milo with "something looks degraded — run a health check and tell me what's wrong." Only visible when degraded/critical.
  - Change to: ___

- [ ] **2e. Visibility:** Shows for both admin and non-admin users on /operator.
  - Change to: ___

- [ ] **2f. Mobile:** Same 36px height, padding shrinks to `0 16px`. Status text truncates; "Troubleshoot" stays visible.
  - Change to: ___

---

## 3. Long-Press Jiggle (Spec 03 — PillBar edit mode)

**What it looks like:**
```
Normal:
  ┌─────────┐ ┌────────────┐ ┌──────────┐ ┌──────────┐
  │ QA queue │ │ Ping health │ │ Calls    │ │ + Add    │
  └─────────┘ └────────────┘ └──────────┘ └──────────┘

Jiggle mode (all pills wobble ±2°):
  ╭──╮         ╭──╮            ╭──╮          
  │×│          │×│             │×│          
  ╰──╯         ╰──╯            ╰──╯          
  ┌~─────────┐ ┌~────────────┐ ┌~──────────┐ ┌──────────┐
  │ QA queue ~│ │ Ping health ~│ │ Calls   ~│ │  Done    │
  └~─────────┘ └~────────────┘ └~──────────┘ └──────────┘
  ↑                                            ↑
  16px red circle × button                     replaces "+ Add pill"
  at top-left, offset -6px                     "Done" in #0a84ff
```

- [ ] **3a. Entry trigger:** 500ms long-press (touch hold or mousedown without move). Haptic `vibrate(10)` on mobile.
  - Change to: ___

- [ ] **3b. Jiggle animation:** ±2 degree rotation, 0.3s cycle, infinite. Random 0–150ms delay per pill so they wobble out of sync.
  - Change to: ___

- [ ] **3c. Delete button:** 16px red circle (`#ff453a`) with white `×` (10px). Top-left of pill, offset -6px. 24px tap target.
  - Change to: ___

- [ ] **3d. "Done" button:** Replaces `+ Add pill` at end of bar. Dashed border style, label "Done" in `#0a84ff`.
  - Change to: ___

- [ ] **3e. Exit:** Tap "Done", tap outside pill bar, or Escape key. All save immediately.
  - Change to: ___

- [ ] **3f. Drag reorder:** DEFERRED to v4.1. Jiggle mode in v4.0 only supports hide/delete, not reorder.
  - Change to: ___

---

## 4. Curated Pill Picker (Spec 04 — replaces free-text "Add pill")

**What it looks like (desktop modal, 480px):**
```
┌─────────────────────────────────────────────┐
│ Add a pill                               ✕  │
├─────────────────────────────────────────────┤
│ YOUR PILLS                                  │
│ ● QA queue                       (active)   │
│ ● Ping health                    (active)   │
├─────────────────────────────────────────────┤
│ AVAILABLE                                    │
│ ○ MediaRite xref ▦ Missing CIDs + mismatch │
│ ○ Needs attention ▦ Alerts needing action   │
│ ○ Calls ▦ Call-quality alerts               │
│ ○ Pipeline ▧ Prospect pipeline overview     │
├─────────────────────────────────────────────┤
│ OTHER ROLES                             ▸   │
├─────────────────────────────────────────────┤
│ + Create custom pill                        │
└─────────────────────────────────────────────┘

Legend: ● = active (green fill)  ○ = available (gray outline)
        ▦ = opens drawer         ▧ = opens chat
```

**Mobile (bottom sheet, full width, 60vh max):**
```
        ─────    ← drag handle, 36×4px, #48484a
┌─────────────────────────────────────────────┐
│ Add a pill                               ✕  │
│ (same content as desktop, scrollable)        │
└─────────────────────────────────────────────┘
```

- [ ] **4a. Desktop container:** 480px wide, 70vh max, centered modal. Background `#1c1c1e`, border `0.5px solid #38383a`, radius 16px. Backdrop `rgba(0,0,0,0.6)` + `blur(8px)`. z-index 60.
  - Change to: ___

- [ ] **4b. Mobile container:** Full width, 60vh max, bottom sheet. Slide up 200ms ease-out. Drag handle 36×4px `#48484a`. Dismiss: drag down 50% or tap backdrop.
  - Change to: ___

- [ ] **4c. Header:** "Add a pill" 17px/600, `#f5f5f7`. Close `×` in `#636366`. Bottom border `0.5px solid #38383a`.
  - Change to: ___

- [ ] **4d. Section headers:** "YOUR PILLS" / "AVAILABLE" / "OTHER ROLES" — 11px/600, `#636366`, uppercase, 0.5px letter-spacing.
  - Change to: ___

- [ ] **4e. Pill rows:** 44px height, full-width tap. Active: green filled dot `#30d158`, label dimmed `#636366`. Available: gray outline dot `#48484a`, label `#f5f5f7`, description `#8e8e93` 12px. Hover: `#2c2c2e`.
  - Change to: ___

- [ ] **4f. Drawer vs chat icon:** Small 12px icon on each row — grid icon for 19 drawer pills, chat bubble for 21 chat pills. Color `#48484a`.
  - Change to: ___

- [ ] **4g. "Other Roles" section:** Collapsed by default (`▸`). Tap toggles. Hidden for admin (already has all pills).
  - Change to: ___

- [ ] **4h. "Create custom pill" row:** Bottom of list. `+` and label in `#0a84ff`. Separator `0.5px solid #38383a` above. Opens existing Milo conversation flow.
  - Change to: ___

---

## Design Token Reference

All values below are sourced from shipped components (PillBar, PillDrawer, MorningBriefing, SearchBar, UniversalDropZone). No new tokens are introduced — every assumption above uses an existing value.

| Role | Token | Hex |
|---|---|---|
| Page background | `bg-page` | `#000000` |
| Surface background | `bg-surface` | `#1c1c1e` |
| Hover background | `bg-hover` | `#2c2c2e` |
| Default border | `border` | `#38383a` |
| Hover border | `border-hover` | `#48484a` |
| Primary text | `text-primary` | `#f5f5f7` |
| Secondary text | `text-secondary` | `#8e8e93` |
| Tertiary text | `text-tertiary` | `#636366` |
| Disabled text | `text-disabled` | `#48484a` |
| Accent (blue) | `accent` | `#0a84ff` |
| Badge (orange) | `badge` | `#ff9f0a` |
| Severity red | `red` | `#ff453a` |
| Severity green | `green` | `#30d158` |

| Role | Token | Value |
|---|---|---|
| Pill radius | `radius-pill` | 20px |
| Card/input radius | `radius-card` | 12px |
| Modal radius | `radius-modal` | 16px |
| Pill font | `font-pill` | 13px |
| Input font | `font-input` | 15px |
| Small font | `font-small` | 11–12px |
