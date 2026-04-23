# V4.1: Left Drawer + Bottom Sheet + Full-Screen Surfaces

> Coder-3 stub | D164b | Source: D85 lock — "surfaces render as right drawer / left drawer / center modal / bottom sheet / full-screen takeover depending on surface type"

## What

D85 describes 5 surface types for pill content. V4.0 ships only the right drawer (PillDrawer.tsx, 460px right-side slide-out). V4.1 adds the remaining 3 non-modal surface types:
- **Left drawer** — mirror of right drawer, slides from left
- **Bottom sheet** — mobile-first surface, slides up from bottom
- **Full-screen takeover** — replaces the page entirely (for complex workflows)

(Center modal already exists via the ConversationModal pattern.)

## Why

Some pill content doesn't fit a 460px right drawer. Pipeline kanban needs width (left drawer or full-screen). Mobile users need bottom sheets for quick actions. The surface type should be a property of the pill, not hardcoded.

## Scope

| File | Change | ~LOC |
|---|---|---|
| `src/lib/operator-pills.ts` | Add `surface?: 'right-drawer' | 'left-drawer' | 'bottom-sheet' | 'full-screen' | 'modal'` to Pill interface. Default: `right-drawer`. | ~5 |
| `src/components/operator/LeftDrawer.tsx` | **New.** Mirror of PillDrawer with left-side slide. Reuse DrawerItem rendering. | ~100 (extracted from PillDrawer) |
| `src/components/operator/BottomSheet.tsx` | **New.** Bottom sheet with drag handle, snap points (30%/60%/90vh), backdrop. | ~120 |
| `src/components/operator/FullScreenSurface.tsx` | **New.** Full viewport replacement with back button. | ~60 |
| `src/app/operator/page.tsx` | Route pill click to appropriate surface based on `pill.surface`. | ~20 |

## Rough LOC

~305 LOC across 5 files (3 new, 2 modified).

## Dependencies

- V4.0 PillDrawer must be stable first (it becomes the template for other surfaces)
- Need Mark input on which pills use which surface type
- Bottom sheet snap-point behavior needs design iteration (how many snap points, threshold for dismiss)
- Consider: should surface type be per-pill or per-device? A pill might be a right-drawer on desktop and a bottom-sheet on mobile.
