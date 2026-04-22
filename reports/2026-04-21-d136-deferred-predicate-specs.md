# D136: Deferred Predicate Specs — Contracts, Sales, Accounting

> Coder-3 Research | Directive #136 | Spec only — no code changes
> Prerequisite: D134 ships Q&A + Disputes + Aging + Calls retune (Coder-1)

---

## Spec 1: Contracts Drawer

### Rendering decision: LIST OF CONTRACTS (not flat issues)

**Pick:** Each DrawerItem = one contract (latest version per `contract_group_id`). Tapping "Review" opens Milo chat with the contract's flagged-clause context, which A1 IssueCard components will render inside the drawer in a future iteration.

**Rationale:** The operator's workflow is contract-centric, not clause-centric. When Mark opens the Contracts pill, his mental model is "which counterparties have contracts I need to act on?" — not "show me 47 flagged clauses across 8 contracts." A flat issue list loses the grouping that matters (which contract, which counterparty, what stage). The existing `/api/contract/documents` route already groups by `contract_group_id` and returns `flaggedCount` + `clauseDecisions` per version — the drawer summarizes at contract level, and drill-down to clause level happens via the Review action.

Once A1 IssueCard components wire into the drawer (per D85 amendment), each contract row can expand inline to show its clauses. That's a v4.1 enhancement on top of a v4.0 contract list.

### risk_level → severity mapping

`contract_documents.risk_level` has 3 stored values (from `analysis_result.summary.overall_risk`). `DrawerItem.severity` has 3 values. Mapping:

| contract_documents.risk_level | DrawerItem.severity | Rationale |
|---|---|---|
| `HIGH` or `CRITICAL` | `critical` | Red dot. Operator must act before signing. |
| `MEDIUM` | `warning` | Amber dot. Review recommended but not blocking. |
| `LOW` or null | `info` | Blue dot. Clean or unanalyzed. |

Note: `FlaggedClause.riskLevel` has 5 levels (critical/high/medium/low/favorable) but that's per-clause, not per-contract. The drawer uses the contract-level `risk_level` which is already reduced to 3 tiers by the analysis pipeline.

### revenue_at_risk

**Pick:** Option (a) — `revenue_at_risk: 0`.

**Rationale:** Contract documents don't carry a dollar value in any current column. `analysis_result` stores flagged clauses and risk assessments but no monetary figure. Deriving from terms metadata (option b) would require parsing clause text for dollar amounts — unreliable and inconsistent. Pulling from `analysis_records` (option c) requires a cross-table join that doesn't exist today and the analysis pipeline doesn't compute contract value.

The absence of `$X at risk` in the drawer is actually correct for contracts: the risk is legal/operational, not a dollar figure. The severity dot (red/amber/blue) communicates urgency. If Mark later wants dollar values, the right path is adding a `deal_value` column to `contract_documents` and populating it during entity linking.

### Data shape from pill route

```typescript
// GET /api/operator/pill?pill=contracts_review
// Source: contract_documents, grouped by contract_group_id, latest version per group
{
  pill: 'contracts_review',
  count: 4,           // number of contract groups with non-terminal status
  badge_label: '4',
  items: [
    {
      id: 'cd_uuid',                              // latest version id
      severity: 'critical',                        // from risk_level mapping
      title: 'Acme Corp — MSA v3',                // counterparty_name + document_type + version
      description: '6 flagged clauses (2 critical, 1 high). Status: redlined.',
      entity_name: 'Acme Corp',                   // counterparty_name
      revenue_at_risk: 0,
      action_label: 'Review',
      action_prompt: 'Show me the contract analysis for Acme Corp MSA v3 — walk through the critical and high-risk clauses and recommend next steps.',
      created_at: '2026-04-20T...'
    }
  ]
}
```

**Query:** `contract_documents` WHERE `status` IN ('uploaded', 'analyzing', 'analyzed', 'redlined', 'pending_signature') — excludes terminal states (signed, voided, expired). Group by `contract_group_id`, take latest version. Parse `analysis_result.flagged_clauses` for count/severity breakdown in description.

### Files to modify

| File | Change |
|---|---|
| `src/app/api/operator/pill/route.ts` | Add `contracts_review` branch: query `contract_documents`, group by contract_group_id, map to DrawerItem[] |
| `src/lib/operator-pills.ts` | Verify `contracts_review` pill exists in admin role (it does — line 65) |

### Open questions for Mark

None. The rendering decision (contracts, not flat issues) and revenue_at_risk=0 are both derivable from the data shape and operator workflow.

---

## Spec 2: Sales Drawer

### "Interesting" filter definition

**Default filter:** Entities that need operator attention, defined as the union of:

1. **Stalled** — `daysInStage > threshold` per stage:
   - outreach: > 14 days (should have made contact by now)
   - qualifying: > 21 days (decision overdue)
   - drip: > 30 days (drip sequence exhausted)
   - onboarding: > 14 days (paperwork stuck)
   - activation: > 7 days (should be live by now)
   - active/dormant/blacklisted: excluded (terminal or steady-state)

2. **Recently changed** — `stage_changed_at` within last 48 hours. These are entities that just moved stages — operator should verify the transition is clean.

3. **Milo recommendation** — entities with an unresolved `action_item` of type `pipeline_action` in `ai_action_log`. This captures when Milo's morning briefing or alert system flagged a pipeline entity.

**Severity from daysInStage:**

| Condition | severity | Rationale |
|---|---|---|
| Stalled > 2× threshold (e.g., qualifying > 42d) | `critical` | Red — entity is stuck, losing momentum |
| Stalled > 1× threshold | `warning` | Amber — needs nudge |
| Recently changed (< 48h) | `info` | Blue — informational, verify transition |
| Milo recommendation | `warning` | Amber — Milo flagged for a reason |

**Configurable thresholds:** Store in a `SALES_STALL_THRESHOLDS` constant in the pill route (not user-configurable in v4.0). Can promote to `user_profiles.pill_config` JSONB in v4.1 if operators want different sensitivity.

### revenue_at_risk

**Pick:** 0 for v4.0. `prospect_pipeline` has no `deal_value` column. Pipeline entities are publishers and buyers — their value is ongoing revenue, not a deal amount. Computing "revenue at risk from losing this entity" would require joining `call_records` for 30d revenue by entity, which is expensive and conceptually different from the `revenue_at_risk` field (which means "dollars currently at risk from inaction").

If Mark wants pipeline revenue context, the description line can include "7d revenue: $X" derived from the optional `call_records` join the pipeline route already supports (`include7d=true`). This is a description enhancement, not a `revenue_at_risk` value.

### Data shape from pill route

```typescript
// GET /api/operator/pill?pill=sales_pipeline
{
  pill: 'sales_pipeline',
  count: 7,
  badge_label: '7',
  items: [
    {
      id: 'pp_uuid',
      severity: 'warning',
      title: 'LeadSource Media — qualifying (28d)',
      description: 'Publisher. Stalled 28 days in qualifying (threshold: 21d). Last activity: drip email sent.',
      entity_name: 'LeadSource Media',
      revenue_at_risk: 0,
      action_label: 'Follow up',
      action_prompt: 'Show me the full story on LeadSource Media — why are they stalled in qualifying for 28 days? Pull recent activity and recommend whether to push forward, drip, or drop.',
      created_at: '2026-03-25T...'
    }
  ]
}
```

**Query:** `prospect_pipeline` WHERE `stage` NOT IN ('active', 'dormant', 'blacklisted'). Compute `daysInStage` from `stage_changed_at`. Filter to stalled OR recently changed. Optionally join `ai_action_log` for Milo recommendations. Sort by severity desc, then daysInStage desc.

### Files to modify

| File | Change |
|---|---|
| `src/app/api/operator/pill/route.ts` | Add `sales_pipeline` branch: query `prospect_pipeline`, compute stall thresholds, map to DrawerItem[] |
| `src/lib/operator-pills.ts` | Add `sales_pipeline` pill to outreach and prospecting roles (or admin). Current `pipeline_summary` and `kanban` pills are chat-only — `sales_pipeline` would be the drawer-capable equivalent. |

### Open questions for Mark

**Q1.** The stall thresholds (14d outreach, 21d qualifying, 30d drip, 14d onboarding, 7d activation) — are these reasonable for the PPC vertical, or does Mark want different numbers?

**Q2.** Should the Sales drawer show ALL non-terminal entities (like a mini pipeline view in the drawer), or only the "needs attention" subset? Current spec shows attention-only. A full list would be 94 entities, which is the firehose D85 wants to avoid.

---

## Spec 3: Accounting Drawer (with Billing merge)

### Tab layout decision

**Pick:** Per-category wrapper component inside existing PillDrawer — NOT a tabbed PillDrawer variant.

**Rationale:** Modifying PillDrawer to support tabs would change its API surface for all 8 pills. Only Accounting needs tabs. Instead: the Accounting pill returns a response with a `tabs` field, and the operator page renders a thin `AccountingDrawer` wrapper inside `PillDrawer`'s content area. The wrapper renders tab headers (styled as a segmented control strip at the top of the scroll area) and filters DrawerItems by tab.

This means PillDrawer stays generic (renders `items[]`), and the Accounting-specific tab logic lives in a single new component. Other pills are unaffected.

**Implementation sketch:**

```
PillDrawer (unchanged shell: overlay, header, close, scroll area)
  └── if pill === 'accounting':
        AccountingTabBar (segmented strip: Invoices | Reconciliation | Reports | Disputes)
        DrawerItem[] filtered by active tab
      else:
        DrawerItem[] (current behavior)
```

The branching can happen in PillDrawer's render body (a 4-line conditional) or in the operator page (pass a `renderContent` prop). The simplest path is in PillDrawer itself — it already knows `pillId`.

### Sub-tabs

| Tab | Source table | Filter | Severity mapping |
|---|---|---|---|
| **Invoices** | `invoices` | status IN ('sent', 'overdue', 'jen_flagged') | overdue 90+ → critical, overdue 60+ → warning, sent/flagged → info |
| **Reconciliation** | alerts (existing) | title matches /reconciliation\|mismatch\|discrepancy/i | Uses alert type directly (already critical/warning) |
| **Reports** | `buyer_reports` | status IN ('uploaded', 'mapped', 'processing') — unfinished reports needing action | All → info (reports are tasks, not emergencies) |
| **Disputes** | `disputes` | status IN ('detected', 'raised', 'pending_response', 'escalated') | Has native `revenue_at_risk`. High $ → critical, medium → warning, low → info |

**Why Reconciliation stays alert-based:** The reconciliation cron (`/api/cron/reconciliation`) already generates deduped alerts for mismatches ([reconciliation/route.ts:25-41](src/app/api/reconciliation/route.ts#L25-L41)). Building a separate reconciliation drawer query duplicates logic. The alert-based approach is correct here — reconciliation mismatches ARE alerts.

**Why Disputes appears in both the Disputes pill AND the Accounting drawer:** The Disputes pill shows the full dispute lifecycle (operator manages disputes). The Accounting drawer's Disputes tab shows a summary view (CFO sees revenue at risk across disputes as part of the financial picture). Different audience, different depth. The Disputes tab in Accounting shows fewer columns and doesn't expose raise/respond actions.

### Top-level severity/order across sub-tabs

The pill badge and the default tab are driven by the highest-severity item across all tabs:

1. **Badge:** Show total count across all tabs. Badge label format: count (e.g., "12"), not dollars — Accounting is heterogeneous, a single dollar figure would be misleading.

2. **Default active tab on open:** Whichever tab contains the highest-severity item. If Invoices has a 90+ day overdue (critical) and Reconciliation has a warning, open to Invoices.

3. **Sort within each tab:** Same as all drawers — severity desc, then revenue_at_risk desc, then created_at desc.

4. **Tab badge counts:** Each tab header shows its own count: `Invoices (3) | Reconciliation (1) | Reports (0) | Disputes (8)`. Zero-count tabs still render (not hidden) — the operator should see that reconciliation is clean.

### Data shape from pill route

```typescript
// GET /api/operator/pill?pill=accounting
{
  pill: 'accounting',
  count: 12,
  badge_label: '12',
  tabs: [
    {
      id: 'invoices',
      label: 'Invoices',
      count: 3,
      items: [DrawerItem, DrawerItem, DrawerItem]
    },
    {
      id: 'reconciliation',
      label: 'Reconciliation',
      count: 1,
      items: [DrawerItem]
    },
    {
      id: 'reports',
      label: 'Reports',
      count: 0,
      items: []
    },
    {
      id: 'disputes',
      label: 'Disputes',
      count: 8,
      items: [DrawerItem, ...]
    }
  ],
  default_tab: 'disputes',  // tab with highest-severity item
  items: []                  // empty — client reads from tabs[].items
}
```

The `items: []` top-level field keeps backward compatibility with `PillDrawer`'s existing fetch logic. The Accounting branch in the drawer checks for `tabs` and renders accordingly.

### Files to modify

| File | Change |
|---|---|
| `src/app/api/operator/pill/route.ts` | Add `accounting` branch: 4 parallel queries (invoices, alerts/reconciliation, buyer_reports, disputes), merge into tabbed response |
| `src/components/operator/PillDrawer.tsx` | Add ~20-line conditional: if response has `tabs`, render `AccountingTabBar` + filtered items. All other pills unchanged. |
| `src/lib/operator-pills.ts` | Add `accounting` pill to billing role. Existing `overdue_invoices`, `weekly_billing`, `ap_aging`, `td_balance`, `disputes` pills remain as chat shortcuts — `accounting` is the drawer-capable superset. |

### Open questions for Mark

**Q1.** Should the Accounting drawer's Disputes tab show the same data as the Disputes pill (full lifecycle) or a reduced "revenue summary" view? Current spec says reduced — summary only, no raise/respond actions. If Mark wants full depth, the tab becomes redundant with the Disputes pill.

**Q2.** `buyer_reports` tab — is "unfinished reports" (uploaded but not yet processed) the right filter? Or should it show recent completed reports with discrepancies? The current `/reports` page is a CSV upload + mapping tool — unclear if that workflow belongs in a drawer.

**Q3.** Does the Reconciliation tab need to show anything beyond what already surfaces as alerts? If not, it's purely a filtered view of the existing alert-based drawer — no new data source needed.
