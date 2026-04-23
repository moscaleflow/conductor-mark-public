# D114: Milo-for-PPC + MOP Surface Audit for V4 Design

> Coder-3 Research | Directive #114 | Read-only audit — no code changes

---

## Executive Summary

- **12 pills audited**
- **BUILD-EXISTS-WRAP: 5** — Alerts, Contracts, Aging, Sales, Q&A (UI exists in milo-for-ppc, V4 rewraps as drawer/sheet)
- **BUILD-PARTIAL-FILL: 4** — Calls, Publishers, Buyers, Drafts (some UI but incomplete — MOP has the full version)
- **BUILD-NEW: 1** — E-sign (zero surface in milo-for-ppc; MOP has it but frozen)
- **MERGE-SCOPE-DECISION: 2** — Accounting + Billing (both exist and overlap heavily; Mark must decide if they're one pill or two)
- **Surprise flag**: Disputes (not on Mark's pill list) has more built-out surface than 4 listed pills. See §Cross-cutting.

---

## Route Inventory

### milo-for-ppc — 27 page routes

| Route | Purpose | Relevant pill |
|---|---|---|
| `/dashboard-v2` | Main operator dashboard — 33 block components, role-filtered | ALL pills |
| `/operator` | Duplicate of dashboard-v2 for certain roles | ALL pills |
| `/pipeline` | Entity list — 94 entities across 8 stages | Sales |
| `/pipeline-board` | Drag-and-drop kanban board | Sales |
| `/invoices` | Invoice generation + workflow | Accounting, Billing |
| `/jen` | Jen's invoice review queue | Accounting |
| `/tiffani` | Tiffani's payment view | Accounting |
| `/campaigns` | Campaign management (36 campaigns) | — |
| `/pings` | Ping/post analysis | — |
| `/ping-post` | Another ping view | — |
| `/partners` | TrackDrive partner sync | Publishers, Buyers |
| `/reconciliation` | Our data vs TrackDrive reconciliation | Accounting |
| `/reports` | Buyer report cross-referencing (CSV upload, mapping) | Accounting |
| `/disputes` | Dispute management + recovery | (NOT LISTED) |
| `/contract-review/[id]` | Contract analysis display (flagged clauses, risk, negotiate) | Contracts |
| `/observatory` | Admin-only live team monitor | — |
| `/eod-reports` | End of day reports | — |
| `/feedback` | Milo chat feedback log | — |
| `/support` | Support ticket system | Q&A |
| `/proof` | v1 funnel proof page | — |
| `/setup` | Setup | — |
| `/onboard` | Onboarding flow | — |
| `/malvin` | Malvin's DID management view | — |

### MOP (frozen) — 28+ page routes

| Route | Purpose | Relevant pill |
|---|---|---|
| `/` | Widget dashboard (AR/AP, overdue, fraud, outreach, campaigns) | ALL |
| `/publishers` + `[id]` | Full publisher list + detail + quality | Publishers |
| `/buyers` + `[id]` | Full buyer list + detail | Buyers |
| `/end-buyers` | End-buyer list | Buyers |
| `/calls` | Full call log + ConvoQC + fraud scoring + zero-revenue analysis | Calls |
| `/invoices` | Full invoice management (line items, deductions, credits, PDF) | Accounting, Billing |
| `/contract-analyzer` + history + rider-templates | Contract analysis with upload + rules | Contracts |
| `/signing` | Signing document list (pending/viewed/signed/voided) | E-sign |
| `/sign/[token]` | Signing page for counterparty | E-sign |
| `/review/[token]` | Review page for counterparty negotiation | E-sign, Contracts |
| `/negotiations` + `[id]` | Negotiation management | Contracts |
| `/outreach` | Full outreach queue (message queue + priorities + triggers) | Drafts |
| `/master-blacklist` | Blacklist management | — |
| `/convoqc-coverage` | ConvoQC coverage stats | Calls |
| `/dids` | DID management | — |
| `/campaigns` + `[id]` | Campaign management | — |

---

## Per-Pill Detail

### 1. Alerts

**a. milo-for-ppc: EXISTS**
- [AlertsBlock.tsx](src/components/dashboard-v2/AlertsBlock.tsx) — dashboard block, `span: 2`, status: `required` for admin + operations roles
- [BriefingPanel.tsx](src/components/BriefingPanel.tsx) — morning briefing with "Needs Your Attention" + "Watch List" + "Looking Ahead"
- [AdminExecutiveCards.tsx](src/components/AdminExecutiveCards.tsx) — Sprint 3 §0 top-5 ranked executive cards
- [Top5Frame.tsx](src/components/Top5Frame.tsx) — shared frame enforcing <=5 cards per role
- API: `GET /api/alerts`, `POST /api/alerts`, `POST /api/alerts/[id]/resolve`, `GET /api/briefing`
- Routes: `/dashboard-v2` (main), `/operator`

**b. MOP: EXISTS**
- `~/Mop3.18/MOP/app/page.tsx` — `FraudAlertsWidget`, `CampaignAlertsWidget` on dashboard
- Action items with dismiss/snooze/resolve from `actionItemsClient`

**c. Data wiring:**
- Source: Supabase `alerts` table + `ai_action_log` + `impact-engine`
- TrackDrive + ConvoQC + publisher grades + buyer health feed alert generation
- LIVE in milo-for-ppc — alert generation runs every 15-min sync cycle

**d. Presentation:** Dashboard block pattern. `AlertsBlock` renders as a rectangular tile in the block grid. Sprint 3 §0 designed `Top5Frame` + `AdminExecutiveCards` — this is architecturally closer to V4's drawer model (ranked, capped list with "everything else is clean" footer). The executive card pattern is the **closest existing code to V4 pill-drawer**.

**e. Gaps:** The Sprint 3 executive ranker (`src/lib/executive-ranker.ts`) is the natural V4 pill data source. Gap: alerts currently rendered as block tiles, not as a scrollable drawer/sheet list. V4 needs: pill tap → drawer opens with top-N ranked alerts, each expandable to action.

---

### 2. Contracts

**a. milo-for-ppc: EXISTS**
- [contract-review/[id]/page.tsx](src/app/contract-review/%5Bid%5D/page.tsx) — full contract analysis display (flagged clauses with risk levels, perspectives, recommendations, counter-text, MSA improvements)
- [ContractAnalysisDisplay.tsx](src/components/dashboard-v2/ContractAnalysisDisplay.tsx) — reusable display component
- API: `/api/contract/analyze-bg`, `/api/contract/negotiate`, `/api/contract/analyze-proxy`, `/api/contract/documents`, `/api/contract/process`
- Sprint 2 tools: `analyze_contract`, `generate_redlined_version`, `publish_contract_for_buyer_review`, `ingest_returned_contract`

**b. MOP: EXISTS**
- `~/Mop3.18/MOP/app/contract-analyzer/page.tsx` — upload + analysis (drag-drop, paste, or URL)
- `~/Mop3.18/MOP/app/contract-analyzer/history/page.tsx` — analysis history list
- `~/Mop3.18/MOP/app/contract-analyzer/rider-templates/page.tsx` — rider template management
- `~/Mop3.18/MOP/app/negotiations/page.tsx` + `[id]` — negotiation tracking

**c. Data wiring:**
- Source: @milo/contract-analysis + @milo/contract-signing + @milo/contract-negotiation (shared Supabase `tappyckcteqgryjniwjg`)
- `contract_documents` table for storage
- LIVE in milo-for-ppc — analysis, negotiate, and document routes work

**d. Presentation:** Full page (`/contract-review/[id]`). Dashboard-style — full-screen dedicated page, not drawer/modal. Clause-by-clause risk display with accept/reject/counter decisions per clause.

**e. Gaps:** No *list* view in milo-for-ppc — you navigate to `/contract-review/[id]` directly. MOP has the history list. V4 needs: pill tap → drawer with list of in-flight contracts (grouped by status: draft, analyzing, redlined, sent, signed), each drills to the review detail.

---

### 3. Drafts

**a. milo-for-ppc: PARTIAL**
- [FollowupsBlock.tsx](src/components/dashboard-v2/FollowupsBlock.tsx) — follow-ups due dashboard block
- [OutreachStatsBlock.tsx](src/components/dashboard-v2/OutreachStatsBlock.tsx) — outreach statistics
- [LinkedInActivityBlock.tsx](src/components/dashboard-v2/LinkedInActivityBlock.tsx) — LinkedIn activity tracking
- `src/lib/outreach-engine.ts` — generates follow-up tasks + prospect lists
- `src/lib/message-templates.ts` — 6 pre-built templates (publisher check-in, buyer bid increase, quality warning, report request, pause warning, capacity outreach)
- Sprint 2 tools: `draft_first_outreach`, `handle_objection`
- Role: `outreach` gets blocks `pipeline_summary`, `followups_due`, `outreach_stats`, `linkedin_activity`
- No dedicated page for viewing/editing draft messages

**b. MOP: EXISTS**
- `~/Mop3.18/MOP/app/outreach/page.tsx` — full outreach queue with items: entity name/type, trigger type, priority, status (pending/approved/claimed/completed/skipped/expired), message text, context, dedup key, claimed-by, timestamps

**c. Data wiring:**
- Source: `prospect_pipeline` + `drip_campaigns` + `ai_action_log` in milo-for-ppc
- MOP: `outreach_queue` table (frozen)
- milo-outreach (separate app) has its own leads + drafts on shared Supabase
- @milo/crm leads table could feed — see D106 scoping report
- NOT fully live — AI-generated drafts exist but no unified "drafts inbox"

**d. Presentation:** Dashboard blocks only. No dedicated draft management page. MOP has a full outreach queue page.

**e. Gaps:** Biggest build gap among "core" pills. V4 needs: pill tap → drawer showing all in-flight messages (openers, follow-up 1, follow-up 2) with status (draft/approved/sent/replied), ability to edit/approve/send. This is essentially the MOP outreach queue reimagined as a drawer. Requires wiring to either milo-outreach or building a unified draft queue in milo-for-ppc.

---

### 4. E-sign

**a. milo-for-ppc: NONE**
- Zero signing management UI. No page, no component, no dashboard block for listing signing documents.
- Contract review exists (`/contract-review/[id]`) but that's analysis, not signing.
- `AgreedTermsForm.tsx` in `dashboard-v2/` exists but is for recording agreed terms, not e-sign management.

**b. MOP: EXISTS**
- `~/Mop3.18/MOP/app/signing/page.tsx` — full list of signing documents with status badges (pending, viewed, signed, voided, draft), short codes, signing tokens, counterparty info
- `~/Mop3.18/MOP/app/sign/[token]/page.tsx` — signing page (counterparty-facing)
- `~/Mop3.18/MOP/app/review/[token]/page.tsx` — review page (counterparty-facing)
- Document types: MSA, IO, W-9

**c. Data wiring:**
- Source: @milo/contract-signing primitive, `signing_documents` table on shared Supabase
- Data migrated (D54: 109 docs, 81 active tokens with `expires_at: null`)
- Redirect from MOP → tlp.justmilo.app deployed (D103)
- The counterparty-facing pages (`/sign/[token]`, `/review/[token]`) work on tlp.justmilo.app via redirect
- BUT: no operator-facing management UI in milo-for-ppc

**d. Presentation:** N/A in milo-for-ppc. MOP has a full page with status badges, filter, and action links.

**e. Gaps:** Full build needed. V4 needs: pill tap → drawer showing signing documents grouped by status (pending, viewed, signed), each with counterparty info + link to signing URL. The data layer exists (primitives + shared Supabase) — only the operator-facing list UI is missing.

---

### 5. Aging

**a. milo-for-ppc: EXISTS**
- [ApAgingBlock.tsx](src/components/dashboard-v2/ApAgingBlock.tsx) — dashboard block for AP aging
- API: `GET /api/invoices/aging` — returns aging receivables data
- Part of `billing` role blocks: `ap_aging` is in the billing role block list
- `GET /api/invoices/[id]/mark-overdue` — marks invoices overdue

**b. MOP: PARTIAL**
- Invoice page shows status + due_date but no dedicated aging view or aging buckets (30/60/90)
- `OverdueSummaryWidget` on MOP dashboard at `~/Mop3.18/MOP/app/page.tsx`

**c. Data wiring:**
- Source: `invoices` table in milo-for-ppc Supabase (`tmzsmdkfqvjjjwkukysg`)
- LIVE — aging endpoint computes buckets from invoice due dates

**d. Presentation:** Dashboard block. `ApAgingBlock` renders in the billing role's block grid.

**e. Gaps:** The block exists but renders as a dashboard tile. V4 needs: pill tap → drawer with aging buckets (current, 30-day, 60-day, 90+ day) with dollar sums and drill-to-invoice. The data and API exist; only the drawer presentation is missing.

---

### 6. Calls

**a. milo-for-ppc: PARTIAL**
- [FlaggedCallsBlock.tsx](src/components/dashboard-v2/FlaggedCallsBlock.tsx) — flagged calls dashboard block (call-monitoring role)
- [PublisherMonitoringBlock.tsx](src/components/dashboard-v2/PublisherMonitoringBlock.tsx) — publisher call monitoring
- [BuyerMonitoringBlock.tsx](src/components/dashboard-v2/BuyerMonitoringBlock.tsx) — buyer call monitoring
- [CallDetailModal.tsx](src/components/CallDetailModal.tsx) — modal for call details
- [QualityBlock.tsx](src/components/dashboard-v2/QualityBlock.tsx) — quality stats
- Sprint 2 tools: `review_call_quality`, `detect_coached_call`, `diagnose_connection_issue`
- No dedicated `/calls` page

**b. MOP: EXISTS**
- `~/Mop3.18/MOP/app/calls/page.tsx` — full call log with:
  - ConvoQC disposition/summary/transcription/flags
  - Fraud scoring (`computeFraudScore`, `getCallerVerdict`, FraudTier)
  - Zero-revenue analysis (`ZeroRevenueAnalysis` component)
  - Caller resolution menus (`CallerResolutionMenu`)
  - PageHeaderWithStats
  - Rich filtering by publisher, buyer, campaign, date range
- `~/Mop3.18/MOP/app/convoqc-coverage/page.tsx` — ConvoQC coverage stats per campaign

**c. Data wiring:**
- Source: TrackDrive API (15-min sync) → `call_records` table + ConvoQC enrichment
- LIVE in milo-for-ppc — call data syncs continuously
- ConvoQC integration live (37% coverage, campaign-level)

**d. Presentation:** Dashboard blocks + modal in milo-for-ppc. MOP has the full rich calls page. The directive says this pill is "problem calls for operator research (not firehose)" — the `FlaggedCallsBlock` is closer to this intent than MOP's full call log.

**e. Gaps:** V4 needs: pill tap → drawer showing problem calls (flagged, zero-revenue, coached, connection issues) with ConvoQC data inline, expandable to full detail. The `CallDetailModal` is close to the V4 pattern already. Building the filtered list + wiring to ConvoQC is the gap.

---

### 7. Sales

**a. milo-for-ppc: EXISTS**
- [pipeline/page.tsx](src/app/pipeline/page.tsx) — entity pipeline list (94 entities, 8 stages: outreach, qualifying, drip, onboarding, activation, active, dormant, blacklisted), filterable by entity type/stage/search. Contact details joined. Entity detail side panel.
- [pipeline-board/page.tsx](src/app/pipeline-board/page.tsx) — drag-and-drop kanban board (`@hello-pangea/dnd`). Entities as cards, stages as columns. Auto-compose signal for email/LinkedIn/Teams.
- [PipelineBlock.tsx](src/components/dashboard-v2/PipelineBlock.tsx) — summary dashboard block
- [KanbanBlock.tsx](src/components/dashboard-v2/KanbanBlock.tsx) — kanban in dashboard context
- [EntityDetailPanel.tsx](src/components/dashboard-v2/EntityDetailPanel.tsx) — sliding detail panel with contacts, stats, blacklist status, match suggestions
- Source tables: `prospect_pipeline`, `contacts`, `drip_campaigns`, `action_items`

**b. MOP: PARTIAL**
- No pipeline or kanban. Publisher/buyer lists serve as entity management but without stage tracking.

**c. Data wiring:**
- Source: Supabase `prospect_pipeline` + `contacts` tables
- LIVE in milo-for-ppc — pipeline CRUD works, kanban drag-drop works

**d. Presentation:** Full pages (pipeline list + kanban board) + dashboard blocks. This is a dashboard-page pattern, not drawer/modal. But `EntityDetailPanel` is already a sliding side panel — close to V4 drawer.

**e. Gaps:** V4 needs: pill tap → drawer/sheet showing pipeline by stage. The kanban board could be the drawer content. `EntityDetailPanel` is already a sliding panel. Main gap: the full page needs to be refactored into a drawer-launched surface instead of a standalone route.

---

### 8. Publishers

**a. milo-for-ppc: PARTIAL**
- [partners/page.tsx](src/app/partners/page.tsx) — TrackDrive partner sync page showing publishers AND buyers in a combined diff view. Shows sync status (would_add, would_update, already_synced), naming groups, but NOT a traditional publisher management page.
- [TopPublishersBlock.tsx](src/components/dashboard-v2/TopPublishersBlock.tsx) — top publisher stats dashboard block
- [PublisherMonitoringBlock.tsx](src/components/dashboard-v2/PublisherMonitoringBlock.tsx) — publisher monitoring block
- Publisher grades: `src/lib/publisher-grades.ts` + `GET /api/publishers/grades` (A-F grading)
- `EntityDetailPanel` — sliding detail panel for publisher entities

**b. MOP: EXISTS**
- `~/Mop3.18/MOP/app/publishers/page.tsx` — full publisher list with quality data (answer rate, duration, short calls, conversions, repeat callers), filter bar, side panel, status badges, warning badges, form modals (add/edit), blacklist/unblacklist modals, duplicate merge, company merge, synonym management
- `~/Mop3.18/MOP/app/publishers/[id]/page.tsx` — publisher detail page

**c. Data wiring:**
- milo-for-ppc: `prospect_pipeline` for entities + `call_records` for stats + publisher grades
- MOP: `publishers` table (frozen)
- @milo/crm `crm_counterparties WHERE counterparty_type = 'publisher'` — the canonical table

**d. Presentation:** Partner sync page in milo-for-ppc — this is a dev/ops tool, not an operator publisher view. MOP has the operator-facing publisher management.

**e. Gaps:** V4 needs: pill tap → drawer with publisher list, each showing: name, status, grade (A-F), 7d calls/revenue, primary concern. Detail drill to `EntityDetailPanel`. The MOP publisher page is the reference; milo-for-ppc has no equivalent operator-facing publisher list.

---

### 9. Buyers

**a. milo-for-ppc: PARTIAL**
- Same `/partners` page as Publishers (combined view)
- [BuyerMonitoringBlock.tsx](src/components/dashboard-v2/BuyerMonitoringBlock.tsx) — buyer monitoring
- Buyer health scoring: `src/lib/buyer-health.ts` + `GET /api/buyers/health` (0-100 score)
- `EntityDetailPanel` — same sliding panel

**b. MOP: EXISTS**
- `~/Mop3.18/MOP/app/buyers/page.tsx` — full buyer list + detail
- `~/Mop3.18/MOP/app/buyers/[id]/page.tsx` — buyer detail page
- `~/Mop3.18/MOP/app/end-buyers/page.tsx` — end-buyer list (downstream buyers)

**c. Data wiring:**
- Same pattern as Publishers but `counterparty_type = 'buyer'`
- Buyer health scores LIVE in milo-for-ppc

**d. Presentation:** Same gap as Publishers — no dedicated operator-facing buyer list in milo-for-ppc.

**e. Gaps:** V4 needs: pill tap → drawer with buyer list showing: name, health score (0-100 / grade), 7d revenue, overdue invoices, conversion rate. MOP buyer page is the reference.

---

### 10. Accounting

**a. milo-for-ppc: EXISTS**
- [invoices/page.tsx](src/app/invoices/page.tsx) — invoice generation + workflow (draft, finalize, approve, flag, payment tracking). Full page with TD comparison, type/status/TD-match filters.
- [JenView.tsx](src/components/JenView.tsx) at `/jen` — Jen's invoice review queue (TD comparison inline, $1 threshold, approve/flag)
- [TiffaniView.tsx](src/components/TiffaniView.tsx) at `/tiffani` — Tiffani's payment view (mark paid, record payment)
- [reconciliation/page.tsx](src/app/reconciliation/page.tsx) — our data vs TrackDrive reconciliation (buyer-level, CID-level mismatches)
- [reports/page.tsx](src/app/reports/page.tsx) — buyer report cross-referencing (CSV upload, column mapping, match/unmatch, revenue diff)
- Dashboard blocks: `InvoiceBlock`, `WeeklyBillingBlock`, `ApAgingBlock`, `BillingPipelineBlock`, `BankTransactionsBlock`, `CommissionTrackerBlock`, `MediaRiteXRefBlock`

**b. MOP: EXISTS**
- `~/Mop3.18/MOP/app/invoices/page.tsx` — full invoice management with line items, deductions, credits, PDF generation, status lifecycle, payment tracking

**c. Data wiring:**
- Source: `invoices`, `buyer_reports`, `buyer_report_rows`, `disputes`, `reconciliation_runs` tables
- All on milo-for-ppc Supabase (`tmzsmdkfqvjjjwkukysg`)
- LIVE — full invoice lifecycle works, TD comparison, aging, reconciliation all functional

**d. Presentation:** Multiple full pages (invoices, jen, tiffani, reconciliation, reports) + 7 dashboard blocks. This is the most built-out surface in the entire app. All pages are dashboard/full-page style.

**e. Gaps:** V4 needs to collapse 5 pages + 7 blocks into a coherent drawer surface. Suggested sub-views: AP summary, AR summary, overdue, reconciliation status. The data is 100% there; the challenge is information design for a drawer that surfaces the right slice without the firehose.

---

### 11. Billing

**a. milo-for-ppc: EXISTS (heavy overlap with Accounting)**
- [WeeklyBillingBlock.tsx](src/components/dashboard-v2/WeeklyBillingBlock.tsx) — weekly billing summary
- [BillingPipelineBlock.tsx](src/components/dashboard-v2/BillingPipelineBlock.tsx) — billing pipeline (draft → pending_jen → approved → sent → paid)
- [InvoiceBlock.tsx](src/components/dashboard-v2/InvoiceBlock.tsx) — invoice overview
- API: `/api/invoices/[id]/mark-overdue`, `/api/invoices/[id]/record-payment`, `/api/invoices/[id]/pdf`
- Invoice status lifecycle documented in SYSTEMS.md: draft → pending_jen → jen_approved/jen_flagged → sent → paid

**b. MOP: EXISTS** — same as Accounting

**c. Data wiring:** Same as Accounting — `invoices` table

**d. Presentation:** Dashboard blocks. The billing role in `ROLE_BLOCKS` gets: `billing_pipeline`, `overdue_invoices`, `weekly_billing`, `ap_aging`, `td_balance`, `commission_tracker`, `bank_transactions`.

**e. Gaps:** **The primary gap is scope definition, not UI.** Billing and Accounting share the same data layer. If they're separate pills:
- **Billing** = invoices due, payment status, who owes what, PDF generation
- **Accounting** = AP/AR reconciliation, buyer report cross-referencing, bank transactions, commission tracking, dispute recovery
If they're one pill, the drawer needs tabs or sub-sections. Mark must decide.

---

### 12. Q&A

**a. milo-for-ppc: EXISTS**
- [support/page.tsx](src/app/support/page.tsx) — full support ticket system with: user info, description, screenshot upload, Milo auto-response, admin response, status (open/in-progress/resolved), priority (low/normal/high/urgent), assigned-to, resolved-at
- [SupportTicketsBlock.tsx](src/components/dashboard-v2/SupportTicketsBlock.tsx) — dashboard block summary

**b. MOP: NONE** — no support ticket system

**c. Data wiring:**
- Source: Supabase (support tickets table on `tmzsmdkfqvjjjwkukysg`)
- LIVE in milo-for-ppc

**d. Presentation:** Full page + dashboard block. Dashboard-style full page.

**e. Gaps:** V4 needs: pill tap → drawer showing open tickets sorted by priority, each expandable to detail + admin response. The data and full page exist — this is a rewrap.

---

## Cross-Cutting Findings

### The "dashboard" pattern Mark wants to leave behind

Located in milo-for-ppc:

| File | What it does |
|---|---|
| `src/app/dashboard-v2/page.tsx` | Main dashboard — imports 33 block components, renders per-role grid |
| `src/app/operator/page.tsx` | Alternate entry point, same dashboard content |
| `src/lib/dashboard-v2-types.ts` | `ROLE_BLOCKS` mapping: 8 roles × up to 12 blocks each |
| `src/components/dashboard-v2/` | 32 block component files (AlertsBlock, PulseBlock, TopPublishersBlock, etc.) |

The dashboard-v2 renders a grid of rectangular blocks per role. Sprint 3 §0 (documented in `docs/SPRINT3_TARGETS.md` §0.1–0.6) designed a **Top5Frame + "Everything else is clean" + pills footer** to replace the firehose. This is architecturally the V3→V4 bridge — V4 pill-fan continues the same evolution.

The specific files that embody the "dashboard tiles" pattern Mark rejected:
- `src/lib/dashboard-v2-types.ts:66-75` — `ROLE_BLOCKS` hardcoded block lists
- `src/app/dashboard-v2/page.tsx:1-46` — 33 block component imports
- Every `*Block.tsx` file in `src/components/dashboard-v2/`

### MOP dashboard

`~/Mop3.18/MOP/app/page.tsx` imports 13 widget components:
- `QuickActionsWidget`, `ARSummaryWidget`, `APSummaryWidget`, `OverdueSummaryWidget`, `DraftInvoicesWidget`, `FraudAlertsWidget`, `EndBuyersWidget`, `CampaignAppsWidget`, `OutreachQueueWidget`, `RecentActivityWidget`, `CampaignAlertsWidget`, `DailyBriefingWidget`, `CallVolumeWidget`, `RevenueChartWidget`
- Plus `DashboardCustomizeModal` for layout customization and `PresetPickerModal` for preset layouts

This was "the dashboard that had the right info" — every widget surfaces genuinely useful operator data. The problem was presentation (tile grid), not content.

### Markdown documentation inventory

| Pill domain | Key docs |
|---|---|
| Alerts | `SYSTEMS.md` §Alert System, §Impact Estimation Engine; `SPRINT3_TARGETS.md` §0 executive view; `docs/ROADMAP.md` Stage 3 (Decision-Grade Alerts) |
| Contracts | `vendor/contract-analysis/SCHEMA.md`, `vendor/contract-signing/SCHEMA.md`, `vendor/contract-negotiation/SCHEMA.md`; MOP `docs/riders/publisher-master-rider-v1.md`, `docs/riders/buyer-master-rider-v1.md` |
| Drafts | `docs/team-skills/VEE_complete.md`, `docs/team-skills/CAM_complete.md` (outreach operators); MOP `docs/outreach-design.md` |
| E-sign | `vendor/contract-signing/SCHEMA.md`; MOP `docs/signing-page-refactor-plan.md` |
| Aging | `docs/billing-rules.md`; `SYSTEMS.md` §Invoice Generation + Workflow |
| Calls | `docs/integration-map.md` §1 TrackDrive, §2 ConvoQC; `docs/trackdrive-mcp/CONVOQC_INTEGRATION.md`; `SYSTEMS.md` §Call Sync, §ConvoQC |
| Sales | `SYSTEMS.md` §Pipeline Management; `docs/team-skills/CAM_complete.md` (prospecting); `docs/discovery/publisher-onboarding-current-state.md` |
| Publishers | `SYSTEMS.md` §Publisher Grades, §Publisher Quality Reporter; `docs/team-skills/FAB_complete.md` (publisher QA) |
| Buyers | `SYSTEMS.md` §Buyer Health Scoring, §Buyer Report Cross-Referencing |
| Accounting/Billing | `SYSTEMS.md` §Invoice Generation + Workflow, §Reconciliation; `docs/billing-rules.md`; `SPRINT3_TARGETS.md` §13 Buyer Report Tracking |
| Q&A | No dedicated docs — support system was built without specification doc |

### Pills Mark DIDN'T list that exist as surfaces today

| Surface | Current UI | Data backing | Pill candidate? |
|---|---|---|---|
| **Disputes** | `/disputes` page + `DisputeRecoveryBlock` + full API (detect, raise, batch-raise, respond, stats) + `dispute-drafter.ts` | `disputes` table, buyer report rows, call records | **YES — strong candidate.** More built-out than Drafts, Calls, E-sign. Consider adding as core pill. |
| **Campaigns** | `/campaigns` page + `CampaignHealthBlock` + full CRUD + auto-detect billing type | `campaigns`, `campaign_routes`, `campaign_qualifiers` | Maybe. Operator reference, not daily attention surface. |
| **DID Management** | `/malvin` page + `DIDTrackerBlock` | TrackDrive DIDs | No — Malvin-specific ops tool. |
| **Quality/QC** | `QualityBlock` + `FlaggedCallsBlock` + publisher grades | `call_records` + ConvoQC | Subsumed by Calls pill. |
| **Reconciliation** | `/reconciliation` page | `reconciliation_runs` | Subsumed by Accounting pill. |

### Pills Mark should consider DROPPING

| Pill | Reason |
|---|---|
| **Billing** (as separate from Accounting) | 100% data overlap with Accounting. Every billing surface is also an accounting surface. Merge into one pill with sub-tabs, or redefine Billing as "what's owed to us" (AR) vs Accounting as "what we owe" (AP). |

### Surprise flag

**Disputes are the most glaring omission from the pill list.** The dispute surface in milo-for-ppc has:
- A full dedicated page (`/disputes`) with status lifecycle (detected → raised → pending_response → escalated → accepted/rejected/expired/withdrawn)
- `DisputeRecoveryBlock` dashboard block
- Full API: detect, raise, batch-raise, respond, stats endpoints
- `dispute-drafter.ts` for AI-generated dispute messages
- Sprint 3 §8 designed pre-loaded context for dispute cards
- Revenue recovery tracking (`revenue_at_risk`, `revenue_recovered`)

This has more infrastructure than E-sign (which needs a full build) and Drafts (which is partial). If the V4 pill list doesn't include Disputes, that surface loses its home in the new UI.

---

## Recommended V4 Pill Scope

### Ship with V4 shell (v4.0) — 8 pills

| Pill | Rationale |
|---|---|
| **Alerts** | Closest to V4 pattern already (Top5Frame, executive cards). Rewrap only. |
| **Contracts** | UI exists, data wired. Needs list view added to drawer. |
| **Sales** | Full pipeline + kanban exists. EntityDetailPanel is already a sliding panel. |
| **Aging** | Block + API exist. Simple drawer build. |
| **Accounting** | Most built-out surface. Collapse 5 pages into tabbed drawer. |
| **Q&A** | Full page exists. Simple rewrap. |
| **Calls** | Blocks + modal exist. Build filtered "problem calls" list for drawer. |
| **Disputes** (ADD) | Full page + API + AI drafter exist. Essential revenue recovery surface. |

### Defer to v4.1 — 4 pills

| Pill | Rationale |
|---|---|
| **E-sign** | Zero UI in milo-for-ppc. Full build needed. Data layer ready but UI is net-new. |
| **Drafts** | Partial blocks only. Needs unified drafts inbox — significant build or milo-outreach wiring. |
| **Publishers** | No operator-facing list page in milo-for-ppc. Needs port from MOP pattern. Can use EntityDetailPanel for interim. |
| **Buyers** | Same as Publishers. |

### Merge decision needed from Mark

- **Billing**: merge into Accounting (as sub-tab) or redefine scope boundary

Report at: https://raw.githubusercontent.com/moscaleflow/conductor-mark-public/main/reports/2026-04-21-d114-v4-design-surface-audit.md
