# V4.1: Pill Drag Reorder

> Coder-3 stub | D164b | Prerequisite: V4.0 spec 03 (jiggle + hide) ships first

## What

Add drag-to-reorder to jiggle mode. When the operator enters jiggle mode (long-press), pills become draggable. Dropping a pill between two others reorders the bar. Order persists in `user_profiles.pill_order` JSONB.

## Why

Spec 03 deferred reorder to v4.1 because mobile drag-and-drop is complex. V4.0 jiggle mode only supports hide/delete. Operators who want a specific pill ordering must hide and re-add pills in the desired sequence — drag is the natural UX.

## Scope

| File | Change | ~LOC |
|---|---|---|
| `src/components/operator/PillBar.tsx` | Add drag handlers. Desktop: HTML5 DnD API. Mobile: `touchmove` + `elementFromPoint`. Visual: grabbed pill scales 1.05×, drop gap widens 8→24px. | ~80 |
| `src/app/api/profile/pill-preferences/route.ts` | Already handles `pill_order` JSONB (shipped with spec 03). No change. | 0 |

## Rough LOC

~80 LOC in PillBar. No new API. No schema change (column ships with spec 03).

## Dependencies

- Spec 03 must ship first (creates `pill_order` column and `applyPillPreferences()`)
- Mobile touch DnD needs cross-browser testing (Safari, Chrome Android)
- Consider a lightweight drag library (`@dnd-kit/core`, ~8KB) vs hand-rolled touch handlers. Hand-rolled is consistent with the no-deps pattern in this codebase but more brittle.

## Risk

Mobile Safari `touchmove` + scroll interaction. If the pill bar is inside a scrollable container, dragging a pill may fight with scroll. Mitigation: disable page scroll during active drag via `touch-action: none` on the pill bar.
