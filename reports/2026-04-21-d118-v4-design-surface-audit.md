# D118: Milo-for-PPC + MOP Surface Audit for V4 Design

> Coder-3 Research | Directive #118 | Read-only audit — no code changes
> Prerequisite: D85 (V4 operator UI pivot) locked in D117

---

## Executive Summary

- **12 pills audited**
- **BUILD-EXISTS-WRAP: 6** — Alerts, Contracts, Aging, Sales, Q&A, Calls
- **BUILD-PARTIAL-FILL: 3** — Publishers, Buyers, Drafts
- **BUILD-NEW: 1** — E-sign (zero operator-facing surface in milo-for-ppc)
- **MERGE-SCOPE-DECISION: 2** — Accounting + Billing (data overlap; one pill or two?)

**CRITICAL FINDING: The V4 pill-fan + drawer architecture already exists.**

The `/operator` page (`src/app/operator/page.tsx`) ships:
- `MorningBriefing` — greeting + needs-you cards with "Everything else is clean" footer
- `PillBar` — horizontal strip, role-defaults + custom pills, badge counts, `+ Add pill`
- `PillDrawer` — right-side 460px slide-out with severity dots, $ at risk, Review/Snooze
- `SearchBar` — single input with rotating examples, dispatches to Milo chat
- `Top5Frame` + `AdminExecutiveCards` — admin executive 5-card view
- `OperatorTour` — 4-step guided onboarding for first-time operators

9 drawer-pill IDs have server predicates: `qa_queue`, `qc_queue`, `ping_summary`, `ping_health`, `quality_overview`, `mediarite_xref`, `needs_attention`, `dispute_recovery`, `disputes`. Non-drawer pills dispatch to Milo chat.

**D85's V4 vision is 70% implemented.** The shell exists. What's missing: file-drop auto-routing, mic input, footer sync status, long-press jiggle customization, curated `+` picker (current `+` dispatches to chat), and drawer predicates for the 12 content pills Mark specified.

---

## Existing V4 Architecture — Component Map

| Component | File | What it does |
|---|---|---|
| Operator page | `src/app/operator/page.tsx` | Main V4 shell — briefing → pills → search → "Full dashboard" link |
| PillBar | `src/components/operator/PillBar.tsx` | Horizontal pill strip, role-default + custom, badges, `+ Add pill` |
| PillDrawer | `src/components/operator/PillDrawer.tsx` | Right-side slide-out (460px), severity dots, $ at risk, Review/Snooze 24h |
| MorningBriefing | `src/components/operator/MorningBriefing.tsx` | "Good morning, Fab" + needs-you cards + "Everything else is clean" |
| SearchBar | `src/components/operator/SearchBar.tsx` | Single input, rotating example prompts, Enter → Milo chat |
| Top5Frame | `src/components/Top5Frame.tsx` | Admin executive: capped 5-card view with pill badges footer |
| AdminExecutiveCards | `src/components/AdminExecutiveCards.tsx` | Ranked cards inside Top5Frame |
| OperatorTour | `src/components/OperatorTour.tsx` | 4-step cinematic onboarding |
| operator-pills | `src/lib/operator-pills.ts` | 8 role-based pill sets, focus-area reranking, custom pill merge |
| executive-ranker | `src/lib/executive-ranker.ts` | Score = max(dollar_at_risk, decision_urgency_score), caps at top N |
| /api/operator | `src/app/api/operator/route.ts` | Backend: profile + briefing + pills |
| /api/operator/pill | `src/app/api/operator/pill/route.ts` | Drawer data: pill predicates → filtered alert list ranked by $ at risk |

### Existing Pill Sets (from `operator-pills.ts`)

| Role | Pills | Count |
|---|---|---|
| admin | disputes_signoff, contracts_review, team_blocked, revenue_anomalies, mediarite_xref, today_pulse, needs_attention, dispute_recovery, top_publishers, billing_snapshot, team_time | 11 |
| operations | today_pulse, needs_attention, top_publishers | 3 |
| publisher-qa | qa_queue, mediarite_xref, ping_summary | 3 |
| billing | overdue_invoices, weekly_billing, ap_aging, td_balance, disputes | 5 |
| outreach | followups_due, pipeline_summary, new_leads, outreach_stats | 4 |
| prospecting | kanban, new_prospects, blacklist_hits | 3 |
| call-monitoring | did_tracker, publisher_push, campaign_health | 3 |
| qc | qc_queue, quality_overview | 2 |

### Drawer vs Chat Pills

| Drawer pills (server predicate returns items) | Chat pills (dispatch to Milo conversation) |
|---|---|
| qa_queue, qc_queue, ping_summary, ping_health, quality_overview, mediarite_xref, needs_attention, dispute_recovery, disputes, disputes_signoff, contracts_review, team_blocked, revenue_anomalies | today_pulse, top_publishers, billing_snapshot, team_time, overdue_invoices, weekly_billing, ap_aging, td_balance, followups_due, pipeline_summary, new_leads, outreach_stats, kanban, new_prospects, blacklist_hits, did_tracker, publisher_push, campaign_health |

---

## Route Inventory

### milo-for-ppc — 27 page routes

| Route | Purpose | Relevant pill |
|---|---|---|
| `/operator` | **V4 shell** — briefing + pill bar + drawer + search | ALL pills |
| `/dashboard-v2` | Legacy dashboard — 33 block components, role-filtered grid | ALL (old pattern) |
| `/pipeline` | Entity list — 94 entities, 8 stages | Sales |
| `/pipeline-board` | Drag-and-drop kanban | Sales |
| `/invoices` | Invoice generation + workflow | Accounting, Billing |
| `/jen` | Jen's invoice review queue | Accounting |
| `/tiffani` | Tiffani's payment view | Accounting |
| `/campaigns` | Campaign management | — |
| `/pings` | Ping/post analysis | — |
| `/ping-post` | Another ping view | — |
| `/partners` | TrackDrive partner sync (diff view) | Publishers, Buyers |
| `/reconciliation` | Our data vs TrackDrive reconciliation | Accounting |
| `/reports` | Buyer report cross-referencing (CSV) | Accounting |
| `/disputes` | Dispute management + recovery | (NOT LISTED — see surprise) |
| `/contract-review/[id]` | Contract analysis (clauses, risk, negotiate) | Contracts |
| `/observatory` | Admin-only live team monitor | — |
| `/eod-reports` | End of day reports | — |
| `/feedback` | Milo chat feedback log | — |
| `/support` | Support ticket system | Q&A |
| `/proof` | v1 funnel proof page | — |
| `/setup` | Setup | — |
| `/onboard` | Onboarding flow | — |
| `/malvin` | Malvin's DID management | — |

### MOP (frozen) — 33 page routes

| Route | Purpose | Relevant pill |
|---|---|---|
| `/` | Widget dashboard (AR/AP, overdue, fraud, outreach, campaigns) | ALL |
| `/publishers` + `[id]` | Full publisher list + detail + quality | Publishers |
| `/buyers` + `[id]` | Full buyer list + detail | Buyers |
| `/end-buyers` | End-buyer list | Buyers |
| `/calls` | Full call log + ConvoQC + fraud + zero-revenue | Calls |
| `/invoices` | Invoice management (line items, deductions, PDF) | Accounting, Billing |
| `/contract-analyzer` + history + rider-templates | Contract upload + rules + riders | Contracts |
| `/signing` | Signing document list (status badges, tokens) | E-sign |
| `/sign/[token]` | Signing page (counterparty-facing) | E-sign |
| `/review/[token]` | Review page (counterparty negotiation) | E-sign, Contracts |
| `/negotiations` + `[id]` | Negotiation management | Contracts |
| `/outreach` | Full outreach queue (triggers, priorities, dedup) | Drafts |
| `/master-blacklist` | Blacklist management | — |
| `/convoqc-coverage` | ConvoQC coverage stats | Calls |
| `/dids` | DID management | — |
| `/campaigns` + `[id]` | Campaign management | — |
| `/routes` | Route management | — |
| `/reports` | Reports | Accounting |
| `/settings` + company | Settings | — |
| `/ping-post` | Ping/post | — |
| `/offers` | Offers list | E-sign |
| `/offer/[token]` | Offer view | E-sign |
| `/invoice/[token]` | Invoice view | Billing |
| `/onboard/[token]` | Publisher onboarding | Publishers |
| `/o/[code]`, `/s/[code]` | Short code redirects | — |

---

## Per-Pill Detail

### CORE PILLS (always visible)

---

### 1. Alerts

**a. milo-for-ppc: EXISTS**
- `/operator` page: `MorningBriefing` renders needs-you cards ranked by severity
- `Top5Frame` + `AdminExecutiveCards`: admin executive view, 5-card cap
- `PillDrawer` with `needs_attention` predicate (returns all active alerts)
- `AlertsBlock` at `src/components/dashboard-v2/AlertsBlock.tsx` (legacy dashboard)
- `BriefingPanel` at `src/components/BriefingPanel.tsx` — morning brief
- API: `GET /api/alerts`, `POST /api/alerts`, `POST /api/alerts/[id]/resolve`, `GET /api/briefing`, `GET /api/operator/pill?pill=needs_attention`
- Existing pills: `needs_attention` (operations), `dispute_recovery` (admin) — both have drawer predicates

**b. MOP: EXISTS**
- `~/Mop3.18/MOP/app/page.tsx` — `FraudAlertsWidget`, `CampaignAlertsWidget` on dashboard

**c. Data wiring:**
- Source: Supabase `alerts` table + `ai_action_log` + impact-engine + executive-ranker
- TrackDrive + ConvoQC + publisher grades + buyer health feed alert generation
- LIVE in milo-for-ppc — alert generation runs every 15-min sync cycle

**d. Presentation:** Already V4 pattern. `MorningBriefing` is the conversation-first home for alerts. `PillDrawer` opens on `needs_attention` tap with filtered, ranked list. This is NOT the dashboard tile pattern — it IS the D85 drawer pattern, already shipping.

**e. Gaps:** Minimal. The `needs_attention` pill is a catch-all. V4 needs Alerts as a dedicated always-visible pill with its own tuned predicate (not just "all active alerts"). Consider: pill shows count of unresolved alerts, drawer ranks by executive score (dollar_at_risk + urgency).

---

### 2. Contracts

**a. milo-for-ppc: EXISTS**
- `/contract-review/[id]` — full analysis display: flagged clauses with risk levels (critical/high/medium/low/favorable), perspectives, recommendations, counter-text, MSA improvements
- `ContractAnalysisDisplay.tsx` at `src/components/dashboard-v2/ContractAnalysisDisplay.tsx`
- API: `/api/contract/analyze-bg`, `/api/contract/negotiate`, `/api/contract/analyze-proxy`, `/api/contract/documents`, `/api/contract/process`
- Sprint 2 tools: `analyze_contract`, `generate_redlined_version`, `publish_contract_for_buyer_review`, `ingest_returned_contract`
- Existing pill: `contracts_review` (admin) — has drawer predicate matching `/contract|msa|redline|addendum/i`

**b. MOP: EXISTS**
- `~/Mop3.18/MOP/app/contract-analyzer/page.tsx` — upload + analysis
- `~/Mop3.18/MOP/app/contract-analyzer/history/page.tsx` — analysis history
- `~/Mop3.18/MOP/app/contract-analyzer/rider-templates/page.tsx` — rider templates
- `~/Mop3.18/MOP/app/negotiations/page.tsx` + `[id]` — negotiation tracking

**c. Data wiring:**
- Source: @milo/contract-analysis + @milo/contract-signing + @milo/contract-negotiation (shared Supabase `tappyckcteqgryjniwjg`)
- `contract_documents` table for storage
- LIVE in milo-for-ppc — analysis, negotiate, and document routes work

**d. Presentation:** Full page (`/contract-review/[id]`). The `contracts_review` admin pill already has a drawer predicate — but it filters the `alerts` table for contract-related alerts, not the `contract_documents` table directly. The drawer would show "contracts that need attention" (alerts), not "all contracts" (documents). This is actually closer to D85's intent than a document list.

**e. Gaps:** No contract *list* in the drawer — only contract-related alerts. V4 needs: pill badge shows count of in-flight contracts, drawer shows contracts grouped by status (analyzing, redlined, awaiting signature, signed). Requires a new drawer data source (contract_documents query), not just the alerts predicate.

---

### 3. Drafts

**a. milo-for-ppc: PARTIAL**
- `FollowupsBlock.tsx` at `src/components/dashboard-v2/FollowupsBlock.tsx` — follow-ups due
- `OutreachStatsBlock.tsx` at `src/components/dashboard-v2/OutreachStatsBlock.tsx` — outreach stats
- `LinkedInActivityBlock.tsx` at `src/components/dashboard-v2/LinkedInActivityBlock.tsx` — LinkedIn activity
- `src/lib/outreach-engine.ts` — generates follow-up tasks + prospect lists
- `src/lib/message-templates.ts` — 6 pre-built templates
- Sprint 2 tools: `draft_first_outreach`, `handle_objection`
- Existing pills: `followups_due`, `new_leads`, `outreach_stats` (outreach role) — all are chat pills (no drawer predicate)
- No dedicated page for viewing/editing draft messages

**b. MOP: EXISTS**
- `~/Mop3.18/MOP/app/outreach/page.tsx` — full outreach queue: entity name/type, trigger_type, priority, status (pending/approved/claimed/completed/skipped/expired), message_text, context, dedup_key, claimed_by

**c. Data wiring:**
- Source: `prospect_pipeline` + `drip_campaigns` + `ai_action_log` in milo-for-ppc
- MOP: `outreach_queue` table (frozen)
- milo-outreach (separate app) has own leads + drafts on shared Supabase
- NOT fully live — AI-generated drafts exist but no unified "drafts inbox"

**d. Presentation:** Dashboard blocks only. All outreach pills are chat-dispatch, no drawer. MOP has the full outreach queue page.

**e. Gaps:** Biggest build gap among core pills. V4 needs: pill tap → drawer showing all in-flight messages (openers, follow-up 1, follow-up 2) with status, ability to edit/approve/send. Requires either: (1) wiring to milo-outreach, (2) building a unified draft queue in milo-for-ppc, or (3) building a new `/api/operator/pill` data source that queries across `drip_campaigns` + `ai_action_log`. The existing drawer architecture supports adding new data sources — `PillDrawer` just needs items with the `DrawerItem` interface.

---

### 4. E-sign

**a. milo-for-ppc: NONE**
- Zero signing management UI. No page, no component, no dashboard block.
- Contract review exists (`/contract-review/[id]`) but that's analysis, not signing.
- No pill for signing in any role set.

**b. MOP: EXISTS**
- `~/Mop3.18/MOP/app/signing/page.tsx` — signing document list with status badges (pending/viewed/signed/voided/draft), short codes, tokens
- `~/Mop3.18/MOP/app/sign/[token]/page.tsx` — signing page (counterparty-facing)
- `~/Mop3.18/MOP/app/review/[token]/page.tsx` — review page (counterparty-facing)
- `~/Mop3.18/MOP/app/offers/page.tsx` — offers list
- Document types: MSA, IO, W-9

**c. Data wiring:**
- Source: @milo/contract-signing primitive, `signing_documents` table on shared Supabase
- Data migrated (D54: 109 docs, 81 active tokens with `expires_at: null`)
- Redirect from MOP → tlp.justmilo.app deployed (D103)
- Counterparty-facing pages work via redirect — operator-facing list missing

**d. Presentation:** N/A in milo-for-ppc. MOP has full page with status badges and action links.

**e. Gaps:** Full build needed. V4 needs: pill tap → drawer showing signing documents grouped by status (pending, viewed, signed), each with counterparty + link. Data layer exists (shared Supabase) — only the drawer UI + pill predicate are missing. New `/api/operator/pill` data source needed (query `signing_documents` instead of `alerts`).

---

### 5. Aging

**a. milo-for-ppc: EXISTS**
- `ApAgingBlock.tsx` at `src/components/dashboard-v2/ApAgingBlock.tsx` — AP aging dashboard block
- API: `GET /api/invoices/aging` — returns aging receivables data
- `GET /api/invoices/[id]/mark-overdue` — marks invoices overdue
- Existing pills: `ap_aging`, `overdue_invoices` (billing role) — both are chat pills (no drawer predicate)

**b. MOP: PARTIAL**
- Invoice page shows status + due_date but no dedicated aging view
- `OverdueSummaryWidget` on MOP dashboard

**c. Data wiring:**
- Source: `invoices` table in milo-for-ppc Supabase (`tmzsmdkfqvjjjwkukysg`)
- LIVE — aging endpoint computes buckets from invoice due dates

**d. Presentation:** Dashboard block. Chat pills. No drawer.

**e. Gaps:** Data and API exist. V4 needs: pill badge shows total overdue $, drawer shows aging buckets (current, 30-day, 60-day, 90+) with dollar sums and drill-to-invoice. Requires new `/api/operator/pill` data source for invoices-as-drawer-items (different from the current alerts-only drawer).

---

### 6. Calls

**a. milo-for-ppc: PARTIAL → EXISTS (via drawer)**
- `FlaggedCallsBlock.tsx` at `src/components/dashboard-v2/FlaggedCallsBlock.tsx` — flagged calls
- `PublisherMonitoringBlock.tsx`, `BuyerMonitoringBlock.tsx` — call monitoring blocks
- `CallDetailModal.tsx` at `src/components/CallDetailModal.tsx` — modal for call details (already close to V4 pattern)
- `QualityBlock.tsx` — quality stats
- Sprint 2 tools: `review_call_quality`, `detect_coached_call`, `diagnose_connection_issue`
- Existing pills: `qa_queue` (publisher-qa) — has DRAWER predicate, dollar-rollup badge
- `qa_queue` drawer shows: dispute_detected + ivr_self_dq + flagged titles, ranked by revenue at risk
- No dedicated `/calls` page

**b. MOP: EXISTS**
- `~/Mop3.18/MOP/app/calls/page.tsx` — full call log: ConvoQC fields, fraud scoring, zero-revenue analysis, caller resolution, rich filtering
- `~/Mop3.18/MOP/app/convoqc-coverage/page.tsx` — ConvoQC coverage per campaign

**c. Data wiring:**
- Source: TrackDrive API (15-min sync) → `call_records` table + ConvoQC enrichment
- LIVE — call data syncs continuously, ConvoQC 37% coverage

**d. Presentation:** The `qa_queue` pill already opens a drawer with flagged/problem calls ranked by $ at risk — this IS the D85 pattern for "problem calls for operator research." The `CallDetailModal` is a center modal, not a drawer, but close.

**e. Gaps:** The `qa_queue` predicate filters alerts about calls, not `call_records` directly. V4 "Calls" pill could either: (a) keep the alert-based approach (what operators actually need), or (b) add a separate `call_records` query for a call-log drawer. Recommendation: (a) is correct for D85's "show nothing until something matters" — a call log is the firehose Mark rejected.

---

### CUSTOMIZABLE PILLS (from + picker)

---

### 7. Sales

**a. milo-for-ppc: EXISTS**
- `/pipeline` — entity list (94 entities, 8 stages: outreach → qualifying → drip → onboarding → activation → active → dormant → blacklisted), filterable, side panel
- `/pipeline-board` — drag-and-drop kanban (`@hello-pangea/dnd`), auto-compose signal
- `PipelineBlock.tsx`, `KanbanBlock.tsx` — dashboard blocks
- `EntityDetailPanel.tsx` at `src/components/dashboard-v2/EntityDetailPanel.tsx` — **sliding detail panel** with contacts, stats, blacklist status, match suggestions
- API: `/api/pipeline`, `/api/pipeline/[id]`, `/api/pipeline/[id]/stage`
- Existing pills: `pipeline_summary` (outreach), `kanban` (prospecting) — both chat pills

**b. MOP: PARTIAL**
- No pipeline or kanban. Publisher/buyer lists serve as entity management without stage tracking.

**c. Data wiring:**
- Source: `prospect_pipeline` + `contacts` + `drip_campaigns` + `action_items` tables
- LIVE — pipeline CRUD and kanban drag-drop work

**d. Presentation:** Full pages + dashboard blocks. `EntityDetailPanel` is already a sliding side panel — architecturally close to V4 drawer.

**e. Gaps:** V4 needs: pill tap → drawer showing pipeline by stage. The kanban board could be the drawer content. Main gap: the full page needs to become a drawer-launched surface. `EntityDetailPanel` survives as-is inside the drawer.

---

### 8. Publishers

**a. milo-for-ppc: PARTIAL**
- `/partners` — TrackDrive partner sync page (diff view: would_add, would_update, already_synced). NOT an operator-facing publisher management page.
- `TopPublishersBlock.tsx`, `PublisherMonitoringBlock.tsx` — dashboard blocks
- Publisher grades: `src/lib/publisher-grades.ts` + `GET /api/publishers/grades` (A-F grading)
- `EntityDetailPanel` — sliding detail for publisher entities
- Existing pill: `top_publishers` (admin, operations) — chat pill

**b. MOP: EXISTS**
- `~/Mop3.18/MOP/app/publishers/page.tsx` — full publisher list: quality data (answer rate, duration, short calls, conversions, repeat callers), filter bar, side panel, form modals, blacklist/merge, synonym management
- `~/Mop3.18/MOP/app/publishers/[id]/page.tsx` — publisher detail

**c. Data wiring:**
- `prospect_pipeline` for entities + `call_records` for stats + publisher grades in milo-for-ppc
- @milo/crm `crm_counterparties WHERE counterparty_type = 'publisher'`

**d. Presentation:** Partner sync page is a dev/ops tool. No operator-facing publisher list. MOP has the full version.

**e. Gaps:** V4 needs: pill → drawer with publisher list showing name, grade (A-F), 7d calls/revenue, primary concern. Detail drills to `EntityDetailPanel`. MOP publisher page is the reference design. Requires new drawer data source (query `prospect_pipeline` or `crm_counterparties`).

---

### 9. Buyers

**a. milo-for-ppc: PARTIAL**
- Same `/partners` page as Publishers (combined view)
- `BuyerMonitoringBlock.tsx` — buyer monitoring
- Buyer health scoring: `src/lib/buyer-health.ts` + `GET /api/buyers/health` (0-100)
- `EntityDetailPanel` — same sliding panel
- No buyer pill in any role set

**b. MOP: EXISTS**
- `~/Mop3.18/MOP/app/buyers/page.tsx` + `[id]` — full buyer list + detail
- `~/Mop3.18/MOP/app/end-buyers/page.tsx` — end-buyer list

**c. Data wiring:**
- Same as Publishers but `counterparty_type = 'buyer'`
- Buyer health scores LIVE

**d. Presentation:** Same gap as Publishers — no operator-facing buyer list.

**e. Gaps:** V4 needs: pill → drawer with buyer list showing name, health score (0-100), 7d revenue, overdue invoices. MOP buyer page is the reference. Same infrastructure need as Publishers.

---

### 10. Accounting

**a. milo-for-ppc: EXISTS**
- `/invoices` — invoice generation + workflow (draft, finalize, approve, flag, payment tracking), TD comparison
- `/jen` — Jen's invoice review queue (TD comparison, $1 threshold)
- `/tiffani` — Tiffani's payment view (mark paid, record payment)
- `/reconciliation` — our data vs TrackDrive reconciliation (buyer-level, CID-level)
- `/reports` — buyer report cross-referencing (CSV upload, column mapping)
- Dashboard blocks: `InvoiceBlock`, `WeeklyBillingBlock`, `ApAgingBlock`, `BillingPipelineBlock`, `BankTransactionsBlock`, `CommissionTrackerBlock`, `MediaRiteXRefBlock`
- API: `/api/invoices/*` (7 endpoints), `/api/reconciliation/*` (3 endpoints), `/api/buyer-reports/*` (4 endpoints), `/api/bank-transactions/*` (3 endpoints)
- Existing pills: `overdue_invoices`, `weekly_billing`, `ap_aging`, `td_balance` (billing role) — all chat pills

**b. MOP: EXISTS**
- `~/Mop3.18/MOP/app/invoices/page.tsx` — full invoice management
- `~/Mop3.18/MOP/app/reports/page.tsx` — reports

**c. Data wiring:**
- Source: `invoices`, `buyer_reports`, `buyer_report_rows`, `disputes`, `reconciliation_runs` tables
- All on milo-for-ppc Supabase (`tmzsmdkfqvjjjwkukysg`)
- LIVE — full lifecycle works

**d. Presentation:** Most built-out surface in the entire app: 5 pages + 7 dashboard blocks. All pages are full-page dashboard-style.

**e. Gaps:** V4 needs to collapse 5 pages + 7 blocks into a drawer. Suggested sub-views: AP summary, AR summary, overdue, reconciliation status. The data is 100% there — the challenge is information design.

---

### 11. Billing

**a. milo-for-ppc: EXISTS (heavy overlap with Accounting)**
- `WeeklyBillingBlock.tsx` — weekly billing summary
- `BillingPipelineBlock.tsx` — billing pipeline (draft → pending_jen → approved → sent → paid)
- `InvoiceBlock.tsx` — invoice overview
- API: invoice lifecycle endpoints (same as Accounting)
- Invoice status lifecycle: draft → pending_jen → jen_approved/jen_flagged → sent → paid
- Existing pills: `billing_snapshot` (admin), `disputes` (billing) — disputes has drawer predicate

**b. MOP: EXISTS** — same as Accounting

**c. Data wiring:** Same as Accounting — `invoices` table

**d. Presentation:** Dashboard blocks. Billing role gets 7 blocks via `ROLE_BLOCKS`.

**e. Gaps:** **Scope definition is the gap, not UI.** Billing and Accounting share the same data layer. If separate pills:
- **Billing** = invoices due, payment status, who owes what, PDF generation
- **Accounting** = AP/AR reconciliation, buyer report cross-referencing, bank transactions, commission tracking
If merged: tabbed drawer with Billing and Accounting as sub-sections. Mark must decide.

---

### 12. Q&A

**a. milo-for-ppc: EXISTS**
- `/support` — full support ticket system: user info, description, screenshot upload, Milo auto-response, admin response, status (open/in-progress/resolved), priority (low/normal/high/urgent), assigned_to
- `SupportTicketsBlock.tsx` — dashboard block
- API: `/api/support/ticket`, `/api/support/tickets`, `/api/support/tickets/[id]`, `/api/support/tickets/user`
- No existing pill for support/Q&A

**b. MOP: NONE** — no support system

**c. Data wiring:**
- Source: support tickets table on `tmzsmdkfqvjjjwkukysg`
- LIVE

**d. Presentation:** Full page + dashboard block.

**e. Gaps:** V4 needs: pill → drawer showing open tickets sorted by priority. Data and full page exist. New drawer data source needed (query support tickets as `DrawerItem`s).

---

## Cross-Cutting Findings

### The V4 shell already exists — `/operator` IS the V4 page

D114 misidentified `/operator` as "duplicate of dashboard-v2 for certain roles." That's wrong. `/operator` is the **V4 conversation-first shell** with:

1. **Briefing-first home** — no tiles, no grid. Just "Good morning" + needs-you cards.
2. **Pill bar** — role-based defaults + custom pills + `+ Add pill`.
3. **Drawer on pill tap** — 460px right-side slide-out with severity-ranked items.
4. **Chat on non-drawer pill tap** — dispatches natural-language query to Milo.
5. **Search bar** — rotating example prompts, Enter dispatches to chat.
6. **"Full dashboard →" escape hatch** — link to legacy `/dashboard-v2` at bottom.
7. **Admin executive view** — `Top5Frame` + `AdminExecutiveCards` with pill badges.

This is D85's architecture, already shipping. The gap is content breadth (9 drawer predicates vs 12 desired pills) and the V4 additions (file-drop, mic, footer sync, jiggle).

### What D85 adds beyond existing `/operator`

| D85 spec | Existing | Gap |
|---|---|---|
| Conversation-first home | MorningBriefing + SearchBar | **EXISTS** |
| Pill-fan on `+` trigger | `+ Add pill` → chat "Make a new custom pill" | **Partial** — current `+` dispatches to chat for custom pill creation, not a curated picker |
| Right drawer | PillDrawer (460px right slide-out) | **EXISTS** |
| Left drawer | — | **NEW** — no left-side drawer component exists |
| Center modal | CallDetailModal, NotificationModal, AgreedTermsForm | **EXISTS** (partial — per-component, not a generic surface) |
| Bottom sheet | — | **NEW** — no bottom sheet component exists |
| Full-screen takeover | `/contract-review/[id]`, `/pipeline`, etc. | **EXISTS** (as full pages, not as a takeover from drawer) |
| File-drop auto-routing | — | **NEW** — no file-drop handler on operator page |
| Mic input (right of search) | — | **NEW** — SearchBar is text-only |
| Footer "Live · synced" | — | **NEW** — no footer sync status |
| Long-press jiggle customization | — | **NEW** — pills are static strips, no reorder/remove UX |
| No sidebar navigation | `/operator` has no sidebar | **EXISTS** — operator page is clean |
| No dashboard tiles | `/operator` renders briefing + pills + search, no tiles | **EXISTS** |

### The "dashboard pattern" Mark rejected

Located in milo-for-ppc — this is `/dashboard-v2`, NOT `/operator`:

| File | What it does |
|---|---|
| `src/app/dashboard-v2/page.tsx` | Legacy dashboard — imports 33 block components, renders per-role grid |
| `src/lib/dashboard-v2-types.ts` | `ROLE_BLOCKS` mapping: 8 roles × up to 12 blocks each |
| `src/components/dashboard-v2/` | 32 block component files (AlertsBlock, PulseBlock, etc.) |

The `/operator` page links to `/dashboard-v2` as a "Full dashboard →" escape hatch. V4 could remove this link or keep it as power-user fallback.

### MOP dashboard — "right info, wrong presentation"

`~/Mop3.18/MOP/app/page.tsx` imports 13 widgets: QuickActions, ARSummary, APSummary, OverdueSummary, DraftInvoices, FraudAlerts, EndBuyers, CampaignApps, OutreachQueue, RecentActivity, CampaignAlerts, DailyBriefing, CallVolume, RevenueChart. Plus layout customization modals.

Every widget surfaces useful operator data. The problem was presentation (tile grid).

### Drawer data source architecture

Current `PillDrawer` is hardwired to the `alerts` table via `/api/operator/pill`. All 9 drawer-capable pills filter the same `alerts` query. This works for alert-category pills (qa_queue, disputes, needs_attention) but not for entity-category pills (Contracts, E-sign, Publishers, Buyers, Aging). V4 needs the drawer to support multiple data sources:

| Pill category | Data source | Current support |
|---|---|---|
| Alert-based (Alerts, Calls, Disputes) | `alerts` table | **EXISTS** — current pill predicates |
| Contract-based (Contracts, E-sign) | `contract_documents`, `signing_documents` | **NEW** — needs new API endpoints or pill data sources |
| Entity-based (Publishers, Buyers, Sales) | `prospect_pipeline`, `crm_counterparties` | **NEW** — needs entity-as-DrawerItem mapping |
| Financial (Aging, Accounting, Billing) | `invoices`, `reconciliation_runs` | **NEW** — needs invoice-as-DrawerItem mapping |
| Drafts | `drip_campaigns`, `ai_action_log`, or milo-outreach | **NEW** — needs draft queue data source |
| Q&A | `support_tickets` | **NEW** — needs ticket-as-DrawerItem mapping |

The `DrawerItem` interface (id, severity, title, description, entity_name, revenue_at_risk, action_label, action_prompt, created_at) is generic enough for all categories. The change is server-side: `/api/operator/pill` needs to branch on pill category and query the appropriate table.

### Pills Mark DIDN'T list that exist as surfaces

| Surface | Current UI | Data | Pill candidate? |
|---|---|---|---|
| **Disputes** | `/disputes` page + `DisputeRecoveryBlock` + 6 API endpoints + `dispute-drafter.ts` + existing drawer predicates (`dispute_recovery`, `disputes`, `disputes_signoff`) | `disputes` table, buyer report rows | **YES — strong candidate.** More built-out than Drafts or E-sign. Already has 3 drawer predicates. |
| **Campaigns** | `/campaigns` page + `CampaignHealthBlock` + auto-detect billing type | `campaigns` tables | Maybe — reference, not daily attention |
| **DID Management** | `/malvin` page + `DIDTrackerBlock` | TrackDrive DIDs | No — Malvin-specific ops tool |

### Markdown documentation per pill

| Pill | Key docs |
|---|---|
| Alerts | `SYSTEMS.md` §Alert System; `SPRINT3_TARGETS.md` §0; `docs/ROADMAP.md` Stage 3 |
| Contracts | `vendor/contract-analysis/SCHEMA.md`, `vendor/contract-signing/SCHEMA.md`, `vendor/contract-negotiation/SCHEMA.md`; MOP `docs/riders/*.md` |
| Drafts | `docs/team-skills/VEE_complete.md`, `CAM_complete.md`; MOP `docs/outreach-design.md` |
| E-sign | `vendor/contract-signing/SCHEMA.md`; MOP `docs/signing-page-refactor-plan.md` |
| Aging | `docs/billing-rules.md`; `SYSTEMS.md` §Invoice Generation |
| Calls | `docs/integration-map.md` §1-2; `docs/trackdrive-mcp/CONVOQC_INTEGRATION.md`; `SYSTEMS.md` §Call Sync |
| Sales | `SYSTEMS.md` §Pipeline Management; `docs/team-skills/CAM_complete.md`; `docs/discovery/publisher-onboarding-current-state.md` |
| Publishers | `SYSTEMS.md` §Publisher Grades; `docs/team-skills/FAB_complete.md` |
| Buyers | `SYSTEMS.md` §Buyer Health; `docs/team-skills/KENT_complete.md` |
| Accounting/Billing | `SYSTEMS.md` §Invoice Generation, §Reconciliation; `docs/billing-rules.md`; `SPRINT3_TARGETS.md` §13 |
| Q&A | No dedicated docs |

---

## Recommended V4 Pill Scope

### Ship with V4 shell (v4.0) — 8 pills

| Pill | Category | Rationale |
|---|---|---|
| **Alerts** | Core | Already V4 pattern. MorningBriefing + PillDrawer + executive ranker. Tune predicate only. |
| **Contracts** | Core | UI exists. Add contract_documents drawer data source. |
| **Aging** | Core | Block + API exist. Add invoice-aging drawer data source. |
| **Sales** | Customizable | Full pipeline + kanban + EntityDetailPanel sliding panel. Rewrap as drawer. |
| **Accounting** | Customizable | Most built-out surface (5 pages). Collapse into tabbed drawer. |
| **Q&A** | Customizable | Full page exists. Add support-ticket drawer data source. |
| **Calls** | Core | qa_queue drawer already works with $ badge. Rename/retune for "Calls" pill. |
| **Disputes** (ADD) | Customizable | 3 existing drawer predicates + full page + API + AI drafter. Essential revenue recovery. |

### Defer to v4.1 — 4 pills

| Pill | Category | Rationale |
|---|---|---|
| **E-sign** | Core | Zero operator UI. Full build: drawer data source + signing_documents query. Data layer ready. |
| **Drafts** | Core | Partial blocks only. Needs unified draft queue — significant build or milo-outreach wiring. |
| **Publishers** | Customizable | No operator-facing list. Needs entity-as-DrawerItem mapping from pipeline or CRM. |
| **Buyers** | Customizable | Same as Publishers. |

### Merge decision needed from Mark

- **Billing**: merge into Accounting (as sub-tab) or redefine scope boundary

### V4 shell changes needed (D85 additions beyond existing `/operator`)

| Change | Effort | Dependency |
|---|---|---|
| Curated `+` picker (replace chat-based custom pill creation) | Medium | Pill list finalized |
| Left drawer component | Medium | Surface type definitions |
| Bottom sheet component | Medium | Surface type definitions |
| File-drop auto-routing on operator page | Medium | File type → pill mapping |
| Mic input right of SearchBar | Low | Web Speech API or Whisper |
| Footer "Live · synced" with troubleshoot escalation | Low | Health-check endpoint exists |
| Long-press jiggle for pill reorder/customization | Medium | Custom pill persistence (already exists in user_profiles.custom_pills) |
| Multi-data-source drawer (not just alerts) | **High** | Core V4 infrastructure — enables non-alert pills |

**The highest-leverage single change: make `/api/operator/pill` support multiple data sources.** Everything else is incremental UI work on an existing shell.

---

## Surprise Flag

**The V4 architecture is 70% built.** D114 (and Mark's D85 directive) framed V4 as a "radical departure." It's not — `/operator` already ships the conversation-first, pill-fan, drawer-based pattern. The "Full dashboard →" link at the bottom is the only remaining thread to the old model. V4 is an expansion of existing infrastructure (more pill types, more drawer data sources, new input modalities), not a greenfield build.
