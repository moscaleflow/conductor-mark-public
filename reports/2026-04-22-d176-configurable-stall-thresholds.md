# D176 — Configurable Sales Stall Thresholds

**Date:** 2026-04-22  
**Agent:** Coder-3  
**Scope:** V4.1 Item 06 — promote hardcoded SALES_STALL_DAYS to per-operator JSONB config

---

## Design decision

**DB column on `user_profiles`** — `pill_config JSONB DEFAULT '{}'`.

Why not alternatives:
- **Env vars**: require a deploy to change. The whole point is operator-level customization without code changes.
- **Dedicated settings table**: overkill for 5 numbers. `pill_config` on the existing `user_profiles` table reuses the same pattern as `custom_pills JSONB` (already shipped, migration `20260410200001`).
- **Admin UI page**: overkill. Milo conversation tool (future: "Set my qualifying stall threshold to 30 days") or a simple JSON edit on the profile is sufficient.

`pill_config` is generic — keyed by feature name. First key: `sales_stall_days`. Future keys can hold settings for other pills without additional schema changes.

## Changes

### 1. Migration: `pill_config` column

**File:** `supabase/migrations/20260422300001_pill_config.sql`

```sql
ALTER TABLE user_profiles
  ADD COLUMN IF NOT EXISTS pill_config JSONB NOT NULL DEFAULT '{}'::jsonb;
```

### 2. Pill route: configurable thresholds

**File:** `src/app/api/operator/pill/route.ts`

**Renamed** `SALES_STALL_DAYS` → `DEFAULT_STALL_DAYS` (same values: outreach=14, qualifying=21, drip=30, onboarding=14, activation=7).

**Added** `loadStallThresholds(userId)` — reads `pill_config.sales_stall_days` from user_profiles, merges with defaults. If no custom config exists, returns defaults unchanged. Single query, ~1ms overhead.

**Updated** `fetchSalesPill()` → `fetchSalesPill(stallDays)` — accepts thresholds as parameter instead of referencing the module-level constant.

**Updated** call site in GET handler — loads thresholds only when `pill === 'sales_pipeline'`, avoiding unnecessary DB reads for other pills.

## LOC

| Component | LOC |
|---|---|
| Migration | 6 |
| loadStallThresholds function | 12 |
| fetchSalesPill signature change | 2 |
| Call site update | 1 |
| **Total** | **21** |

(Spec estimated ~50 LOC — came in at 21 because the settings UI / Milo tool is deferred.)

## Verification

- **Build:** PASS (clean — verified with page.tsx reverted; a pre-existing page.tsx error from another Coder's in-progress D178 cross-pill linking is unrelated)
- **Default behavior preserved:** When `pill_config` is `{}` (all existing users), `loadStallThresholds` returns `DEFAULT_STALL_DAYS` unchanged
- **Custom override path:** Setting `pill_config = '{"sales_stall_days": {"outreach": 7}}'` on a user_profiles row would lower the outreach threshold to 7d while keeping all other stages at defaults

## Config format

```json
{
  "sales_stall_days": {
    "outreach": 7,
    "qualifying": 14,
    "drip": 21,
    "onboarding": 10,
    "activation": 5
  }
}
```

Partial overrides merge with defaults — only specify the stages you want to change.

## Commit

| Repo | SHA | Description |
|---|---|---|
| milo-for-ppc | `5863eb2` | Migration + pill route configurable thresholds |

## What's NOT included (future)

- **Milo conversation tool** for setting thresholds ("Set my qualifying stall threshold to 30 days") — deferred, can reuse `createCustomPill` pattern
- **Settings UI** — not needed until multiple operators request different thresholds

---

Report at: https://raw.githubusercontent.com/moscaleflow/conductor-mark-public/main/reports/2026-04-22-d176-configurable-stall-thresholds.md
