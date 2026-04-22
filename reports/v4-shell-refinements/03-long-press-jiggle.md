# V4 Shell Refinement: Long-Press Jiggle for Pill Edit Mode

> Coder-3 Spec | D152 | For Coder-1 execution
> Source: D85 lock — "long-press jiggle for pill customization"
> Build order: 4 of 6 (deferred — nice-to-have)

---

## User-observable behavior

**Before:** PillBar shows role-default pills + custom pills. Custom pills have a small `×` to remove. No way to reorder, hide, or enter an edit mode for the default pills. The `+ Add pill` button opens a Milo conversation to create a custom pill.

**After:**

**Entry:** Long-press (touch: 500ms hold; mouse: 500ms hold on mousedown) on any pill in the PillBar triggers "jiggle mode."

**Jiggle mode:**
1. All pills start a subtle CSS wiggle animation (iOS home screen style — ±2deg rotation at ~3Hz)
2. Each pill shows an `×` delete button (top-left corner, red circle, 16px)
3. Tapping `×` on a **default pill** hides it (stores pill id in `user_profiles.hidden_pills` JSONB array)
4. Tapping `×` on a **custom pill** removes it (existing `removeCustomPill` flow)
5. Pills become draggable (drag to reorder — stores ordered pill ids in `user_profiles.pill_order` JSONB array)
6. A "Done" button appears at the end of the pill bar to exit jiggle mode
7. Tapping anywhere outside the pill bar also exits jiggle mode

**Visual feedback:**
- Haptic feedback on long-press trigger (mobile, via `navigator.vibrate(10)`)
- Pill being dragged has slight scale-up (1.05×) and elevated shadow
- Drop target gap widens between adjacent pills during drag

**Acceptance criteria:**
- Long-press works on touch (500ms) and mouse (500ms mousedown without mousemove)
- Jiggle animation is pure CSS (no JS animation frame)
- `×` appears on ALL pills in jiggle mode (default + custom)
- Hiding a default pill persists across sessions (stored on profile)
- Hidden pills can be restored via the picker (see spec 04-curated-picker)
- Reorder persists across sessions (stored on profile)
- "Done" button exits jiggle mode and saves order
- Jiggle mode does not trigger on normal tap or short press
- Exiting jiggle mode stops all animations immediately

---

## File-level scope

| File | Change | ~LOC |
|---|---|---|
| `src/components/operator/PillBar.tsx` | Add jiggle-mode state. Add long-press detection (touchstart/mousedown timer). Render `×` on all pills in jiggle mode. Add CSS keyframes for wiggle. Add drag handlers (HTML5 Drag and Drop or touch-based). Add "Done" button. | ~120 |
| `src/app/operator/page.tsx` | Pass `onHidePill` and `onReorderPills` callbacks to PillBar. Wire to API calls. | ~15 |
| `src/app/api/profile/pill-preferences/route.ts` | **New file.** POST endpoint to update `user_profiles.hidden_pills` and `user_profiles.pill_order`. | ~40 |
| `src/lib/operator-pills.ts` | Add `applyPillPreferences()` function: filters out hidden pills, applies custom order, then merges. Called in `/api/operator` route. | ~25 |
| `src/app/api/operator/route.ts` | Call `applyPillPreferences()` when building the pill list, reading `hidden_pills` and `pill_order` from the user profile. | ~10 |

**Total: ~210 LOC across 5 files.**

**Schema change required:**
```sql
ALTER TABLE user_profiles
  ADD COLUMN IF NOT EXISTS hidden_pills JSONB DEFAULT '[]'::jsonb,
  ADD COLUMN IF NOT EXISTS pill_order JSONB DEFAULT '[]'::jsonb;
```

---

## Data dependencies

**New columns on `user_profiles`:**
- `hidden_pills`: JSONB array of pill id strings (e.g., `["today_pulse", "team_time"]`)
- `pill_order`: JSONB array of pill id strings representing custom order (e.g., `["needs_attention", "qa_queue", "top_publishers"]`)

**New API route:** `POST /api/profile/pill-preferences`
```typescript
// Request body
{ hidden_pills?: string[]; pill_order?: string[] }
// Response
{ success: true }
```

---

## Visual requirements

**No HTML design reference exists.** D144 never landed.

**Assumed jiggle animation (flagged for Mark):**
```css
@keyframes jiggle {
  0%, 100% { transform: rotate(0deg); }
  25% { transform: rotate(-2deg); }
  75% { transform: rotate(2deg); }
}
```
- Duration: 0.3s, infinite
- Each pill gets a random animation-delay (0ms–150ms) so they don't wiggle in sync
- `×` button: 16px red circle (#ff453a) with white `×`, positioned top-left of pill, offset by -6px

**Drag reorder:** The simplest approach is CSS `order` property driven by drag state, with a visual gap (8px → 24px) at the drop target. No third-party drag library. HTML5 Drag and Drop API for desktop; touch-based `touchmove` + `elementFromPoint` for mobile.

---

## Blast radius

**Medium.** Touches PillBar (high-traffic component), adds schema columns, and adds a new API route.

**Risk 1:** Drag-and-drop on mobile is notoriously tricky. HTML5 DnD doesn't work on mobile Safari. A touch-based implementation adds complexity. **Mitigation:** Ship jiggle + hide/delete first (no drag). Add reorder in a follow-up if Mark confirms it's needed.

**Risk 2:** `pill_order` can become stale when new role pills are added. If `pill_order` lists 5 pills but the role now has 7, the 2 new pills need to appear somewhere. **Mitigation:** `applyPillPreferences()` appends any role pills not in `pill_order` to the end.

**Risk 3:** Long-press conflicts with text selection on desktop. **Mitigation:** Call `e.preventDefault()` on the long-press mousedown and use `user-select: none` on the pill bar during jiggle mode.

---

## Build order rationale

Ranked 4 of 6 because:
- Nice-to-have interaction — operators can already add/remove custom pills via Milo conversation
- Requires schema migration (small but nonzero coordination)
- Mobile drag-and-drop is complex and may need iterating
- No blocking dependency on other refinements, but pairs with 04-curated-picker (hidden pills need a way to be restored)

**Recommendation:** Ship a simplified v1 first — jiggle + hide (no drag reorder). Reorder can come in v4.1.

---

## Open questions

**Q1 (assumed answer flagged):** Should hiding a default pill actually remove it from the bar, or dim it (grayed out, still visible)? **Assumed: remove entirely.** The picker (spec 04) is where hidden pills reappear. Dimming creates visual clutter in jiggle mode.

**Q2 (assumed answer flagged):** Should pill reorder be included in v4.0 or deferred? **Assumed: defer reorder to v4.1.** Hide + delete in jiggle mode is sufficient for v4.0. Reorder adds significant mobile complexity.
