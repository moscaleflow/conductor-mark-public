# V4.1: Snooze for Non-Alert Drawer Items

> Coder-3 stub | D164b | Source: D140 impl spec recommendation

## What

Extend the PillDrawer's "Snooze 24h" button to work on items from non-alert data sources (Sales pipeline entities, Contracts, Accounting items). Currently, snooze only works on alert-sourced items because it POSTs to `/api/briefing/action` with an `alert_id`.

## Why

D140 spec (committed aaa4fac) identified this gap: PillDrawer renders a "Snooze 24h" button on every DrawerItem, but the snooze API (`/api/briefing/action`) only accepts `alert_id`. For Sales pipeline entities (from `prospect_pipeline`), Contracts (from `contract_documents`), and Accounting items (from `invoices`, `disputes`, etc.), there is no alert to snooze.

D140 recommended skipping snooze for non-alert items in v4.0 and adding a `snoozable?: boolean` flag to DrawerItem. V4.1 builds the actual snooze mechanism.

## Scope

**Option A: Generic snooze table**

| File | Change | ~LOC |
|---|---|---|
| New: `snoozed_items` table | `user_id, source_table, source_id, snoozed_until` | Schema migration |
| `src/app/api/operator/pill/route.ts` | Each direct-source pill filters out snoozed items by checking `snoozed_items` | ~20 per pill (7 pills × 20 = ~140) |
| `src/app/api/operator/snooze/route.ts` | **New.** POST: creates snooze record. Uses source_table + source_id (generic). | ~40 |
| `src/components/operator/PillDrawer.tsx` | Snooze button calls new endpoint for non-alert items. | ~10 |
| DrawerItem interface | Add `snoozable?: boolean` (default true for alerts, conditional for others) | ~5 |

**Option B: Per-user JSONB on user_profiles**
Store `snoozed_pills: { [pill_id]: { [item_id]: snoozed_until } }` on user_profiles. Simpler schema, but JSONB gets large if operators snooze frequently.

**Recommended: Option A** — normalized table, clean queries, no JSONB bloat.

## Rough LOC

~195 LOC + 1 schema migration.

## Dependencies

- D140's `snoozable` flag proposal should ship with v4.0 so the button is hidden on non-alert items until this feature lands
- All direct-source pills (7+) need the snoze filter added
