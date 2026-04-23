# D173 — Complete V4.0: 3 Shell Refinements

**Date:** 2026-04-22  
**Agent:** Coder-1  
**Scope:** Three commits shipping the remaining V4.0 shell features from D152 specs

---

## Summary

Three commits completing the V4.0 shell:

1. `ffbd1fa` — **D173a: Curated pill picker** (~230 LOC, 4 files)
2. `2c7896b` — **D173b: Long-press jiggle mode** (~280 LOC, 4 files + new API route)
3. `a3386f1` — **D173c: Mic placement** (~130 LOC, 1 file)

Build passes clean on all three. No visual regressions.

---

## D173a: Curated Pill Picker (`ffbd1fa`)

Replaces the free-text "ask Milo" flow for `+ Add pill` with a browsable catalog overlay.

### Files

| File | Change |
|---|---|
| src/components/operator/PillPicker.tsx | **New.** Responsive picker: center modal (480px desktop) / bottom sheet (60vh mobile). Groups pills: YOUR PILLS / AVAILABLE / OTHER ROLES. |
| src/lib/operator-pills.ts | Added `PillWithRole`, `ALL_PILLS` (40 deduplicated pills), `PILL_DESCRIPTIONS` (40-entry static map), `ROLE_LABELS` |
| src/app/operator/page.tsx | `pickerOpen` state, `handlePickerAdd`, `handleCreateCustom` callbacks, PillPicker rendering |
| src/components/operator/PillBar.tsx | Title attribute update |

### Behavior

- Picker shows all 40 unique pills grouped by role membership
- Active pills shown as checked/grayed, not tappable
- Drawer-vs-chat icon per pill row
- "Other roles" collapsed section (hidden for admin)
- "Create custom pill" at bottom preserves Milo conversation path
- Dismissible via close button, backdrop click, Escape key

---

## D173b: Long-Press Jiggle Mode (`2c7896b`)

500ms hold on any pill enters iOS-style edit mode with wiggle animation and delete buttons.

### Files

| File | Change |
|---|---|
| src/components/operator/PillBar.tsx | Jiggle state, long-press detection (mouse + touch), CSS wiggle keyframes, x buttons on all pills, Done button |
| src/app/operator/page.tsx | `handleHidePill` callback (optimistic remove + POST to server) |
| src/app/api/profile/pill-preferences/route.ts | **New.** POST endpoint: `hide_pill` appends to hidden_pills, `unhide_pill` removes from array |
| src/app/api/operator/route.ts | Reads `hidden_pills` from user_profiles, filters hidden pills from response |

### Schema required

```sql
ALTER TABLE user_profiles
  ADD COLUMN IF NOT EXISTS hidden_pills JSONB DEFAULT '[]'::jsonb;
```

**This migration must be run before the feature works.** Without it, the code gracefully degrades (hidden_pills defaults to empty array, no pills are hidden).

### Behavior

- Long-press (500ms) on any pill triggers jiggle mode with haptic feedback
- All pills show red x button (top-left, -6px offset)
- Default pills: x hides (persists to `user_profiles.hidden_pills`)
- Custom pills: x removes (existing removeCustomPill flow)
- "Done" button exits jiggle mode
- Jiggle blocked during normal tap/click
- Drag reorder deferred to v4.1 per spec recommendation

---

## D173c: Mic Placement (`a3386f1`)

Web Speech API microphone button in SearchBar for voice-to-text input.

### Files

| File | Change |
|---|---|
| src/components/operator/SearchBar.tsx | Mic button (right-aligned SVG icon), SpeechRecognition integration, red pulse animation while listening, auto-submit on speech end |

### Behavior

- Mic icon visible on Chrome/Safari (Web Speech API supported)
- Hidden entirely on Firefox/unsupported browsers
- Tapping starts speech recognition, icon pulses red
- Transcribed text auto-submits via `onSubmit` callback
- Haptic feedback on mobile (navigator.vibrate)
- No new dependencies — browser-native API only

---

## Verification

| Commit | Build | Notes |
|---|---|---|
| `ffbd1fa` (D173a) | PASS | Picker renders, all 40 pills present |
| `2c7896b` (D173b) | PASS | Jiggle animation, x buttons, API endpoint created |
| `a3386f1` (D173c) | PASS | Mic SVG renders, SpeechRecognition type fixes applied |

---

## Full git log (D177 + D173)

```
a3386f1 feat: mic button in SearchBar — Web Speech API voice input (Coder-1 D173c)
2c7896b feat: long-press jiggle mode for pill hide/remove (Coder-1 D173b)
ffbd1fa feat: curated pill picker — browse + add from full 40-pill catalog (Coder-1 D173a)
8d00458 feat: aging overdue items link to disputes drawer (Coder-1 D177)
95e32ec feat: cross-pill drawer linking — action_pill_id on DrawerItem (Coder-1 D177)
```

## Remaining action item

- **Schema migration**: Run the `ALTER TABLE` for `hidden_pills` on the Supabase project (tappyckcteqgryjniwjg) before D173b jiggle mode can persist pill hides
