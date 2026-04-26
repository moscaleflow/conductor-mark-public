# Aging / AP / AR Landscape Audit for Phase 2 Drawer Spec

**Coder-3 Research | 2026-04-25**
**Scope:** Pure audit. No code. Map the existing AP/AR/aging infrastructure so Surface 11 (Aging drawer) design matches actual primitive capabilities.
**Drift guard:** Every capability claim below cites a file path. Every gap is marked explicitly. Nothing is guessed.

---

## Executive Summary

The billing infrastructure is **far more mature than you'd guess** — full invoice lifecycle (draft→Jen review→Tiffani approval→sent→paid), a `/api/invoices/aging` endpoint with 4-bucket breakdown, automated invoice generation via daily cron, bank transaction CSV import with AI-powered categorization, and 6 Milo operator tools for billing work. **However, the system is internally complete but externally disconnected**: it can compute what's owed and match what's been paid, but it cannot send an invoice email, trigger a payment, or push data to any external financial system. The human operators (Jen for billing review, Tiffani for payment approval/recording, Mark for actual payment execution) are load-bearing in every step after invoice creation. There is no `@milo/aging` or `@milo/billing` package — everything lives in milo-for-ppc. No three-stance honesty rule exists in the decision log; if the drawer needs one, it must be created fresh.

---

## 1. Primitives Inventory

### @milo/aging — DOES NOT EXIST
### @milo/invoicing — DOES NOT EXIST
### @milo/billing — DOES NOT EXIST

All 8 milo-engine packages: `ai-client`, `blacklist`, `contract-analysis`, `contract-negotiation`, `contract-signing`, `crm`, `feedback`, `onboarding`. Zero aging/billing/invoicing packages.

### Aging-related logic in other @milo/* packages

| Package | Aging/billing content | Detail |
|---|---|---|
| `@milo/contract-analysis` | String constants only | Base registry rules detect payment terms and billing audit clauses in contracts. Not financial logic. |
| `@milo/contract-signing` | Zero | No invoice, billing, payment, or aging references. |
| `@milo/contract-negotiation` | Zero | Same. |
| `@milo/crm` | Migration metadata only | PPCRM migration script packs `billing_type`, `payment_model`, `payment_terms`, `billing_cycle`, `net_terms`, `min_invoice_threshold` into `metadata` JSONB on counterparties. NOT first-class typed fields. Zero billing logic in CRM src/. |

### Where ALL aging/billing logic lives: `milo-for-ppc`

**Tables (shared Supabase `tappyckcteqgryjniwjg`):**

**`invoices`** — Foundation schema, 30+ columns:
```
id UUID, invoice_type (receivable|payable), entity_type (publisher|buyer),
entity_name, prospect_id FK, billing_cycle (weekly_net_7|weekly_net_10|biweekly|
monthly_net_30|monthly_net_35|monthly_net_45|custom), period_start DATE,
period_end DATE, total_calls, qualified_calls, flagged_calls, under_duration_calls,
rate NUMERIC, subtotal, adjustments, adjustment_notes, amount_due NUMERIC NOT NULL,
status (draft|pending_jen|jen_approved|jen_flagged|pending_tiffani|sent|paid|
overdue|disputed|void), jen_reviewed_at, jen_note, tiffani_approved_at,
tiffani_note, due_date DATE, paid_date DATE, paid_amount NUMERIC, call_ids JSONB,
td_comparison JSONB, dispute_details JSONB, created_at, updated_at,
billing_schedule_id FK, auto_generated BOOLEAN, expected_amount, variance_pct,
variance_flag, cleared_to_pay BOOLEAN
```

**`billing_schedules`** — Per-entity billing automation config:
```
id UUID, entity_name, entity_type (publisher|buyer), invoice_type (receivable|payable),
billing_cycle (weekly|biweekly|monthly), billing_day INT, payment_terms INT DEFAULT 7,
auto_send BOOLEAN DEFAULT false, variance_threshold NUMERIC DEFAULT 10.0,
avg_amount, avg_payment_days, last_invoice_date, next_invoice_date, notes,
created_at, updated_at
```

**`bank_transactions`** — CSV-imported bank records:
```
id UUID, transaction_date DATE, description, amount NUMERIC, transaction_type
(credit|debit), category (buyer_payment|publisher_payout|td_recharge|va_payroll|
owner_draw|business_expense|uncategorized), category_confidence NUMERIC,
matched_invoice_id FK, matched_entity_name, match_method (auto|manual|suggested),
matched_at, bank_account DEFAULT 'us_bank_checking', raw_csv_row JSONB,
imported_at, reviewed_by, reviewed_at
```

**Indexes on invoices:** `idx_invoices_status`, `idx_invoices_entity`, `idx_invoices_type`, `idx_invoices_due` (partial, status IN sent/overdue), `idx_invoices_period`.

---

## 2. Existing UI / Features

### milo-for-ppc

| Surface | File | What it does |
|---|---|---|
| **`/invoices` page** | `src/app/invoices/page.tsx` (701 lines) | Draft invoice management. Generate form (period picker, billing cycle, entity type filter), AP/AR filter toggle, TD match/mismatch filter, invoice table with expandable detail (call breakdown, TD comparison, adjustments), batch "Send to Jen" action, regenerate action. |
| **JenView (`/jen`)** | `src/components/JenView.tsx` (887 lines) | Invoice review queue (pending_jen). InvoiceCard with call breakdown, TD verification grid, per-call drill-down. Actions: Approve / Flag Issue. **BankImportSection**: CSV drag-drop, auto-categorize, suggested match review (Accept/Reject/Recategorize), learned pattern intelligence, amount anomaly warnings. |
| **TiffaniView (`/tiffani`)** | `src/components/TiffaniView.tsx` (430 lines) | Four sections: Ready to Send (jen_approved), Sent — Awaiting Payment (with days-until-due), Overdue (red border, days-overdue count), Paid. Actions: Send Payment, Record Payment (amount, date, reference, note). |
| **BriefingPanel** | `src/components/BriefingPanel.tsx` | Morning brief surfaces cashflow items with `send_to_jen` action. NeedsYouItem carries `invoice_ids`, `invoice_total`, `invoice_count`. |
| **`/reconciliation` page** | `src/app/reconciliation/page.tsx` | Buyer-by-buyer revenue reconciliation (Milo vs TrackDrive). Per-CID drill-down on mismatches. Not aging-specific but billing-adjacent. |

### milo-ops (duplicate)

| Surface | Status |
|---|---|
| `/invoices` page | **Identical 701-line copy** of milo-for-ppc. Pre-extraction duplicate. |
| `ApAgingBlock` (dashboard-v2) | 79 lines. 4-bucket AP aging with proportional bar chart. Current (green) / 31-60d (yellow) / 61-90d (orange) / 90+ (red). |
| `InvoiceBlock` (dashboard-v2) | 117 lines. Overdue invoices list with entity name, AR/AP type, days overdue, amount. Color-coded severity. |
| `WeeklyBillingBlock` (dashboard-v2) | 177 lines. AR vs AP for a period with This Week / Last Week / This Month tabs. |
| `BillingPipelineBlock` (dashboard-v2) | 121 lines. Full invoice lifecycle as horizontal flow: Pending Review → Ready to Send → Sent → Paid → Cleared to Pay. Stage counts + clickable links. |
| `billing_snapshot` / `td_balance` blocks | **Stubbed — render "Coming soon"** |

### No dedicated `/aging` page exists anywhere

Aging data is surfaced through dashboard blocks, the TiffaniView overdue section, BriefingPanel, and the `/api/invoices/aging` endpoint — but there is no standalone aging page. **This is the gap Surface 11 fills.**

### MOP / PPCRM legacy

`ppcrm-client.ts` exposes 4 billingbot functions: `billingbotDashboard()` (AP/TBP/AR/net position), `billingbotAging()` (aging receivables with severity + blocked publisher invoice counts), `billingbotReconcile()`, `billingbotGenerate()`. The operator executive route tries PPCRM billingbot first, falls back to local Supabase. PPCRM has richer data (pay-if-paid enforcement, blocked publisher counts) that local fallback does not replicate.

---

## 3. Data Feeds

### How invoice data gets INTO the system

| Path | Trigger | Status lands at | Source |
|---|---|---|---|
| **Manual generation** | POST `/api/invoices/generate` (from `/invoices` page) | `pending_jen` | `call_records` aggregation |
| **Automated cron** | `/api/cron/billing-prepare` (6 AM Denver, weekdays) | `pending_jen` | `billing_schedules` + `call_records` |
| **Milo tool** | `generate_invoice` (Tool 8A) | `pending_jen` | `call_records` aggregation |

**All three paths derive amounts from `call_records`** — there is NO direct import of external invoice data.

### TrackDrive billing sync

TrackDrive is the **sole upstream data source** for all billing numbers:
1. **Hourly call sync** (`/api/sync/route.ts`): pulls calls from TD API → `call_records`. Each row stores `revenue` (AR), `payout` (AP), `original_td_payout`, `td_buyer_revenue`, `billing_type`, `is_converted`.
2. **Payout backfill** (sync step 4): corrects CPA $0 payout anomalies ("Vasdar prevention").
3. **TD change detection** (cron): monitors rate changes, unconversions, config drift. Generates alerts.
4. **Reconciliation** (cron + manual): compares `call_records` vs TD API for last 7 days. Discrepancies trigger backfill.
5. **TD comparison on invoices**: every generated invoice includes `td_comparison` JSONB cross-checking sums.

**There is NO billing-summary import from TD.** All billing is computed from individual call_records rows.

### Publisher payable (AP) sources

Publisher payables = SUM of `call_records.payout` for qualified calls in billing period. `generatePublisherInvoice()` in `src/lib/invoice-generate.ts` (line 150). For RTB, uses `original_td_payout`. For CPA, uses the IO-agreed rate corrected at sync.

### Buyer receivable (AR) sources

Buyer receivables = SUM of `call_records.revenue` for qualified calls. `generateBuyerInvoice()` in `src/lib/invoice-generate.ts` (line 266). Cross-validated against `td_buyer_revenue`.

### Bank transaction import

CSV upload via JenView's BankImportSection → POST `/api/bank-transactions/import`. Auto-categorized via POST `/api/bank-transactions/categorize` (AI-powered). Categories: `buyer_payment`, `publisher_payout`, `td_recharge`, `va_payroll`, `owner_draw`, `business_expense`, `uncategorized`. Matched to invoices by amount tolerance ($1) + date window (7 days) + name similarity.

---

## 4. Automated Capabilities — What Milo Can ACTUALLY Do

### CAN DO (Stance B — Milo executes fully)

| Action | How | File |
|---|---|---|
| Generate AR/AP invoice | Creates `invoices` row + PDF URL | `src/lib/tools/generate-invoice.ts` |
| Check publisher invoice against data | Read-only comparison, PASS/FLAG/REJECT | `src/lib/tools/check-publisher-invoice.ts` |
| Match bank transactions to invoices | Persists to `reconciliation_runs` | `src/lib/tools/reconcile-bank-transactions.ts` |
| Draft dispute message | Creates `disputes` row with `is_draft=true` | `src/lib/tools/draft-dispute-message.ts` |
| Surface overdue alerts | Cron transitions to `overdue`, emits alerts | `/api/cron/evaluate/route.ts` |
| Compute aging buckets | 4-bucket breakdown from invoice data | `/api/invoices/aging/route.ts` |
| Generate morning billing brief | Role-aware brief with overdue counts | `src/lib/tools/morning-priority-briefing.ts` |

### CANNOT DO (no programmatic path exists)

| Action | Status | What's missing |
|---|---|---|
| **Send invoice email** | DOES NOT EXIST | No email templates, no Resend wiring for invoices. `auto_send` column on billing_schedules exists but nothing reads it. |
| **Send payment nudge / reminder** | DOES NOT EXIST | No nudge templates, no collection email pipeline. |
| **Mark invoice as paid** | HUMAN ONLY | PATCH `/api/invoices/[id]/record-payment` exists but is only triggered by human (TiffaniView). No tool wraps it. |
| **Write off a receivable** | DOES NOT EXIST | No write-off/bad-debt status, no write-off tool, no accounting journal logic. |
| **Trigger payment to publisher** | DOES NOT EXIST | No ACH, Mercury, Wise, Stripe, or any payment processor integration. |
| **Void an invoice** | PARTIAL | Status `void` exists in schema. Regenerate action voids implicitly. No standalone void tool. |
| **Send to external accounting** | DOES NOT EXIST | No QuickBooks, Xero, or accounting system integration. |
| **Live bank feed** | DOES NOT EXIST | Bank transactions require CSV upload. Plaid integration stubbed as empty array. |

### Boundary: Milo executes vs operator acts elsewhere

```
MILO EXECUTES:                    OPERATOR ACTS IN EXTERNAL SYSTEM:
├── Generate invoice               ├── Send invoice (email/portal)
├── Check publisher invoice        ├── Record payment received
├── Match bank transactions        ├── Execute publisher payment (Mercury/bank)
├── Draft dispute message          ├── Write off bad debt
├── Flag overdue                   ├── Push to QuickBooks
├── Compute aging report           └── Download bank CSV for import
└── Generate PDF
```

---

## 5. Operator Workflow Reality

### When a buyer is 30 days late (current workflow)

1. **Cron detects overdue**: `/api/cron/evaluate` transitions invoice status to `overdue`, generates alert
2. **Morning brief surfaces it**: BriefingPanel shows "N overdue invoices, $X total" in NeedsYou section
3. **Operator (Tiffani/Fab) sees overdue in TiffaniView**: Red border, days-overdue count
4. **Operator takes action OUTSIDE Milo**: Sends follow-up email manually (Teams, email, phone call). No template, no send mechanism in Milo.
5. **When payment arrives**: Operator downloads bank statement CSV, uploads to JenView BankImportSection, AI matches transaction to invoice, operator confirms match
6. **Record payment**: Tiffani clicks "Record Payment" in TiffaniView, enters amount/date/reference

**Milo's role: detection + alerting + data. Human's role: collection + payment recording.**

### When Mark needs to pay a publisher (current workflow)

1. **Cron generates AP invoice**: billing-prepare creates publisher invoice from call_records
2. **Jen reviews**: InvoiceCard in JenView, cross-checks against TD data, approves
3. **Cleared-to-pay rule** (HUMAN-MANAGED, not codified in app): Publisher gets paid ONLY after corresponding buyer AR payment clears. This back-to-back constraint is documented in Jen's skill doc but not enforced in code. The `cleared_to_pay` BOOLEAN on invoices exists but is set during bank reconciliation, not as a payment gate.
4. **Tiffani approves**: Moves to "Ready to Send" in TiffaniView
5. **Mark executes payment**: Logs into Mercury/bank and initiates wire/ACH. No payment automation in Milo.
6. **Operator records**: Tiffani records the payment in TiffaniView

**The cleared-to-pay rule is the most important AP business logic and it's not enforced in code.**

### Billing cadence (from production data)

| Cycle | Entities | Invoiced |
|---|---|---|
| Weekly | MediaRite, LeadLum, other weekly buyers | Every Monday |
| Biweekly | TruePath, 1st Choice, Actual Sales, Aragon, Esta, IClick, Aware Ads, B-452, Motiv8, DDR/Turtle, PPC, CrossBlade, CallCore, Dependable, Naked Media, OfferGlobe | 1st and 16th |
| Monthly | ELocal | End of month |
| CPA/CPQL weekly | MediaRite SSDI CPA, TruePath CPA/CPQL, LeadLum CPA/CPQL | Every Monday (converted calls only) |

### Team involvement

Billing is distributed across the operator team:
- **Jen**: Invoice review, flag issues, bank import + reconciliation, call-level verification
- **Tiffani**: Payment approval, send to buyer, record incoming payments, overdue tracking
- **Fab**: Collections follow-up, dispute escalation
- **Mark**: Actual payment execution (Mercury/bank login), strategic payment decisions
- No "Filipino VA" billing term found — the VA team IS the operator team (Jen, Tiffani, Fab, etc.)

---

## 6. Integrations Gap Analysis

### What exists

| Integration | Status | File |
|---|---|---|
| **TrackDrive call sync** | ACTIVE | `/api/sync/route.ts` — hourly pull of calls with revenue/payout |
| **TrackDrive reconciliation** | ACTIVE | `/api/cron/reconciliation/route.ts` — 7-day comparison |
| **TrackDrive change detection** | ACTIVE | `/api/cron/td-change-detection/route.ts` — rate/config monitoring |
| **Bank CSV import** | ACTIVE | `/api/bank-transactions/import/route.ts` — manual CSV upload |
| **AI transaction categorization** | ACTIVE | `/api/bank-transactions/categorize/route.ts` — learned patterns |
| **PPCRM billingbot bridge** | ACTIVE (legacy) | `ppcrm-client.ts` — aging + dashboard from frozen PPCRM |
| **Plaid bank feed** | STUB | `operator/executive/route.ts` L1075-1088 — returns empty arrays, comment says "wire when configured" |
| **QuickBooks** | MIGRATION FIELD ONLY | PPCRM migration script has `quickbooks_id` — no live integration |

### What would need to exist for autonomous AP/AR

| Integration | What it enables | Effort |
|---|---|---|
| **Resend invoice email templates** | Milo sends invoices to buyers, sends payment reminders | M — templates + send route + email queue |
| **Resend nudge/collection templates** | Automated overdue reminders at 7/14/30/60 day marks | M — templates + escalation schedule |
| **Mercury API** | Programmatic publisher payments, live balance check | L — OAuth + payment API + approval workflow |
| **Plaid** | Live bank feed (replaces CSV upload) | M — Plaid Link + webhook handler |
| **QuickBooks Online API** | Two-way ledger sync (invoices + payments) | L — OAuth + invoice/payment/chart-of-accounts mapping |
| **Write-off accounting logic** | Bad debt journal entries | S — new status + accounting rules |
| **Cleared-to-pay enforcement** | Codify the back-to-back payment rule in app logic | M — AR payment → AP cleared_to_pay gate |

### Minimum viable set for Phase 2 Aging drawer (v1)

The drawer can ship with **zero new integrations** if it is designed around what Milo can already do:

1. **Read aging data** — `/api/invoices/aging` already returns 4-bucket AR + AP breakdown ✓
2. **Surface overdue invoices** — status flow + cron detection already work ✓
3. **Generate invoices on demand** — `generate_invoice` tool exists ✓
4. **Draft collection messages** — `draft_dispute_message` tool can be repurposed / a new `draft_collection_message` tool built (S effort)
5. **Navigate to JenView/TiffaniView** — for approval + payment recording actions ✓

What v1 should NOT promise: sending emails, triggering payments, writing off debt, live bank feeds.

---

## 7. Honesty Stance Guidance

### Three-stance rule — NOT FOUND IN DECISION-LOG

Searched all 2400+ lines of DECISION-LOG.md, all reports in conductor-mark, and the entire milo-for-ppc src tree. **No "three-stance," "honesty rule," "Stance A/B/C" decision exists.** The closest patterns found:

- milo-outreach has a `human-in-the-loop` gate (`tenants.send_mode` = manual|auto) for email sends
- Disputes use a two-phase pattern: `is_draft=true` → separate action to actually send
- Invoice status flow naturally separates "Milo creates" from "human reviews/sends/pays"

**If the drawer spec needs a three-stance model, it must be created as a new decision.**

### Proposed stance mapping for Aging drawer actions

Based on actual primitive capabilities:

| Action | Proposed stance | Rationale |
|---|---|---|
| **View aging report** | **B (Milo executes)** | `/api/invoices/aging` returns data, drawer renders it. Pure read. |
| **View overdue list** | **B (Milo executes)** | Same — status query + render. |
| **Generate missing invoice** | **B (Milo executes)** | `generate_invoice` tool creates draft + PDF. Lands in Jen's queue. |
| **Draft collection message** | **A (Milo drafts, operator sends)** | No email send capability. Milo drafts, operator copy-pastes to email/Teams. |
| **Send invoice to buyer** | **C (Milo opens, operator drives)** | No email send integration. Drawer shows invoice PDF + "Download PDF" button. Operator emails manually. |
| **Record payment received** | **C (Milo opens, operator drives)** | Navigate to TiffaniView's Record Payment form. No tool wraps this. |
| **Mark as paid** | **C (Milo opens, operator drives)** | Same — PATCH endpoint exists but only TiffaniView calls it. Drawer could link to it or wrap it. |
| **Pay publisher** | **C (Milo opens, operator drives)** | No payment processor. Drawer shows "Pay $X to [publisher]" with Mercury/bank as the destination. Operator logs in elsewhere. |
| **Write off receivable** | **NOT AVAILABLE** | No write-off logic exists. Omit from v1 or show as grayed-out "Coming soon." |
| **Flag for review** | **B (Milo executes)** | PATCH `/api/invoices/[id]/flag` exists. Could be wired as drawer action. |
| **Send payment reminder** | **A (Milo drafts, operator sends)** | No send capability. Draft a nudge message, operator delivers. |
| **Cleared-to-pay check** | **B (Milo executes)** | Can query whether corresponding buyer AR is paid. Display-only — the gate is informational, not enforced in code. |

### Actions that would UPGRADE to Stance B with future integrations

| Action | Currently | With integration | What's needed |
|---|---|---|---|
| Send invoice | C (navigate) | B (execute) | Resend email templates + send route |
| Send reminder | A (draft) | B (execute) | Resend nudge templates + escalation schedule |
| Record payment | C (navigate) | B (execute) | Wrap PATCH endpoint as Milo tool |
| Pay publisher | C (navigate) | B (execute) | Mercury API integration |

---

## Known Bugs / Drift Risks for Spec

1. **Aging endpoint timezone bug**: `/api/invoices/aging/route.ts` uses `new Date().toISOString()` for "today" — does NOT use Denver timezone per billing rule 8. Days-outstanding calculation could be off by 1 day near midnight.

2. **Operator pill duplication**: `/api/operator/pill/route.ts` has its own `agingBucket()` function (line 185) duplicating the aging route's logic. Changes to one won't propagate to the other.

3. **Cleared-to-pay not enforced**: The `cleared_to_pay` BOOLEAN exists on invoices but is set during bank reconciliation as metadata, not as an AP payment gate. The back-to-back rule is human-managed.

4. **auto_send column unused**: `billing_schedules.auto_send` exists but no code reads it. If the drawer exposes a "toggle auto-send" switch, it writes to a column nothing acts on.

5. **PPCRM billingbot fallback**: The operator executive route tries PPCRM first for aging data. If PPCRM is fully frozen/unreachable, the local fallback misses pay-if-paid enforcement and blocked publisher invoice counts.

6. **Dashboard-v2 billing blocks only in milo-ops**: `ApAgingBlock`, `InvoiceBlock`, `WeeklyBillingBlock`, `BillingPipelineBlock` exist only in milo-ops. milo-for-ppc's operator surface uses the executive route instead. The drawer would be net-new for milo-for-ppc.

---

Cross-references: Foundation schema (`20260311000001`), billing_schedules migration (`20260330000001`), bank_transactions migration (`20260330100001`), billing-rules.md, JEN_complete.md, operator-pills.ts, invoice-generate.ts, billing-cycles.ts, ppcrm-client.ts.
