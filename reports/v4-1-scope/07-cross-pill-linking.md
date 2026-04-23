# V4.1: Cross-Pill Linking (Drawer → Drawer Navigation)

> Coder-3 stub | D164b | Source: D140 Accounting impl spec

## What

Allow a DrawerItem's action to open a different pill's drawer instead of opening Milo chat. Example: the Accounting drawer's Disputes tab has an "Open disputes" action that should open the Disputes pill drawer directly, not dispatch a chat message.

## Why

D140 spec (committed aaa4fac) identified this: "The 'Open disputes' action dispatches to Milo chat (same as all drawer items). It does NOT programmatically open the Disputes pill drawer. If Mark wants a direct cross-pill link, that requires a PillDrawer enhancement. Defer to v4.1."

Currently, every DrawerItem action dispatches `action_prompt` to Milo via `openChatWith()`. There's no mechanism to programmatically open a different pill's drawer from within a drawer.

## Scope

| File | Change | ~LOC |
|---|---|---|
| DrawerItem interface | Add optional `action_pill_id?: string` — when present, clicking "Review" opens that pill's drawer instead of dispatching to chat | ~5 |
| `src/components/operator/PillDrawer.tsx` | If `action_pill_id` is set, call `onPillOpen(action_pill_id)` callback instead of `onReview` | ~10 |
| `src/app/operator/page.tsx` | Pass `onPillOpen` callback to PillDrawer that sets `drawerPill` to the target pill | ~10 |

## Rough LOC

~25 LOC across 3 files. Minimal change — adds an optional escape hatch from the "everything goes to chat" pattern.

## Dependencies

- All v4.0 drawers ship first
- Only useful where cross-pill navigation makes sense (Accounting→Disputes, Contracts→Review). Don't over-apply.
