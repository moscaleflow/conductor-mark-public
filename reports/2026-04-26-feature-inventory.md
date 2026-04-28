# Milo-for-PPC: Master Feature Inventory

**Date:** 2026-04-26
**Author:** Coder-3 (Research Lane)
**Scope:** Every page, API route, component, and workflow in `milo-for-ppc`

---

## Legend

| Status | Meaning |
|---|---|
| **WIRED** | Functional, connected to real data (Supabase + TrackDrive + Claude) |
| **PARTIAL** | Core works but has gaps, missing sub-features, or incomplete wiring |
| **PLACEHOLDER** | UI exists but not connected to real data or backend |
| **MISSING** | Needed for the workflow but not yet built |

---

## I. WORKFLOW STAGE: ASK / VET

### Feature: /ask Page (Vet/Sales/Publisher/Buyer Chat)
- **Status:** WIRED
- **Location:** `src/app/ask/page.tsx`, `src/app/api/ask/route.ts`
- **Data source:** Anthropic Claude (Sonnet for research, Haiku for extraction/classification), `blacklist_entries`, `vet_results`, `chat_logs`, `crm_counterparties`, `ask_rate_limits`
- **Notes:** Full multi-mode LLM chat with 4 pills (Vet, Sales, Publisher, Buyer) plus auto-classify. Supports file drag-drop (PDF/DOCX contracts, images), text paste blocks, SSE streaming. Rate limiting per IP. Conversation history (16K char cap) carried across turns.

### Feature: Vet Pill - Entity Vetting
- **Status:** WIRED
- **Location:** `src/app/api/ask/route.ts`, `src/lib/ask/prompts/vet.ts`, `src/lib/ask/vet-extraction.ts`
- **Data source:** Claude (Haiku extraction + parallel Sonnet research with web_search tool), `blacklist_entries`, `vet_results`, `crm_counterparties`
- **Notes:** Two-stage pipeline: (1) Haiku extracts entity names from pasted text, (2) parallel Sonnet researches each entity with web search. Blacklist pre-check on every entity. CRM context injection. Vet result caching/dedup (skips re-research if fresh result exists). Secondary name classification (team/crm/blacklisted/unknown). Results saved to `vet_results` table.

### Feature: Vet Cards (EntityVetCard)
- **Status:** WIRED
- **Location:** `src/components/ask/EntityVetCard.tsx`
- **Data source:** Vet results from `/api/ask`
- **Notes:** Verdict badge (likely_real/likely_scam/unclear/blacklisted), facts grid, flags with severity, recommendation. Action buttons: +CRM, Flag Scam, Assign, Ask More, Re-Vet, Edit/Save, Dismiss. All actions call `/api/ask/vet-action`. Edit mode allows inline modification of entity name/type, facts, flags, recommendation.

### Feature: Secondary Name Stubs
- **Status:** WIRED
- **Location:** `src/components/ask/EntityVetCard.tsx` (SecondaryNameStubs export)
- **Data source:** Classified from `/api/ask` route
- **Notes:** Shows "Also mentioned" names with classification badges (team/crm/blacklisted/unknown). "Vet This" button pre-fills input.

### Feature: Sales Pill - Outreach Drafting
- **Status:** WIRED
- **Location:** `src/app/api/ask/route.ts`, `src/lib/ask/prompts/sales.ts`
- **Data source:** Claude (Sonnet), web_search tool, CRM context
- **Notes:** SSE streaming response. CRM context injected. Conversation history support.

### Feature: Publisher Pill
- **Status:** WIRED
- **Location:** `src/app/api/ask/route.ts`, `src/lib/ask/prompts/publisher.ts`
- **Data source:** Claude (Sonnet), web_search tool, CRM context
- **Notes:** SSE streaming. Same infrastructure as Sales pill with publisher-specific system prompt.

### Feature: Buyer Pill
- **Status:** WIRED
- **Location:** `src/app/api/ask/route.ts`, `src/lib/ask/prompts/buyer.ts`
- **Data source:** Claude (Sonnet), web_search tool, CRM context
- **Notes:** SSE streaming. Same infrastructure as Sales pill with buyer-specific system prompt.

### Feature: Auto-Classify (no pill selected)
- **Status:** WIRED
- **Location:** `src/app/api/ask/route.ts` (classifyInput function)
- **Data source:** Claude (Haiku) with few-shot examples
- **Notes:** When user submits without selecting a pill, Haiku classifies intent. Low-confidence returns a clarifier message. High-confidence auto-routes to the correct pill.

### Feature: File Classifier (drag-drop)
- **Status:** WIRED
- **Location:** `src/app/api/ask/classify-file/route.ts`, `src/components/ask/ClassifierConfirm.tsx`
- **Data source:** Claude
- **Notes:** Classifies dropped files by type (contract, screenshot, etc.). High-confidence auto-fires. Cross-pill conflict shows ClassifierConfirm dialog.

### Feature: Contract Analysis (via /ask)
- **Status:** WIRED
- **Location:** `src/app/api/ask/route.ts` (contract path), `@milo/contract-analysis`
- **Data source:** Claude (Sonnet), pdf-parse, mammoth (DOCX)
- **Notes:** Extracts text from PDF/DOCX, analyzes via Claude for issues (CRITICAL/HIGH/MEDIUM/LOW). Returns structured IssueListExpandable component. Analysis context feeds into follow-up conversation.

### Feature: Ask Rating (Thumbs Up/Down)
- **Status:** WIRED
- **Location:** `src/app/api/ask/rating/route.ts`
- **Data source:** `chat_logs` table (rating column)
- **Notes:** Per-response thumbs up/down. Patches chat_logs.rating.

### Feature: Vet Actions (+CRM, Flag Scam, Assign)
- **Status:** WIRED
- **Location:** `src/app/api/ask/vet-action/route.ts`
- **Data source:** `prospect_pipeline`, `blacklist_entries`, `vet_results`
- **Notes:** `add_crm` creates prospect_pipeline entry + contact. `add_blacklist` adds to blacklist_entries. `assign` updates vet_results.assigned_to. `update_vet_result` saves inline edits.

---

## II. WORKFLOW STAGE: PIPELINE / CRM

### Feature: Pipeline Board (Kanban)
- **Status:** WIRED
- **Location:** `src/app/pipeline-board/page.tsx`
- **Data source:** `/api/pipeline`, `/api/action-items`, `/api/blacklist`
- **Notes:** 6-column Kanban (Vetted, Outreach, Qualifying, Onboarding, Activation, Active). Drag-drop stage changes with confirmation modal. Entity cards show type, vertical, days-in-stage, next-action hints. Fuzzy search (Fuse.js). Sort by days/name/activity. Dormant and blacklisted views. Clicking a card opens EntityDetailPanel.

### Feature: Pipeline List (Table + Board views)
- **Status:** WIRED
- **Location:** `src/app/pipeline/page.tsx`
- **Data source:** `/api/pipeline?include7d=true`, `/api/publishers/grades`, `/api/buyers/health`, Supabase REST direct (contacts, conversation_captures, action_items)
- **Notes:** Dual view (table + board). Table shows calls_7d, revenue_7d, grade/health_score. Expandable rows with contact info, blacklist screening, campaign matches, conversation history, stage-change buttons. Board view delegates to KanbanBlock. Tasks tab shows action items grouped by entity. Screenshot drop-to-capture.

### Feature: Pipeline API
- **Status:** WIRED
- **Location:** `src/app/api/pipeline/route.ts`, `src/app/api/pipeline/[id]/route.ts`, `src/app/api/pipeline/[id]/stage/route.ts`
- **Data source:** `prospect_pipeline` table
- **Notes:** GET with filtering (stage, type, search), optional 7d call stats join. PATCH for field updates. Stage change with history logging.

### Feature: Entity Detail Panel (15 sections)
- **Status:** WIRED (most sections), PARTIAL (some)
- **Location:** `src/components/shared/EntityDetailPanel.tsx` (4,406 lines)
- **Data source:** `/api/entities/[name]`, `/api/billing-schedules`, `/api/ppcrm`, `/api/contract/documents`, `/api/outreach/draft`, `/api/research/entity`, `/api/contacts`, `/api/onboard/generate-docs`, `/api/onboard/doc-status`, `/api/creative/analyze`, `/api/team-members`, `/api/action-items`, `/api/milo-activity`

#### Section-by-section breakdown:

1. **Identity / Contact** - WIRED. Shows name, role, email, phone, preferred channel, LinkedIn/website/Facebook links, research status. Inline edit form (create or update contact). Entity verification badge with match selector and custom context re-verify.

2. **Intelligence** - WIRED. Blacklist status (clear/flagged with match details), risk flags from prospect_pipeline, lead score bar.

3. **Vet Intelligence** - WIRED. Latest vet result: verdict badge, summary, flags with severity dots, facts 2-col grid, recommendation, assigned_to. Collapsible previous vets history.

4. **Performance (7d)** - WIRED (stage-gated: activation/active/dormant). 2x2 grid: calls, revenue, answer rate, flag rate. All from call_records.

5. **Recent Flags** - WIRED (stage-gated). Flag type summary badges + individual flagged call cards with time, duration, flags, ConvoQC summary.

6. **Campaign Lines** - WIRED (stage-gated). Lists campaign_routes with buyer name, billing type, vertical, payout, DID, status dot.

7. **Agreed Terms** - WIRED (stage-gated: qualifying/onboarding). AgreedTermsForm component for capturing deal terms (billing_type, rate, caps, states, qualifiers).

8. **Creative Review** - WIRED (stage-gated). Creative submissions list with qualifier match scores. Submit new script for analysis. Expandable detail with AI analysis, compliance flags, reviewer notes.

9. **Documents (PPCRM)** - WIRED (stage-gated: onboarding/activation/active). Fetches partner from PPCRM, shows documents with status/signing links. Generate MSA/W9/IO buttons. Void document with confirmation + reason. Onboarding progress steps. Go Live button.

10. **Contracts (Versioned)** - WIRED (stage-gated). Groups contract_documents by contract_group_id. Version timeline with status, risk level, flagged clause count, clause decisions (accepted/rejected/countered). Links to /contract-review/[id].

11. **Onboarding** - WIRED (stage-gated). "Start Onboarding" form: entity type, contact name/email, document type checkboxes (MSA/W9/IO). Generates docs via /api/onboard/generate-docs, creates PPCRM partner, advances pipeline stage, creates action item for Tiffani. Doc status checker.

12. **Conversations** - WIRED (stage-gated). Lists conversation_captures with source badge, summary, action items, scam flags. Expandable full context.

13. **Billing Schedule** - WIRED (stage-gated: active). Shows/edits billing cycle, payment terms, next invoice date via /api/billing-schedules.

14. **Invoices** - WIRED (stage-gated: active). Total outstanding, overdue count, last paid. Recent invoices list with type, amount, due date, status badges.

15. **Actions** - WIRED (starts expanded). Buttons: Draft Outreach (opens compose section), Research (runs /api/research/entity with entity verification), Flag as Scam (moves to blacklisted + adds blacklist entry), Start Onboarding.

#### Outreach Compose (sub-section of Actions):
- **Status:** WIRED
- **Notes:** Platform selector (email/teams/linkedin), tone selector (professional/casual/direct). Claude generates draft with real data enrichment (30d performance, payment profile, pipeline info). Copy to clipboard + Save as action item (auto-assigns to Cam or Vee based on entity type).

---

## III. WORKFLOW STAGE: OUTREACH

### Feature: Outreach Draft API
- **Status:** WIRED
- **Location:** `src/app/api/outreach/draft/route.ts`
- **Data source:** Claude (Sonnet), `call_records` (30d performance), `invoices` (payment profile), `prospect_pipeline`
- **Notes:** Identity-based persuasion engine with fabrication refusal. Platform-aware formatting (email with subject, LinkedIn under 300 chars, Teams casual). Enriches with real entity data.

### Feature: Outreach Save as Action Item
- **Status:** WIRED
- **Location:** EntityDetailPanel outreach compose
- **Data source:** `/api/action-items`, `/api/team-members`
- **Notes:** Saves draft text to action_items with auto-owner assignment (publishers -> Cam, buyers -> Vee).

---

## IV. WORKFLOW STAGE: QUALIFYING

### Feature: Agreed Terms Form
- **Status:** WIRED
- **Location:** `src/components/shared/AgreedTermsForm.tsx`, `src/app/api/agreed-terms/route.ts`
- **Data source:** `agreed_terms` table
- **Notes:** Captures billing_type, rate, publisher_share, duration_seconds, states, excluded_states, caps (concurrency/daily/monthly), qualifiers, schedule_notes, special_terms, vertical.

### Feature: Creative Analysis
- **Status:** WIRED
- **Location:** `src/app/api/creative/analyze/route.ts`, `src/app/api/creative/[id]/review/route.ts`
- **Data source:** Claude, `creative_submissions` table
- **Notes:** Script analysis with qualifier matching, compliance checks, tone assessment. Reviewer can approve/request-revision.

---

## V. WORKFLOW STAGE: ONBOARDING (Contracts / IOs)

### Feature: Document Generation (MSA/W9/IO)
- **Status:** WIRED
- **Location:** `src/app/api/onboard/generate-docs/route.ts`
- **Data source:** PPCRM API (via `/api/ppcrm`), `prospect_pipeline`, `agreed_terms`, `contacts`
- **Notes:** Creates partner in PPCRM if not found. Generates documents via MOP API. For IOs, injects agreed_terms merge fields (billing_type, rate, caps, states, qualifiers). Creates action item for Tiffani. Advances stage to onboarding.

### Feature: E-Signing
- **Status:** WIRED (via @milo/contract-signing)
- **Location:** `vendor/contract-signing/`, `src/app/api/ppcrm/route.ts`, `src/app/api/onboard/generate-docs/route.ts`
- **Data source:** PPCRM/MOP e-signing infrastructure
- **Notes:** Token-based signing with short URLs. Signing status tracked. Welcome experience (`/welcome/[slug]`) for new partners.

### Feature: Contract Analysis + Negotiation
- **Status:** WIRED
- **Location:** `src/app/contract-review/[id]/page.tsx`, `src/app/api/contract/analyze-bg/route.ts`, `src/app/api/contract/analyze-proxy/route.ts`, `src/app/api/contract/negotiate/route.ts`, `src/app/api/contract/process/route.ts`
- **Data source:** `contract_documents`, Claude, `@milo/contract-analysis`, `@milo/contract-negotiation`
- **Notes:** Full contract review page with flagged clauses, risk levels, inline redline editor, negotiation state machine. Version comparison across contract_group_id.

### Feature: Contract Clause Decisions
- **Status:** WIRED
- **Location:** `src/components/operator/ContractClauseList.tsx`, `src/app/api/operator/clauses/route.ts`
- **Data source:** `contract_documents.analysis_result`
- **Notes:** Accept/reject/counter per clause. Decisions stored in analysis_result.decisions.

### Feature: PPCRM Integration
- **Status:** WIRED
- **Location:** `src/app/api/ppcrm/route.ts`
- **Data source:** PPCRM Supabase (separate project: jdzqkaxmnqbboqefjolf)
- **Notes:** Partner search, document generation/cancellation, onboarding status, go-live activation. Cross-project Supabase queries.

### Feature: Document Status Tracking
- **Status:** WIRED
- **Location:** `src/app/api/onboard/doc-status/route.ts`
- **Data source:** PPCRM documents table
- **Notes:** Shows sent/signed status per document type.

---

## VI. WORKFLOW STAGE: ACTIVE (Operations)

### Feature: Operator Page (Landing)
- **Status:** WIRED
- **Location:** `src/app/operator/page.tsx`
- **Data source:** `/api/operator`, `/api/operator/executive`, `/api/operator/pill`
- **Notes:** Role-based views. Admin sees Top5Frame with executive cards (ranked by priority). Non-admin sees MorningBriefing with needs-you cards + PillBar. Pill drawer system with 20+ drawer-type pills. Custom pill creation. Guided operator tour for first-time users.

### Feature: Operator Executive View (Admin)
- **Status:** WIRED
- **Location:** `src/components/AdminExecutiveCards.tsx`, `src/components/Top5Frame.tsx`
- **Data source:** `/api/operator/executive`
- **Notes:** 5 priority-ranked cards with actionable prompts. Summary pills showing counts/dollar amounts.

### Feature: Morning Briefing
- **Status:** WIRED
- **Location:** `src/components/operator/MorningBriefing.tsx`, `src/app/api/briefing/morning/route.ts`
- **Data source:** Claude, alerts, pipeline data
- **Notes:** Personalized greeting, date line, summary, needs-you cards with action prompts.

### Feature: Pill Bar + Pill Drawer
- **Status:** WIRED
- **Location:** `src/components/operator/PillBar.tsx`, `src/components/operator/PillDrawer.tsx`, `src/app/api/operator/pill/route.ts`
- **Data source:** Multiple Supabase tables per pill predicate
- **Notes:** Role-default pills + user custom pills. Badge counts. Drawer opens with filtered list for: QA queue, QC queue, ping summary, disputes, calls, aging, contracts, sales pipeline, accounting, publishers, buyers, etc. Snooze capability.

### Feature: Milo Chat (Conversational AI)
- **Status:** WIRED
- **Location:** `src/components/ChatInterface.tsx`, `src/app/api/milo/route.ts`, `src/app/api/milo/stream/route.ts`
- **Data source:** Claude, Supabase tools
- **Notes:** Full conversational Milo with tool use (pipeline queries, entity lookup, action items, alerts, etc.). Streaming responses.

### Feature: Campaigns Management
- **Status:** WIRED
- **Location:** `src/app/campaigns/page.tsx`, `src/app/api/campaigns/route.ts`, `src/app/api/campaigns/[id]/route.ts`, `src/app/api/campaigns/auto-detect/route.ts`
- **Data source:** `campaigns` table
- **Notes:** List all campaigns with inline editing (billing_type, publisher_rate, buyer_rate, duration_threshold, convoqc_enabled). Auto-detect billing type feature.

### Feature: Call Data Sync (TrackDrive)
- **Status:** WIRED
- **Location:** `src/app/api/sync/calls/route.ts`, `src/app/api/sync/campaigns/route.ts`, `src/app/api/sync/campaign-rates/route.ts`, `src/app/api/sync/pings/route.ts`, `src/app/api/sync/convoqc/route.ts`
- **Data source:** TrackDrive API, ConvoQC API, `call_records`, `campaigns`, `campaign_routes`, `ping_post_logs`
- **Notes:** Full sync pipeline: calls, campaigns, campaign rates, pings, ConvoQC enrichment. Billing type-aware payout correction. Cron-triggered via `/api/cron/sync`.

### Feature: Reconciliation
- **Status:** WIRED
- **Location:** `src/app/reconciliation/page.tsx`, `src/app/api/reconciliation/route.ts`, `src/app/api/reconciliation/run/route.ts`, `src/app/api/reconciliation/mediarite/route.ts`
- **Data source:** TrackDrive API, `call_records`, `alerts`
- **Notes:** Compares our call_records vs TrackDrive data per buyer. Shows mismatched CIDs (conversion_mismatch, revenue_mismatch). MediaRite cross-reference. Auto-generates alerts for discrepancies. Cron-triggered.

### Feature: Live Stats
- **Status:** WIRED
- **Location:** `src/app/api/live-stats/route.ts`, `src/app/api/live-stats/pulse/route.ts`
- **Data source:** `call_records`, `campaigns`
- **Notes:** Real-time call volume, revenue, performance metrics.

---

## VII. WORKFLOW STAGE: BILLING / INVOICING

### Feature: Invoice Generation
- **Status:** WIRED
- **Location:** `src/app/invoices/page.tsx`, `src/app/api/invoices/generate/route.ts`, `src/app/api/invoices/finalize/route.ts`
- **Data source:** `call_records`, `invoices`, `billing_schedules`, `campaign_routes`
- **Notes:** Generate invoices for a period (publisher AP + buyer AR). Billing-type aware (CPA, RTB, CPL, CPQL). TD comparison for variance checking. Batch finalize. Per-invoice approve/flag workflow.

### Feature: Invoice List + Management
- **Status:** WIRED
- **Location:** `src/app/invoices/page.tsx`, `src/app/api/invoices/route.ts`, `src/app/api/invoices/[id]/approve/route.ts`, `src/app/api/invoices/[id]/flag/route.ts`, `src/app/api/invoices/[id]/mark-overdue/route.ts`, `src/app/api/invoices/[id]/record-payment/route.ts`
- **Data source:** `invoices` table
- **Notes:** Filter by status, type, entity. Approve, flag, mark overdue, record payment.

### Feature: Invoice PDF Generation
- **Status:** WIRED
- **Location:** `src/app/api/invoices/[id]/pdf/route.ts`
- **Data source:** `invoices` table
- **Notes:** Generates downloadable PDF for an invoice.

### Feature: Billing Schedules
- **Status:** WIRED
- **Location:** `src/app/api/billing-schedules/route.ts`
- **Data source:** `billing_schedules` table
- **Notes:** Per-entity billing cycle (weekly/biweekly/monthly), payment terms, next invoice date. Editable from EntityDetailPanel.

### Feature: Aging Report
- **Status:** WIRED
- **Location:** `src/app/api/invoices/aging/route.ts`
- **Data source:** `invoices` table
- **Notes:** Outstanding invoices bucketed by age (current, 30, 60, 90+).

### Feature: Billing Prepare Cron
- **Status:** WIRED
- **Location:** `src/app/api/cron/billing-prepare/route.ts`
- **Data source:** `billing_schedules`, `call_records`, `invoices`
- **Notes:** Auto-generates draft invoices based on billing schedules.

---

## VIII. WORKFLOW STAGE: DISPUTES

### Feature: Disputes Page
- **Status:** WIRED
- **Location:** `src/app/disputes/page.tsx`, `src/app/api/disputes/route.ts`, `src/app/api/disputes/stats/route.ts`
- **Data source:** `disputes` table
- **Notes:** Full dispute management: list with status filtering, revenue-at-risk sorting. Stats dashboard (counts by status, recovered revenue, top buyers). Individual dispute detail with raise/respond/resolve workflow.

### Feature: Dispute Actions
- **Status:** WIRED
- **Location:** `src/app/api/disputes/[id]/raise/route.ts`, `src/app/api/disputes/[id]/respond/route.ts`, `src/app/api/disputes/batch-raise/route.ts`
- **Data source:** `disputes` table, Resend (email)
- **Notes:** Raise dispute (sends email to buyer), record buyer response, batch raise. Draft initial message using dispute-drafter lib.

### Feature: Auto-Detect Disputes
- **Status:** WIRED
- **Location:** `src/app/api/cron/detect-disputes/route.ts`
- **Data source:** `call_records`, `disputes`
- **Notes:** Cron job detects short-duration calls below buyer thresholds and creates dispute records.

---

## IX. WORKFLOW STAGE: BLACKLIST

### Feature: Blacklist CRUD
- **Status:** WIRED
- **Location:** `src/app/api/blacklist/route.ts`
- **Data source:** `blacklist_entries` table, `ai_action_log`
- **Notes:** GET (list active), POST (add with severity/evidence), DELETE (soft-delete). Actions logged to ai_action_log.

### Feature: Blacklist Screening
- **Status:** WIRED
- **Location:** `src/app/api/blacklist/screen/route.ts`, `@milo/blacklist`
- **Data source:** `blacklist_entries`
- **Notes:** Screens entity name against blacklist with fuzzy matching, alias checking. Used by /ask (pre-check), /pipeline (expand detail), EntityDetailPanel.

### Feature: Flag as Scam (from EntityDetailPanel)
- **Status:** WIRED
- **Location:** EntityDetailPanel handleFlagScam
- **Data source:** `/api/pipeline/[id]/stage`, `/api/blacklist`
- **Notes:** Moves entity to blacklisted stage + adds blacklist_entries record.

---

## X. ROLE-SPECIFIC VIEWS

### Feature: Jen View (Billing/AR)
- **Status:** WIRED
- **Location:** `src/app/jen/page.tsx`, `src/components/JenView.tsx`
- **Notes:** Jen (billing role) dedicated view.

### Feature: Malvin View (Publisher QA)
- **Status:** WIRED
- **Location:** `src/app/malvin/page.tsx`, `src/components/MalvinView.tsx`
- **Notes:** Malvin (publisher QA role) dedicated view.

### Feature: Tiffani View (Onboarding/Contracts)
- **Status:** WIRED
- **Location:** `src/app/tiffani/page.tsx`, `src/components/TiffaniView.tsx`
- **Notes:** Tiffani (onboarding role) dedicated view.

---

## XI. ADMIN / OPS FEATURES

### Feature: Admin Impersonation
- **Status:** WIRED
- **Location:** `src/app/api/admin/impersonate/route.ts`, `src/app/api/admin/stop-impersonate/route.ts`, `src/app/api/admin/impersonation-state/route.ts`
- **Data source:** Cookies, `user_profiles`
- **Notes:** Admin can impersonate any user. Banner shows during impersonation.

### Feature: Admin Skills Management
- **Status:** WIRED
- **Location:** `src/app/api/admin/skills/route.ts`, `src/app/api/admin/skills/[id]/route.ts`, `src/components/admin/AdminSkillBar.tsx`
- **Data source:** `milo_skills` table
- **Notes:** CRUD for Milo's skill definitions.

### Feature: Admin User Management
- **Status:** WIRED
- **Location:** `src/app/api/admin/users/route.ts`
- **Data source:** `user_profiles` table
- **Notes:** List all users.

### Feature: EOD Reports
- **Status:** WIRED
- **Location:** `src/app/eod-reports/page.tsx`, `src/app/api/eod/route.ts`, `src/app/api/eod/generate/route.ts`, `src/app/api/eod/submit/route.ts`, `src/app/api/eod/batch/route.ts`
- **Data source:** `eod_reports`, `milo_conversations`, Claude
- **Notes:** AI-generated end-of-day reports with activity summaries. Batch generation for all team members. Manual submit option.

### Feature: Observatory (Live Monitor)
- **Status:** WIRED
- **Location:** `src/app/observatory/page.tsx`, `src/components/ConversationFeed.tsx`, `src/app/api/observatory/route.ts`
- **Data source:** Supabase realtime (`milo_conversations`), `call_records`
- **Notes:** Admin-only. Real-time conversation feed with filters (user, time range). Aggregated stats.

### Feature: Health Check
- **Status:** WIRED
- **Location:** `src/app/api/health-check/route.ts`, `src/app/api/health-check/summary/route.ts`, `src/app/api/cron/health-check/route.ts`
- **Data source:** Various tables (connectivity checks)
- **Notes:** System health dashboard. Checks DB connectivity, TrackDrive, cron freshness.

### Feature: Alerts System
- **Status:** WIRED
- **Location:** `src/app/api/alerts/route.ts`, `src/app/api/alerts/[id]/resolve/route.ts`, `src/app/api/alerts/generate/route.ts`
- **Data source:** `alerts` table
- **Notes:** Critical/warning/info alerts. Auto-generated by reconciliation, health check, etc. Resolvable from operator view.

### Feature: Action Items
- **Status:** WIRED
- **Location:** `src/app/api/action-items/route.ts`, `src/app/api/action-items/[id]/route.ts`, `src/app/api/action-items/from-alert/route.ts`
- **Data source:** `action_items` table
- **Notes:** Task management with priority, owner, due date, source tracking. Created by outreach saves, onboarding, alerts.

### Feature: Team Status
- **Status:** WIRED
- **Location:** `src/app/api/team/status/route.ts`, `src/app/api/team/force-clockout/route.ts`
- **Data source:** `team_status` table
- **Notes:** Clock in/out tracking. Force clockout.

### Feature: Morning Assignments
- **Status:** WIRED
- **Location:** `src/app/api/morning-assignments/route.ts`, `src/app/api/cron/morning-assignments/route.ts`
- **Data source:** `action_items`, `prospect_pipeline`, `alerts`
- **Notes:** AI-prioritized daily task assignments per team member.

### Feature: Review / QA Testing
- **Status:** WIRED
- **Location:** `src/app/review/page.tsx`, `src/app/api/review/route.ts`
- **Data source:** Claude, `chat_logs`
- **Notes:** Persona-based regression testing for /ask. Runs queries from persona library, checks accuracy, tracks growth. OpsHealth widget.

### Feature: Feedback Dashboard
- **Status:** WIRED
- **Location:** `src/app/feedback/page.tsx`, `src/app/api/milo-feedback/route.ts`, `src/app/api/feedback/record/route.ts`
- **Data source:** `milo_feedback`, `feedback_signals` tables
- **Notes:** Lists positive/negative feedback with user, query, response preview. Polymorphic feedback via @milo/feedback.

### Feature: Support Tickets
- **Status:** WIRED
- **Location:** `src/app/support/page.tsx`, `src/app/api/support/ticket/route.ts`, `src/app/api/support/tickets/route.ts`
- **Data source:** `support_tickets` table
- **Notes:** User-submitted tickets with screenshot URL, context. Admin response. Priority/status management.

---

## XII. EXTERNAL-FACING / DEMO

### Feature: Hero Landing Page (/)
- **Status:** WIRED
- **Location:** `src/app/page.tsx`
- **Data source:** `/api/demo/analyze`, `/api/demo/capture`
- **Notes:** Public-facing demo funnel. Role selection (broker/publisher/buyer/other) -> contract drop -> analysis results -> email capture. Sample contracts provided. Typewriter animation, narration lines.

### Feature: Proof Page (10 Real Analyses)
- **Status:** WIRED
- **Location:** `src/app/proof/page.tsx`
- **Data source:** Static JSON (`public/marketing/real-contracts.json`)
- **Notes:** Displays 10 anonymized real contract analyses as social proof.

### Feature: Welcome Experience
- **Status:** WIRED
- **Location:** `src/app/welcome/[slug]/page.tsx`, `src/app/welcome/[slug]/WelcomeExperience.tsx`
- **Data source:** `/api/welcome/[slug]/claim`
- **Notes:** Partner welcome page after signing. Claim account flow.

### Feature: Publisher Self-Onboarding
- **Status:** WIRED
- **Location:** `src/app/api/publisher-onboarding/route.ts`, `src/app/api/onboard/signup/route.ts`
- **Data source:** `prospect_pipeline`, `contacts`
- **Notes:** External publisher signup flow.

---

## XIII. DATA INTEGRATIONS

### Feature: TrackDrive Integration
- **Status:** WIRED
- **Location:** `src/lib/trackdrive.ts`, `src/app/api/sync/calls/route.ts`, `src/app/api/td/create-entity/route.ts`
- **Data source:** TrackDrive REST API
- **Notes:** Call data sync, campaign sync, campaign rate sync, ping data sync. TD change detection cron. Create entities in TD from Milo.

### Feature: ConvoQC Integration
- **Status:** WIRED
- **Location:** `src/lib/convoqc.ts`, `src/app/api/sync/convoqc/route.ts`
- **Data source:** ConvoQC API
- **Notes:** Enriches call_records with QC flags, dispositions, summaries.

### Feature: Supabase (Database)
- **Status:** WIRED
- **Location:** `src/lib/supabase-admin.ts`, `src/lib/supabase.ts`, `src/lib/supabase-browser.ts`
- **Data source:** Supabase project tappyckcteqgryjniwjg
- **Notes:** Primary data store. 20+ tables. Admin client (service role) for API routes, browser client for direct queries.

### Feature: Anthropic Claude (LLM)
- **Status:** WIRED
- **Location:** `@milo/ai-client`, used by ~30 API routes
- **Data source:** Anthropic API (Sonnet for structured, Haiku for classify/extract)
- **Notes:** Tiered model selection. Extended thinking. Vision support. Streaming. Retry with backoff. Web search tool.

### Feature: Resend (Email)
- **Status:** WIRED
- **Location:** `src/lib/email.ts` (likely), disputes raise
- **Data source:** Resend API
- **Notes:** Used for dispute emails to buyers. Package in dependencies.

### Feature: Google APIs
- **Status:** PARTIAL (installed, usage unclear)
- **Location:** package.json dependency
- **Notes:** `googleapis` is installed. Need to verify if actively used for Gmail/Calendar/Drive integration.

---

## XIV. @milo/* PACKAGES (all actively used)

| Package | Status | What it provides |
|---|---|---|
| `@milo/ai-client` | WIRED | Unified Claude client with tiers, streaming, retry, vision |
| `@milo/blacklist` | WIRED | Cross-reference screening, fuzzy matching, alias checking |
| `@milo/contract-analysis` | WIRED | Rule-based contract analysis, scoring, text extraction |
| `@milo/contract-negotiation` | WIRED | Multi-round negotiation state machine, tokenized review links |
| `@milo/contract-signing` | WIRED | E-signing with tokens, HMAC webhooks, audit trails, templates |
| `@milo/crm` | WIRED | Contact/counterparty/lead/activity CRUD |
| `@milo/feedback` | WIRED | Polymorphic feedback signals (thumbs, ratings) |
| `@milo/onboarding` | WIRED | DAG step state machine, flow snapshots, stall detection |

---

## XV. CRON JOBS

| Route | Status | What it does |
|---|---|---|
| `/api/cron/sync` | WIRED | Orchestrates call + campaign sync from TrackDrive |
| `/api/cron/billing-prepare` | WIRED | Auto-generates draft invoices from billing_schedules |
| `/api/cron/daily-validation` | WIRED | Validates data consistency |
| `/api/cron/detect-disputes` | WIRED | Auto-detects short-duration disputes |
| `/api/cron/evaluate` | WIRED | Performance evaluation |
| `/api/cron/health-check` | WIRED | System health monitoring |
| `/api/cron/mediarite-xref` | WIRED | Cross-reference with MediaRite |
| `/api/cron/morning-assignments` | WIRED | AI task prioritization for team |
| `/api/cron/partner-sync` | WIRED | Sync partners from TrackDrive |
| `/api/cron/reconciliation` | WIRED | Daily reconciliation vs TrackDrive |
| `/api/cron/td-change-detection` | WIRED | Detect changes in TrackDrive config |
| `/api/cron/demo-cleanup` | WIRED | Clean up demo data |

---

## XVI. FEATURES THAT ARE MISSING (needed but not built)

### Missing: Actual Email Sending from Outreach
- **Status:** MISSING
- **Notes:** Outreach drafts are generated and saved as action items, but there is no "Send Email" button that actually delivers via SMTP/Resend. The operator must copy the draft and send manually.

### Missing: Calendar/Meeting Scheduling
- **Status:** MISSING
- **Notes:** No integration for booking calls or meetings with prospects. Would complement the outreach flow.

### Missing: Automated Drip Sequences
- **Status:** MISSING
- **Notes:** The "drip" stage exists in the pipeline but there is no automated drip email/message sequence engine. Manual follow-ups only.

### Missing: AP/AR Payment Processing
- **Status:** MISSING
- **Notes:** Invoices can be generated, approved, and marked as paid, but there is no ACH/wire/check integration for actually processing payments. Manual payment recording only.

### Missing: Publisher Performance Dashboard (standalone)
- **Status:** MISSING
- **Notes:** Publisher grades exist (`/api/publishers/grades`) but there is no dedicated publisher performance dashboard page. Data is only visible within the pipeline list expandable detail.

### Missing: Buyer Performance Dashboard (standalone)
- **Status:** MISSING
- **Notes:** Buyer health scores exist (`/api/buyers/health`) but there is no dedicated buyer dashboard. Data only visible in pipeline list.

### Missing: Financial Reporting / P&L
- **Status:** MISSING
- **Notes:** No profit & loss reports, margin analysis by campaign/vertical, or financial dashboards. Data exists in call_records and invoices but no consolidated view.

### Missing: Notification Push (email/SMS/push)
- **Status:** MISSING
- **Notes:** Notification count and activity endpoints exist (`/api/notifications/count`, `/api/notifications/activity`, `/api/notifications/draft`) but there is no push notification delivery (email alerts, SMS, browser push). Notifications are in-app only.

---

## XVII. SUMMARY COUNTS

| Category | Count |
|---|---|
| Pages (routes) | 28 |
| API Routes | 130+ |
| Components | 50+ |
| @milo/* packages | 8 |
| Cron jobs | 12 |
| Supabase tables referenced | 25+ |
| External integrations | 5 (TrackDrive, ConvoQC, Anthropic, Resend, PPCRM) |
| Features WIRED | ~55 |
| Features PARTIAL | ~3 |
| Features MISSING | ~7 |

---

## XVIII. NOTABLE ARCHITECTURAL OBSERVATIONS

1. **EntityDetailPanel is 4,406 lines** -- the single largest component. It is the universal detail tray opened from pipeline-board, pipeline, and operator views. Stage-gated section visibility keeps it manageable UX-wise.

2. **Dual pipeline pages**: `/pipeline` (table+board, richer data) and `/pipeline-board` (pure Kanban, simpler). Both functional. The pipeline page is the more complete one with stats enrichment.

3. **No build step for vendor packages** -- @milo/* packages are vendored via `file:./vendor/*` and copied by a postinstall script. This is the "no monorepo" approach from Thesis 2.5.

4. **All LLM calls go through @milo/ai-client** -- unified tier system (classify = Haiku, structured = Sonnet). No direct Anthropic SDK calls outside the ai-client wrapper.

5. **Homepage (/) is a public demo funnel**, not a dashboard. Authenticated users land on `/operator` (via middleware redirect).

---

Report at: https://raw.githubusercontent.com/moscaleflow/conductor-mark-public/main/reports/2026-04-26-feature-inventory.md
