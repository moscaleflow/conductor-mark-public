# V4 Shell Refinements: Visual Assumptions for Mark Review

> Coder-3 | D161 | 2026-04-22
> No HTML design reference exists (D144 screen inventory never landed).
> Every visual assumption below is flagged — Mark scans, approves or corrects.
> Format: one section per refinement, every token value explicit.
>
> **Instructions for Mark:** Read each item. If it's fine, skip it. If something
> should change, write the correction next to it. If everything is fine, say
> "approve all" and Coder-1 builds to these specs.

---

## Spec 01: Mic Icon in SearchBar

### Icon

| Property | Assumed value | Notes |
|---|---|---|
| Style | SF Symbols-style outline mic | Inline SVG, no external icon library |
| Stroke weight | 1.5px | Matches UniversalDropZone icon stroke weight |
| Size | 18px | Proportional to input's 15px font-size |
| Color (idle) | `#636366` | Muted gray, matches placeholder text |
| Color (listening) | `#ff453a` | Red, matches severity-critical dot |
| Color (unavailable) | `#48484a` | Darker gray, clearly disabled |

### Position

| Property | Assumed value |
|---|---|
| Placement | Right side of input, inside the border radius |
| Input padding-right | 44px (leaves room for icon + breathing space) |
| Icon right offset | 14px from input right edge |
| Vertical alignment | Centered in input (same as text baseline) |

### Animation (listening state)

| Property | Assumed value |
|---|---|
| Type | Opacity pulse |
| Keyframes | `0%: opacity 1` → `50%: opacity 0.4` → `100%: opacity 1` |
| Duration | 1s, infinite |
| Timing | `ease-in-out` |

### Behavior

| Property | Assumed value |
|---|---|
| Tap/click | Start listening |
| Tap again while listening | Stop listening, submit transcription |
| Speech end (silence) | Auto-submit after 1.5s silence |
| Permission denied | Icon grays out, tooltip: "Microphone not available" |
| Unsupported browser | Icon hidden entirely |

---

## Spec 02: Footer Sync Status Bar

### Layout

```
┌──────────────────────────────────────────────────────────┐
│ ● Live · synced                           Checked 3m ago │
└──────────────────────────────────────────────────────────┘
```

| Property | Assumed value |
|---|---|
| Position | `fixed`, bottom of viewport |
| Width | Full viewport width |
| Height | 36px |
| Padding | `0 24px` (matches page content padding) |
| Background | `#000000` (matches page) |
| Border-top | `0.5px solid #1c1c1e` |
| z-index | 10 (below PillDrawer at 50, below UniversalDropZone at 9999) |

### Text styling

| Element | Font size | Color (healthy) | Color (degraded) | Color (critical) |
|---|---|---|---|---|
| Status dot | 8px circle | `#30d158` (green) | `#ff9f0a` (amber) | `#ff453a` (red) |
| Status label | 12px | `#636366` | `#ff9f0a` | `#ff453a` |
| Timestamp / "Troubleshoot" | 12px | `#48484a` | `#0a84ff` (link blue) | `#f5f5f7` (white, underlined) |

### Content by state

| State | Left text | Right text |
|---|---|---|
| Healthy | `● Live · synced` | `Checked 3m ago` |
| Degraded | `● Degraded · {service} slow` | `Troubleshoot` (clickable) |
| Critical | `● Offline · {service} down` | `Troubleshoot` (clickable, white) |
| Loading | `● Checking...` | (empty) |

### Mobile

| Property | Assumed value |
|---|---|
| Text truncation | Status label truncates, "Troubleshoot" stays visible |
| Height | Same 36px |
| Padding | `0 16px` |

---

## Spec 03: Long-Press Jiggle Mode

### Entry trigger

| Property | Assumed value |
|---|---|
| Touch | 500ms hold without movement |
| Mouse | 500ms mousedown without mousemove (>5px threshold) |
| Haptic (mobile) | `navigator.vibrate(10)` on trigger |

### Jiggle animation

```css
@keyframes jiggle {
  0%, 100% { transform: rotate(0deg); }
  25%      { transform: rotate(-2deg); }
  75%      { transform: rotate(2deg); }
}
```

| Property | Assumed value |
|---|---|
| Duration | 0.3s |
| Iteration | infinite |
| Timing | ease-in-out |
| Per-pill delay | Random 0ms–150ms (so pills don't wiggle in sync) |

### Delete button (x)

| Property | Assumed value |
|---|---|
| Shape | Circle |
| Size | 16px diameter |
| Background | `#ff453a` (red) |
| Icon | White `×`, 10px font, centered |
| Position | Top-left corner of pill, offset -6px both axes |
| Tap target | 24px (larger than visual for touch) |

### Pill drag state (v4.1 — deferred)

| Property | Assumed value |
|---|---|
| Scale on grab | 1.05× |
| Shadow on grab | `0 4px 16px rgba(0,0,0,0.4)` |
| Drop target gap | Adjacent pills spread from 8px to 24px gap |
| Drop animation | 200ms ease-out snap to position |

### Done button

| Property | Assumed value |
|---|---|
| Label | "Done" |
| Position | End of pill bar (after last pill, replacing `+ Add pill`) |
| Style | Same as `+ Add pill` dashed border, but label "Done" in `#0a84ff` |

### Exit triggers

| Trigger | Action |
|---|---|
| Tap "Done" | Exit jiggle, save order |
| Tap outside pill bar | Exit jiggle, save order |
| Press Escape | Exit jiggle, save order |
| Navigate away | Exit jiggle, discard unsaved changes |

---

## Spec 04: Curated Pill Picker

### Desktop modal

| Property | Assumed value |
|---|---|
| Width | 480px |
| Max height | 70vh |
| Border radius | 16px |
| Background | `#1c1c1e` |
| Border | `0.5px solid #38383a` |
| Backdrop | `rgba(0,0,0,0.6)` with `backdrop-filter: blur(8px)` |
| Position | Centered (fixed, inset 0, flex center) |
| z-index | 60 (above PillDrawer at 50) |

### Mobile bottom sheet

| Property | Assumed value |
|---|---|
| Width | 100% |
| Max height | 60vh |
| Border radius | 16px top corners only |
| Drag handle | 36px wide, 4px tall, `#48484a`, centered at top with 8px margin |
| Animation | Slide up from bottom, 200ms ease-out |
| Dismiss | Drag down past 50% threshold, or tap backdrop |

### Header

| Property | Assumed value |
|---|---|
| Text | "Add a pill" |
| Font size | 17px, weight 600 |
| Color | `#f5f5f7` |
| Close button | `×` in `#636366`, 15px, top-right |
| Padding | `16px 20px` |
| Border-bottom | `0.5px solid #38383a` |

### Section headers

| Property | Assumed value |
|---|---|
| Labels | "YOUR PILLS", "AVAILABLE", "OTHER ROLES" |
| Font size | 11px |
| Weight | 600 |
| Color | `#636366` |
| Transform | uppercase |
| Letter spacing | 0.5px |
| Padding | `12px 20px 4px` |

### Pill rows

| Property | Assumed value |
|---|---|
| Height | 44px |
| Padding | `0 20px` |
| Layout | Flex row: checkbox/indicator + label + description |
| Tap target | Full row width |

| Element | Active pill | Available pill |
|---|---|---|
| Indicator | Filled circle check in `#30d158` | Empty circle outline in `#48484a` |
| Label | `#636366` (dimmed) | `#f5f5f7` |
| Description | Hidden | `#8e8e93`, 12px, truncated to 1 line |
| Hover | No hover (disabled) | Background → `#2c2c2e` |

### Drawer vs chat indicator

| Property | Assumed value |
|---|---|
| Icon | Small grid icon (drawer) or chat bubble icon (chat) |
| Size | 12px, color `#48484a` |
| Position | Right side of row, before description |
| Purpose | 19 drawer-capable pills get grid icon; 21 chat-only pills get bubble icon |

### "Other Roles" section

| Property | Assumed value |
|---|---|
| Default state | Collapsed (header only, with `▸` chevron) |
| Tap header | Toggles open/closed with `▸` / `▾` |
| Hidden for admin | Yes — admin role already spans all domains |

### "Create custom pill" row

| Property | Assumed value |
|---|---|
| Position | Bottom of list, below all sections |
| Icon | `+` in `#0a84ff` |
| Label | "Create custom pill" in `#0a84ff` |
| Behavior | Closes picker, opens Milo conversation (existing flow) |
| Separator | `0.5px solid #38383a` above this row |

---

## Design system reference (existing tokens used above)

These values are pulled from the shipped PillBar, PillDrawer, MorningBriefing, and SearchBar components:

| Token | Hex | Usage |
|---|---|---|
| Background (page) | `#000000` | Operator page, Top5Frame |
| Background (surface) | `#1c1c1e` | Pills, inputs, cards, modals |
| Background (hover) | `#2c2c2e` | Input focus, pill hover |
| Border (default) | `#38383a` | Pills, inputs, cards |
| Border (hover) | `#48484a` | Pill hover, add-pill dashed |
| Text (primary) | `#f5f5f7` | Labels, headings |
| Text (secondary) | `#8e8e93` | Descriptions, subtitles |
| Text (tertiary) | `#636366` | Placeholders, muted labels |
| Text (disabled) | `#48484a` | Disabled states |
| Accent (blue) | `#0a84ff` | Links, focus borders, CTA |
| Badge (orange) | `#ff9f0a` | Pill count badges, degraded status |
| Severity (red) | `#ff453a` | Critical dots, errors, delete |
| Severity (green) | `#30d158` | Healthy dots, success checks |
| Border radius (pill) | 20px | Pills |
| Border radius (card) | 12px | Cards, inputs |
| Border radius (modal) | 16px | Modals, sheets |
| Font (pill label) | 13px | Pills |
| Font (input) | 15px | SearchBar |
| Font (small) | 11-12px | Badges, footers, section headers |
