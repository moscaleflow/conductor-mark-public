# D146b — V4 Pill Drawer Visual Verification (Contracts, Sales Pipeline, Accounting)

**Date:** 2026-04-22  
**Agent:** Coder-1  
**Scope:** 3 pills shipped after D146 — contracts (D139), sales_pipeline (D143 C2), accounting (D143 C3)

---

## Summary

All 3 new V4 drawer pills verified against production (`tlp.justmilo.app`). API returns correct data, drawers render cleanly at desktop (1440x900) and mobile (375x812). Accounting tabbed pill — the highest defect risk — renders all 4 tabs flawlessly. No visual defects found. Three data-layer bugs discovered during seed probing (status mismatches).

## Seed Data Added

12 rows inserted across 4 tables (all tagged `d146-seed`):

| Table | Rows | Details |
|---|---|---|
| contract_documents | 4 | 2 sharing contract_group_id (dedup test), risk_level: high/medium/critical/low, all status=redlined |
| prospect_pipeline | 4 | Apex (qualifying 45d→critical), PrimeRate (outreach 16d→warning), NovaCare (drip 0d→info), Sterling (activation 9d→warning) |
| reconciliation_runs | 2 | 2026-04-20 (4 discrepancies), 2026-04-19 (2 discrepancies) |
| buyer_reports | 2 | Both status=mapping, stalled 96h and 48h |

## API Verification — All 3 PASS

| Pill | Items | Badge | Notes |
|---|---|---|---|
| contracts | 3 | 3 | Group dedup working: 4 rows → 3 groups (2 share contract_group_id, highest version kept) |
| sales_pipeline | 4 | 4 | Stall severity correct: Apex critical (45d/21d threshold), PrimeRate warning (16d/14d), Sterling warning (9d/7d), NovaCare info (recent, 0d) |
| accounting | 8 | 8 | 4 tabs: invoices(3), reconciliation(2), reports(2), disputes(1). Badge = sum of all tab items. |

## Visual Verification — 9 Screenshots

All saved to `~/Desktop/d146b-*.png`.

### Contracts

| Screenshot | Verdict |
|---|---|
| d146b-contracts.png | **PASS** — 3 items, severity dots (red/red/blue), flagged clause counts, doc type + version, Review + Snooze 24h buttons |
| d146b-contracts-mobile.png | **PASS** — full width, text wraps cleanly, all buttons accessible |

### Sales Pipeline

| Screenshot | Verdict |
|---|---|
| d146b-sales.png | **PASS** — 4 items sorted by severity, stall days + threshold shown, "Open entity" buttons, severity dots correct (red/orange/orange/blue) |
| d146b-sales-mobile.png | **PASS** — drawer full-width, text wraps at entity names, all 4 items visible |

### Accounting (Tabbed Pill)

| Screenshot | Verdict |
|---|---|
| d146b-accounting-invoices.png | **PASS** — tab strip: Invoices(3) / Reconciliation(2) / Reports(2) / Disputes(1). Active tab highlighted. 3 invoice cards with dollar amounts, AR type, overdue days, "$X at risk" labels, Review buttons |
| d146b-accounting-reconciliation.png | **PASS** — 2 reconciliation runs, Our vs TD dollar comparison, mismatch counts, "$X at risk" labels |
| d146b-accounting-reports.png | **PASS** — 2 buyer reports with filenames, row counts, status, stall hours, Process buttons |
| d146b-accounting-disputes.png | **PASS** — summary card "36 open disputes — $6,770 at risk", "Open disputes" button |
| d146b-accounting-mobile.png | **PASS** — tab strip renders, last tab label truncated to "Di..." at 375px (expected — 4 tabs on mobile). Content wraps cleanly. |

### Visual Observations

- **Tab strip**: Renders as horizontal pills with active state highlight (white background, dark text). Counts in parentheses. All 4 tabs clickable, content switches correctly.
- **Accounting badge**: Shows "8" (sum across all tabs: 3+2+2+1). Consistent with badge calculation logic.
- **Contracts group dedup**: 4 seed rows → 3 drawer items. SafeGuard Direct shows "io v2" (version 2 kept over version 1 from shared group).
- **Pipeline severity dots**: Red (critical) for 2x+ threshold, orange (warning) for 1x+, blue (info) for recent. Matches `SALES_STALL_DAYS` thresholds.
- **Mobile tab truncation**: "Disputes" truncated to "Di..." on 375px. Functional but worth considering shorter tab labels or scrollable tab strip in future.

## Bugs Found (Data Layer, Not Visual)

### Bug 3: `contract_documents` status mismatch

`fetchContractsPill()` queries:
```sql
.in('status', ['uploaded', 'analyzing', 'analyzed', 'redlined', 'pending_signature'])
```
But DB check constraint only allows: `draft`, `redlined`, `signed`, `expired`. Only `redlined` overlaps — 4 of 5 queried statuses are impossible.

**Impact:** Medium. Contracts in statuses like "uploaded" or "analyzing" will never surface in the drawer because those statuses can't exist in the DB. The pill works for `redlined` contracts but misses the full intended workflow.

### Bug 4: `buyer_reports` status mismatch

`fetchReportsTab()` queries:
```sql
.in('status', ['mapping', 'needs_mapping', 'auto_mapped'])
```
But DB check constraint only allows: `mapping`, `uploaded`, `processing`, `complete`, `error`. Only `mapping` overlaps.

**Impact:** Low. Reports tab works for `mapping` reports but `needs_mapping` and `auto_mapped` statuses cannot exist.

### Bug 5: `prospect_pipeline.milo_recommendation` column missing

Directive requested "1 with milo_recommendation NOT NULL" but the column doesn't exist on prospect_pipeline. `fetchSalesPill()` doesn't reference it either. Unfulfillable seed requirement — noted and skipped.

### Status Mismatch Pattern

This is the third instance of code querying statuses that DB constraints don't allow (Bug 1 from D146: `support_tickets` in-progress vs in_progress). Suggests the pill route code was written against intended schema that diverged from actual check constraints during migration.

## Files Created/Modified

- `tests/seed-v4-pills.ts` — extended with contract_documents, prospect_pipeline, reconciliation_runs, buyer_reports seed data
- `tests/v4-pills-d146b.spec.ts` — Playwright visual spec for 3 new pills (contracts, sales_pipeline, accounting with 4 tabs)
- `~/Desktop/d146b-*.png` — 9 screenshots for Mark's review

---

Report at: https://raw.githubusercontent.com/moscaleflow/conductor-mark-public/main/reports/2026-04-22-d146b-v4-pill-visual-verification.md
