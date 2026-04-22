# D139 + D143 Report: Contracts, Sales, Accounting Pills + PillDrawer Tabs

**Agent:** Coder-2 | **Date:** 2026-04-22 | **Status:** Complete (4 commits, all builds clean)

---

## Commit Summary

| Commit | SHA | Directive | Change |
|--------|-----|-----------|--------|
| 1 | `32355b4` | D139 | Contracts direct-source pill |
| 2 | `ec51453` | D143 C1 | PillDrawer snoozable flag + tab rendering |
| 3 | `6e110d8` | D143 C2 | Sales pipeline direct-source pill |
| 4 | `25a0864` | D143 C3 | Accounting tabbed pill (4 sub-queries) |

---

## D139: Contracts Pill

**Predicate conflict resolved:** `contracts_review` IS alert-based with active predicates (matches `/contract/i`, `/msa/i`, `/redline/i`, `/addendum/i` OR `entity_type === 'contract'`). New pill uses ID `'contracts'` — both coexist. `contracts_review` continues to filter alerts for admin executive badges. `contracts` queries `contract_documents` directly.

**Query:** `contract_documents` WHERE `status` IN ('uploaded', 'analyzing', 'analyzed', 'redlined', 'pending_signature'). Grouped by `contract_group_id`, latest version per group. Parses `analysis_result.flagged_clauses` for count/severity breakdown.

**Severity mapping:** `risk_level` HIGH/CRITICAL → critical, MEDIUM → warning, LOW/null → info. `revenue_at_risk: 0` (per D136 spec — contracts carry legal risk, not dollar risk).

**Files:** pill/route.ts (+74 lines), operator-pills.ts (+1 admin pill), operator/page.tsx (+1 DRAWER_PILL_ID).

---

## D143 Commit 1: PillDrawer Cross-Cutting

**snoozable flag:** Added `snoozable?: boolean` to DrawerItem interface (both PillDrawer.tsx export and pill/route.ts local). Defaults true for backward compat. When `false`, Snooze button is hidden. All non-alert pills (contracts, sales, accounting) set `snoozable: false`.

**Tab rendering:** Added `TabData` interface, `tabs`/`activeTab` state. When API response includes `tabs[]`, PillDrawer renders a segmented-control tab strip (fixed, non-scrolling) between header and scroll area. Active tab filters items. `activeTab` set from `response.default_tab` on fetch, reset on pill change.

**Tab strip styling:** Dark pill buttons. Active: `#2c2c2e` background, `#f5f5f7` text. Inactive: transparent, `#636366` text. Matches existing dark theme.

**Header count:** Updated to show total across all tabs when tabbed response.

**Files:** PillDrawer.tsx (338 → 399 lines, +61 net).

---

## D143 Commit 2: Sales Pipeline Pill

**Pill ID:** `sales_pipeline`

**Query:** `prospect_pipeline` WHERE `stage` NOT IN ('active', 'dormant', 'blacklisted'). Ordered by `stage_changed_at` ascending (most stalled first).

**Filter:** Union of stalled (over threshold) OR recently changed (< 48h).

**Stall thresholds:** `SALES_STALL_DAYS` constant — outreach: 14d, qualifying: 21d, drip: 30d, onboarding: 14d, activation: 7d.

**Severity:** `daysInStage >= threshold * 2` → critical, `>= threshold` → warning, recently changed → info.

**Sort:** Severity desc, then `created_at` asc (oldest stalled first within severity).

**Roles:** Added to outreach, prospecting, admin ROLE_PILLS.

**Files:** pill/route.ts (+78 lines), operator-pills.ts (+3 pills), operator/page.tsx (+1 DRAWER_PILL_ID).

---

## D143 Commit 3: Accounting Tabbed Pill

**Pill ID:** `accounting`

**Architecture:** `fetchAccountingPill()` runs 4 sub-queries in parallel via `Promise.all`. Returns `TabData[]` response with `default_tab` from `pickDefaultTab()` (highest-severity item across all tabs).

### Sub-tabs

| Tab | Source | Filter | Severity |
|-----|--------|--------|----------|
| Invoices | `invoices` | status IN (sent, overdue, jen_flagged) | 90d+ or jen_flagged → critical, 60d+ → warning, else → info |
| Reconciliation | `reconciliation_runs` | status = discrepancies_found, limit 10 | Revenue diff > $500 → critical, else → warning |
| Reports | `buyer_reports` | status IN (mapping, needs_mapping, auto_mapped) AND > 24h old | > 72h → warning, else → info |
| Disputes | `disputes` | status IN (detected, raised, pending_response, escalated) | $5k+ total → critical, > $0 → warning, else → info |

**Disputes tab:** Single summary item (count + total $ at risk), not individual disputes. "Open disputes" action dispatches to Milo chat.

**Badge:** Total count across all tabs. No dollar rollup (heterogeneous data).

**All items:** `snoozable: false`.

**Roles:** Added to billing (first slot) and admin ROLE_PILLS.

**Files:** pill/route.ts (+198 lines), operator-pills.ts (+2 pills), operator/page.tsx (+1 DRAWER_PILL_ID).

---

## File Size After

| File | Before | After | Delta |
|------|--------|-------|-------|
| pill/route.ts | 567 | 923 | +356 |
| PillDrawer.tsx | 338 | 399 | +61 |
| operator-pills.ts | 226 | 231 | +5 |
| operator/page.tsx | 497 | 500 | +3 |

---

## DIRECT_SOURCE_PILLS Final State

```
['qa', 'disputes', 'aging', 'contracts', 'sales_pipeline', 'accounting']
```

6 pills bypass the alerts table. All other pills continue through the `PILL_PREDICATES` filter system.

---

## DRAWER_PILL_IDS Final State

```
['qa_queue', 'qc_queue', 'ping_summary', 'ping_health', 'quality_overview',
 'mediarite_xref', 'needs_attention', 'dispute_recovery', 'disputes',
 'calls', 'aging', 'contracts', 'sales_pipeline', 'accounting']
```

14 pill IDs open the drawer. All others dispatch to Milo chat.

---

## Pattern 17 Note

All tables referenced (contract_documents, prospect_pipeline, invoices, reconciliation_runs, buyer_reports, disputes) had columns verified in the pre-read phase. All tables are currently empty (0 rows) — the pills will return empty states until production data is seeded. PillDrawer renders "Nothing in this queue. Milo is watching." for empty results.

---

## Not Implemented (Deferred per Spec)

- **Option B `renderers?` prop** — Contracts uses standard DrawerItem rendering. Custom per-pill renderers deferred to v4.1 when A1 IssueCard components need to render inside the drawer.
- **Footer links** (e.g., "View full pipeline →") — PillDrawer has no footer mechanism. Defer.
- **Cross-pill links** (Accounting Disputes tab → Disputes pill) — Requires `onPillOpen` callback. Defer.
- **Snooze for non-alert items** — Requires new mechanism (snoozed_pills JSONB or similar). v4.1.
