# D140: Sales + Accounting Drawer Implementation Specs

> Coder-3 Research | Directive #140 | Spec for Coder-2 handoff — no code changes
> Prerequisite: D134 (Q&A + Disputes + Aging + Calls retune) shipped by Coder-1
> Reference: D136 design specs, D86 pill scope, D85 amendment

---

## Architecture Context

Coder-1 D134 established the `DIRECT_SOURCE_PILLS` pattern in [pill/route.ts:65](src/app/api/operator/pill/route.ts#L65):

```typescript
const DIRECT_SOURCE_PILLS = new Set(['qa', 'disputes']);
```

Line 387-395 branches: if pill is in `DIRECT_SOURCE_PILLS`, call the pill-specific fetch function and return early. Otherwise fall through to the alert-based predicate system. Each fetch function returns `{ items: DrawerItem[], count: number, badge_label: string }`.

Both new pills follow this pattern exactly: add to `DIRECT_SOURCE_PILLS`, write a `fetchSalesPill()` / `fetchAccountingPill()` function.

---

## Spec 1: Sales Drawer

### Pill ID

`sales_pipeline`

### Files to modify

| File | Change | Lines affected |
|---|---|---|
| `src/app/api/operator/pill/route.ts` | Add `'sales_pipeline'` to `DIRECT_SOURCE_PILLS`. Add `fetchSalesPill()`. Extend the branching `if` at L387-395. | ~60 new lines |
| `src/lib/operator-pills.ts` | Add `sales_pipeline` pill to `outreach`, `prospecting`, and `admin` ROLE_PILLS arrays. | 3 inserts |

No changes to `PillDrawer.tsx` — Sales uses the standard `DrawerItem[]` response.

### Query

```sql
SELECT id, entity_name, entity_type, stage, stage_changed_at,
       updated_at, created_at, source
FROM prospect_pipeline
WHERE stage NOT IN ('active', 'dormant', 'blacklisted')
ORDER BY stage_changed_at ASC NULLS FIRST
```

Ascending by `stage_changed_at` puts the most stalled entities first.

### Stall thresholds

Define as a constant inside the pill route file (not exported, not in a shared lib):

```typescript
const SALES_STALL_DAYS: Record<string, number> = {
  outreach: 14,
  qualifying: 21,
  drip: 30,
  onboarding: 14,
  activation: 7,
};
```

### Filter logic (inside `fetchSalesPill`)

```typescript
const now = Date.now();
const FORTY_EIGHT_HOURS = 48 * 60 * 60 * 1000;

const interesting = rows.filter((row) => {
  const ref = row.stage_changed_at ?? row.updated_at ?? row.created_at;
  const daysInStage = ref
    ? Math.floor((now - new Date(ref).getTime()) / 86400000)
    : 0;
  const threshold = SALES_STALL_DAYS[row.stage] ?? 21;

  // Stalled: over threshold
  if (daysInStage >= threshold) return true;

  // Recently changed: within 48h
  if (ref && (now - new Date(ref).getTime()) < FORTY_EIGHT_HOURS) return true;

  return false;
});
```

### Severity mapping

```typescript
function salesSeverity(daysInStage: number, threshold: number): DrawerItem['severity'] {
  if (daysInStage >= threshold * 2) return 'critical';
  if (daysInStage >= threshold) return 'warning';
  return 'info';  // recently changed, not stalled
}
```

### DrawerItem mapping

```typescript
{
  id: row.id,
  severity: salesSeverity(daysInStage, threshold),
  title: `${row.entity_name} — ${row.stage} (${daysInStage}d)`,
  description: isStalled
    ? `${capitalize(row.entity_type)}. Stalled ${daysInStage} days in ${row.stage} (threshold: ${threshold}d).`
    : `${capitalize(row.entity_type)}. Moved to ${row.stage} ${hoursAgo}h ago.`,
  entity_name: row.entity_name,
  revenue_at_risk: 0,
  action_label: 'Open entity',
  action_prompt: `Show me the full story on ${row.entity_name} — why are they ${isStalled ? `stalled in ${row.stage} for ${daysInStage} days` : `newly in ${row.stage}`}? Pull recent activity and recommend next steps.`,
  created_at: ref,
}
```

### Sort order

1. Severity descending (critical → warning → info)
2. `daysInStage` descending within same severity
3. `created_at` ascending within same daysInStage (oldest first)

### Badge

Item count: `badge_label: String(items.length)`. No dollar rollup (revenue_at_risk is always 0).

### Empty state

PillDrawer already renders "Nothing in this queue. Milo is watching." when `items.length === 0`. No change needed.

### Footer link

Not in v4.0. PillDrawer has no footer link mechanism. If Mark wants "View full pipeline →" in the drawer, that's a PillDrawer enhancement (add optional `footerLink` prop). Defer — the operator can type "show pipeline" in the search bar.

### Pill definition (operator-pills.ts)

```typescript
// Add to outreach role:
{ id: 'sales_pipeline', label: 'Pipeline attention', query: 'Show pipeline entities that need attention — stalled or recently changed.', focus_area_tag: 'setup' },

// Add to prospecting role:
{ id: 'sales_pipeline', label: 'Pipeline attention', query: 'Show pipeline entities that need attention — stalled or recently changed.', focus_area_tag: 'setup' },

// Add to admin role (after existing pills):
{ id: 'sales_pipeline', label: 'Pipeline attention', query: 'Show pipeline entities that need attention — stalled or recently changed.' },
```

### Acceptance criteria

1. Pill appears in PillBar for outreach, prospecting, admin roles
2. Pill badge shows count of attention-needed entities (stalled + recently changed)
3. Tapping pill opens PillDrawer with entities sorted by severity → daysInStage
4. Each item shows entity name, stage, days in stage, entity type, reason (stalled vs recent)
5. "Open entity" button dispatches action_prompt to Milo chat
6. "Snooze 24h" button calls `/api/briefing/action` — **NOTE:** snooze currently works on alert IDs. For pipeline entities, snooze needs a new mechanism. Options: (a) write a snoozed_pills JSONB on user_profiles, (b) skip snooze for Sales in v4.0. **Recommendation: skip snooze for v4.0.** Pass `action_label: 'Open entity'` only, no secondary button. Snooze on non-alert items is a v4.1 feature.
7. Empty state: "Nothing in this queue. Milo is watching."
8. No stalled entities in active/dormant/blacklisted stages (excluded from query)

---

## Spec 2: Accounting Drawer (4 Sub-Tabs)

### Pill ID

`accounting`

### Files to modify

| File | Change | Lines affected |
|---|---|---|
| `src/app/api/operator/pill/route.ts` | Add `'accounting'` to `DIRECT_SOURCE_PILLS`. Add `fetchAccountingPill()` with 4 parallel sub-queries. Extend branching at L387-395. | ~120 new lines |
| `src/components/operator/PillDrawer.tsx` | Add tab rendering conditional: if response has `tabs` field, render `AccountingTabStrip` + filtered items. | ~50 new lines (inline, no new file) |
| `src/lib/operator-pills.ts` | Add `accounting` pill to `billing` and `admin` ROLE_PILLS arrays. | 2 inserts |

### PillDrawer tab rendering

Add inside [PillDrawer.tsx](src/components/operator/PillDrawer.tsx) at the content area (line 229, inside the scroll div). The change is a conditional branch — if the API response includes a `tabs` array, render tab headers + filtered items instead of the flat items list.

**Response shape with tabs:**

```typescript
interface TabData {
  id: string;
  label: string;
  count: number;
  items: DrawerItem[];
}

// API response when pill === 'accounting':
{
  pill: 'accounting',
  count: 12,           // total across all tabs
  badge_label: '12',
  default_tab: 'invoices',
  tabs: TabData[],
  items: []            // empty — client reads tabs[].items
}
```

**PillDrawer changes:**

```
// Inside the scroll area (L229), before the existing items render:

if (tabs && tabs.length > 0) {
  render:
    <TabStrip>  // horizontal row of tab buttons
      {tabs.map(tab => (
        <TabButton
          key={tab.id}
          active={activeTab === tab.id}
          onClick={() => setActiveTab(tab.id)}
          label={`${tab.label} (${tab.count})`}
        />
      ))}
    </TabStrip>
    <ItemList items={tabs.find(t => t.id === activeTab)?.items ?? []} />
} else {
  // existing flat items render (unchanged)
}
```

**Tab strip styling:** Segmented control (dark pill buttons, active = lighter background + white text, inactive = muted). Sits below the drawer header, above the scroll area. Fixed position (doesn't scroll with items).

```typescript
// Tab strip styles (inline, matching dark theme):
const tabStripStyle: CSSProperties = {
  display: 'flex',
  gap: 4,
  padding: '12px 16px 8px',
  borderBottom: '0.5px solid #38383a',
};

const tabBtnBase: CSSProperties = {
  fontSize: 12,
  fontWeight: 500,
  padding: '6px 12px',
  borderRadius: 8,
  border: 'none',
  cursor: 'pointer',
  whiteSpace: 'nowrap',
};

// Active: background #2c2c2e, color #f5f5f7
// Inactive: background transparent, color #636366
```

**State management:** Add `const [activeTab, setActiveTab] = useState<string | null>(null)` alongside existing PillDrawer state. On fetch completion, set `activeTab` to `response.default_tab` (or first tab if missing). Reset `activeTab` to null when drawer closes or pill changes.

### Sub-tab A: Invoices (overdue)

**Query:**

```sql
SELECT id, entity_name, invoice_type, amount_due, due_date, status, created_at
FROM invoices
WHERE status IN ('sent', 'overdue', 'jen_flagged')
ORDER BY due_date ASC NULLS FIRST
```

Reuse the `agingBucket()` logic from [invoices/aging/route.ts:14-19](src/app/api/invoices/aging/route.ts#L14-L19):

```typescript
function agingBucket(daysOutstanding: number): string {
  if (daysOutstanding <= 30) return 'current';
  if (daysOutstanding <= 60) return 'thirty_day';
  if (daysOutstanding <= 90) return 'sixty_day';
  return 'ninety_plus';
}
```

**Severity mapping:**

| Condition | Severity |
|---|---|
| 90+ days overdue OR status = 'jen_flagged' | `critical` |
| 60-89 days overdue | `warning` |
| < 60 days overdue | `info` |

**DrawerItem mapping:**

```typescript
{
  id: inv.id,
  severity: daysOutstanding >= 90 || inv.status === 'jen_flagged' ? 'critical'
          : daysOutstanding >= 60 ? 'warning' : 'info',
  title: `${inv.entity_name} — $${amount_due.toLocaleString()}`,
  description: `${inv.invoice_type === 'receivable' ? 'AR' : 'AP'} · ${inv.status} · ${daysOutstanding}d overdue`,
  entity_name: inv.entity_name,
  revenue_at_risk: inv.invoice_type === 'receivable' ? amount_due : 0,
  action_label: 'Review',
  action_prompt: `Show me invoice details for ${inv.entity_name} — $${amount_due.toLocaleString()} ${inv.invoice_type}, ${daysOutstanding} days overdue. What's the collection status and what should I do?`,
  created_at: inv.created_at,
}
```

### Sub-tab B: Reconciliation (direct query)

**Query:**

```sql
SELECT run_date, status, discrepancy_count, discrepancies,
       our_call_count, td_call_count, our_revenue, td_revenue
FROM reconciliation_runs
WHERE status = 'discrepancies_found'
ORDER BY run_date DESC
LIMIT 10
```

`reconciliation_runs` stores one row per date per run_type. The `discrepancies` column is a JSONB array of `ReconciliationDiscrepancy` objects (typed at [reconciliation/run/route.ts:17-29](src/app/api/reconciliation/run/route.ts#L17-L29)):

```typescript
interface ReconciliationDiscrepancy {
  buyer_name: string;
  our_call_count: number;
  td_converted_count: number;
  our_revenue: number;
  td_revenue: number;
  mismatched_cids: MismatchedCID[];
}
```

**Severity mapping:**

| Condition | Severity |
|---|---|
| Revenue diff > $500 for any buyer in discrepancies | `critical` |
| Any discrepancies exist | `warning` |
| Clean run | Not shown (filtered out by WHERE clause) |

**DrawerItem mapping:** One item per reconciliation run with discrepancies:

```typescript
{
  id: `recon-${run.run_date}`,
  severity: maxBuyerRevDiff > 500 ? 'critical' : 'warning',
  title: `Reconciliation ${run.run_date} — ${run.discrepancy_count} mismatches`,
  description: `Our: $${ourRev} / TD: $${tdRev} · ${discrepancies.length} buyer${discrepancies.length > 1 ? 's' : ''} with mismatches`,
  entity_name: null,
  revenue_at_risk: Math.abs(ourRev - tdRev),
  action_label: 'Review',
  action_prompt: `Walk me through the reconciliation for ${run.run_date}. Show the ${run.discrepancy_count} mismatched CIDs, which buyers are affected, and whether any of these are dispute-worthy.`,
  created_at: run.run_date + 'T00:00:00Z',
}
```

### Sub-tab C: Reports (stalled buyer reports)

**Query:**

```sql
SELECT id, buyer_name, filename, total_rows, status, created_at
FROM buyer_reports
WHERE status IN ('mapping', 'needs_mapping', 'auto_mapped')
  AND created_at < NOW() - INTERVAL '24 hours'
ORDER BY created_at ASC
```

"Stalled" = non-terminal status AND older than 24 hours. Reports in `mapping`/`needs_mapping`/`auto_mapped` that haven't progressed to `complete` or `error` within 24h are stuck.

**Severity mapping:**

| Condition | Severity |
|---|---|
| Stalled > 72h | `warning` |
| Stalled 24-72h | `info` |

No `critical` for reports — they're tasks, not emergencies.

**DrawerItem mapping:**

```typescript
{
  id: report.id,
  severity: stalledHours > 72 ? 'warning' : 'info',
  title: `${report.buyer_name} — ${report.filename}`,
  description: `${report.total_rows} rows · Status: ${report.status} · Stalled ${stalledHours}h`,
  entity_name: report.buyer_name,
  revenue_at_risk: 0,
  action_label: 'Process',
  action_prompt: `Show me the stalled buyer report for ${report.buyer_name} (${report.filename}). What's blocking it and what do I need to do to finish processing?`,
  created_at: report.created_at,
}
```

### Sub-tab D: Disputes summary

**Query:**

```sql
SELECT COUNT(*) as count,
       SUM(revenue_at_risk) as total_risk
FROM disputes
WHERE status IN ('detected', 'raised', 'pending_response', 'escalated')
  AND dispute_class = 'call_recovery'
```

Single aggregate row. Rendered as ONE summary DrawerItem that links to the Disputes pill, NOT a list of individual disputes (per Mark Q3 — "summary only").

**DrawerItem mapping:**

```typescript
{
  id: 'disputes-summary',
  severity: totalRisk > 5000 ? 'critical' : totalRisk > 0 ? 'warning' : 'info',
  title: `${count} open disputes — $${totalRisk.toLocaleString()} at risk`,
  description: 'Tap to open the full Disputes drawer for detail and actions.',
  entity_name: null,
  revenue_at_risk: totalRisk,
  action_label: 'Open disputes',
  action_prompt: 'Show me all open disputes ranked by revenue at risk. Walk through the top 5 and recommend next steps for each.',
  created_at: new Date().toISOString(),
}
```

**NOTE:** The "Open disputes" action dispatches to Milo chat (same as all drawer items). It does NOT programmatically open the Disputes pill drawer. If Mark wants a direct cross-pill link, that requires a PillDrawer enhancement (dispatch `onPillOpen('disputes')` callback). Defer to v4.1.

### Default tab logic (inside `fetchAccountingPill`)

```typescript
function pickDefaultTab(tabs: TabData[]): string {
  // Pick the tab with the highest-severity item.
  // Severity rank: critical=3, warning=2, info=1
  const rank = { critical: 3, warning: 2, info: 1 };
  let best = tabs[0]?.id ?? 'invoices';
  let bestScore = 0;

  for (const tab of tabs) {
    for (const item of tab.items) {
      const score = rank[item.severity] ?? 0;
      if (score > bestScore) {
        bestScore = score;
        best = tab.id;
      }
    }
  }
  return best;
}
```

### fetchAccountingPill implementation shape

```typescript
async function fetchAccountingPill(): Promise<{
  items: DrawerItem[];
  count: number;
  badge_label: string;
  tabs: TabData[];
  default_tab: string;
}> {
  // Run 4 queries in parallel
  const [invoiceResult, reconResult, reportsResult, disputesSummary] =
    await Promise.all([
      fetchInvoicesTab(),
      fetchReconTab(),
      fetchReportsTab(),
      fetchDisputesSummaryTab(),
    ]);

  const tabs: TabData[] = [
    { id: 'invoices', label: 'Invoices', count: invoiceResult.length, items: invoiceResult },
    { id: 'reconciliation', label: 'Reconciliation', count: reconResult.length, items: reconResult },
    { id: 'reports', label: 'Reports', count: reportsResult.length, items: reportsResult },
    { id: 'disputes', label: 'Disputes', count: disputesSummary.length, items: disputesSummary },
  ];

  const totalCount = tabs.reduce((s, t) => s + t.count, 0);
  const defaultTab = pickDefaultTab(tabs);

  return {
    items: [],
    count: totalCount,
    badge_label: String(totalCount),
    tabs,
    default_tab: defaultTab,
  };
}
```

### Pill definition (operator-pills.ts)

```typescript
// Add to billing role (before existing overdue_invoices):
{ id: 'accounting', label: 'Accounting', query: 'Show me the accounting overview — overdue invoices, reconciliation mismatches, stalled reports, and dispute exposure.', focus_area_tag: 'billing' },

// Add to admin role:
{ id: 'accounting', label: 'Accounting', query: 'Show me the accounting overview — overdue invoices, reconciliation, reports, and disputes.', focus_area_tag: 'billing' },
```

### Acceptance criteria

1. Pill appears in PillBar for billing and admin roles
2. Pill badge shows total item count across all 4 tabs
3. Tapping pill opens PillDrawer with tab strip at top: Invoices | Reconciliation | Reports | Disputes
4. Each tab header shows its count in parentheses: "Invoices (3)"
5. Default active tab is whichever has the highest-severity item
6. Zero-count tabs still render (not hidden) — operator sees "Nothing in this queue" per-tab
7. **Invoices tab:** overdue/flagged invoices ranked by aging bucket. Severity from days overdue (90+ critical, 60+ warning). Revenue_at_risk = amount_due for receivables.
8. **Reconciliation tab:** recent recon runs with discrepancies. Revenue_at_risk = |our_revenue - td_revenue|. One item per run date.
9. **Reports tab:** stalled buyer reports (non-terminal status, older than 24h). Info/warning severity only.
10. **Disputes tab:** single summary item showing count + total $ at risk. "Open disputes" dispatches to Milo chat.
11. Tab strip is visually a segmented control (dark pill buttons, active = lighter).
12. Tab strip is fixed (doesn't scroll with items). Items scroll below it.
13. Switching tabs is instant (all data fetched on drawer open, tabs are client-side filter only).
14. Snooze button: **skip for Invoices, Reconciliation, Reports items** (same reasoning as Sales — snooze only works on alert IDs). Show "Review" / "Process" / "Open disputes" as the only action. Disputes summary item has no snooze.

### Snooze handling across non-alert pills

PillDrawer currently shows "Snooze 24h" on every row ([PillDrawer.tsx:323-327](src/components/operator/PillDrawer.tsx#L323-L327)). The snooze handler POSTs to `/api/briefing/action` with `alert_id`, which only works for alert-sourced items.

**For v4.0:** Hide the Snooze button when the item doesn't come from the alerts table. Two approaches:

- **(a) Add `snoozable: boolean` to DrawerItem interface.** Alert-sourced items set `snoozable: true`, non-alert items set `snoozable: false`. PillDrawer conditionally renders the Snooze button. **Recommended.**
- **(b) Infer from pill ID.** If pill is in `DIRECT_SOURCE_PILLS`, hide snooze. Simpler but less flexible — some direct-source pills might want snooze later.

**Recommendation:** Option (a). Add `snoozable?: boolean` to DrawerItem (defaults true for backward compat). Set `false` in Sales and Accounting fetch functions. One-line conditional in PillDrawer render.

---

## Build Order for Coder-2

1. **PillDrawer tab support + snoozable flag** — prerequisite for Accounting. Also unblocks Sales (snoozable: false). ~50 lines in PillDrawer.tsx, ~2 lines in DrawerItem interface.
2. **Sales pill** — straightforward single-table query. Proves non-alert pills render correctly with snoozable=false.
3. **Accounting pill** — 4 parallel queries, tabbed response. Most complex. Ship last.

Estimated: 3 commits, sequential dependency (1 → 2 → 3).
