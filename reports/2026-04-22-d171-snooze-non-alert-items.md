# D171 Report: Generic Snooze for Non-Alert DrawerItems (D164b Item 04)

**Agent:** Coder-2 | **Date:** 2026-04-22 | **Status:** Complete (build clean, verified E2E)

---

## Summary

Extended the PillDrawer's "Snooze 24h" button to work on all 10 direct-source pills, not just alert-sourced items. Previously, snooze only worked via `/api/briefing/action` which updates the `alerts` table — non-alert pills had `snoozable: false` and no snooze button.

---

## Changes

### 1. Migration: `snoozed_items` table

**File:** `supabase/migrations/20260422200001_snoozed_items.sql`

```sql
CREATE TABLE snoozed_items (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL,
  source_table TEXT NOT NULL,
  source_id TEXT NOT NULL,
  snoozed_until TIMESTAMPTZ NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  UNIQUE (user_id, source_table, source_id)
);
```

Applied to shared project (tappyckcteqgryjniwjg) via `supabase db push`.

### 2. API Route: `/api/operator/snooze`

**File:** `src/app/api/operator/snooze/route.ts` (36 LOC)

POST endpoint. Accepts `{ userId, source_table, source_id }`. Upserts into `snoozed_items` with `snoozed_until = NOW() + 24h`. Uses `onConflict: 'user_id,source_table,source_id'` for idempotent re-snooze.

### 3. Pill Route: snooze filter + source_table injection

**File:** `src/app/api/operator/pill/route.ts` (+42 LOC, -8 LOC)

- Added `PILL_SOURCE_TABLE` map: pill ID → table name (10 entries)
- Added `fetchSnoozedIds(userId, sourceTable)` helper: queries `snoozed_items` for active snoozes
- Post-processing at dispatch level: all direct-source results get `source_table` injected on items and snoozed items filtered out
- Handles both flat items (9 pills) and tabbed items (accounting)
- Removed `snoozable: false` from all 8 fetchers that had it (esign, drafts, publishers, buyers, sales_pipeline, contracts, aging, accounting tabs)

### 4. PillDrawer: branched snooze handler

**File:** `src/components/operator/PillDrawer.tsx` (+19 LOC, -3 LOC)

- Added `source_table?: string` to exported `DrawerItem` interface
- `handleSnooze` branches: `source_table !== 'alerts'` → POST `/api/operator/snooze`; else → POST `/api/briefing/action` (legacy alert path)
- Snooze also updates tab state for tabbed pills (accounting)

---

## Verification

| Test | Result |
|------|--------|
| Build | Clean (0 errors, 0 warnings) |
| E-sign pill: source_table injected | `signing_documents` on all items |
| Snooze POST endpoint | `{"success":true,"snoozed_until":"2026-04-24T01:05:05.876Z"}` |
| Snoozed item filtered | Count dropped 84 → 83, item absent from response |
| Accounting tabs: source_table | All 4 tabs have `source_table: "invoices"` |

---

## Also included in commit

- D163a seed extensions: signing_documents (3), leads (2), send_log (2), crm_counterparties publisher (3) + buyer (3)
- D163a Playwright visual spec: `tests/d163a-d151-visual.spec.ts`

---

## LOC Breakdown

| Component | LOC |
|-----------|-----|
| Migration | 12 |
| Snooze API route | 36 |
| Pill route changes | 42 |
| PillDrawer changes | 19 |
| **Total** | **109** |

(Spec estimated ~195 LOC — came in at 109 due to dispatch-level post-processing instead of per-fetcher modifications.)

---

Report at: https://raw.githubusercontent.com/moscaleflow/conductor-mark-public/main/reports/2026-04-22-d171-snooze-non-alert-items.md
