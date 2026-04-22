---
directive: "Operator-assisted lead capture feature audit"
lane: research
coder: Coder-3
started: 2026-04-20 ~00:30 MDT
completed: 2026-04-20 ~01:15 MDT
---

# Operator-Assisted Lead Capture Audit — 2026-04-20

## 1. Executive Summary

Operator-assisted lead capture exists in all four repos but with **radically different UX, data shapes, and enrichment depth**. There IS a common 6-step pipeline (ingest → normalize → dedup → screen → enrich → route) but the vertical coupling at each step is heavy enough that a `@milo/lead-capture` primitive would be thin infrastructure — useful but not transformative.

**Key findings:**

- **mysecretary** has the cleanest capture flow: paste/CSV → validate → insert → auto-draft. Person-centric (HOA property managers). Post-capture enrichment via Claude web_search with operator review gate.
- **PPCRM** has the most unique flow: Teams-chat-driven capture via SalesBot. Operator describes prospect in chat, AI extracts entity, "Add to Pipeline" button creates lead. Company-centric (PPC partners).
- **milo-ops** has the richest flow: CSV drag-drop with blacklist screening, conversation capture (paste/screenshot) with scam detection, LinkedIn research with operator-pasted profile text. Dual-table model (contacts + prospect_pipeline).
- **milo-outreach** has no operator UI — API-only lead creation. Designed as the engine that other verticals push leads into.

**Common horizontal pattern:** CSV parsing, field normalization (phone/email/name), dedup matching, blacklist screening integration. Worth extracting as utilities, not as a full primitive.

**Recommendation:** Do NOT extract `@milo/lead-capture` as a standalone primitive. Instead, add lead ingestion utilities to `@milo/crm` (normalize, dedup, import helpers) and keep capture UX vertical-specific.

## 2. Per-Repo Capture Flow Descriptions

### 2.1 mysecretary — Paste/CSV + AI Research

**Capture modes:** 2 (manual paste/upload, automated research)

#### Mode A: Manual Paste/Upload

| Aspect | Detail |
|--------|--------|
| **UI entry point** | "Add leads" button → AddLeadsPanel modal |
| **UX** | Paste CSV/TSV/Markdown table OR upload CSV file |
| **Format detection** | Auto-detects CSV, TSV, Markdown pipe tables |
| **Preview** | 5-row sample with validation issues highlighted |
| **Confirmation** | "Confirm — add N leads to queue" button |

**Key files:**

| File | LOC | Purpose |
|------|-----|---------|
| `components/admin/AddLeadsPanel.tsx` | 60 | Modal trigger, PasteImport mount |
| `components/admin/PasteImport.tsx` | 509 | Format detection, preview, validation |
| `components/admin/CsvImport.tsx` | 262 | File upload (fallback path at /admin/import) |
| `app/api/admin/leads/bulk/route.ts` | 176 | POST handler, row validation, Supabase insert |
| `app/api/admin/leads/route.ts` | 102 | Single-lead POST (fallback) |

**Operator provides:** tier, first_name, last_name, organization, hook, source_url (required). Optional: title, state, city, website, email, linkedin_url, credentials, firm_size_estimate, channel_preference.

**Auto-enriched at capture:** Nothing. Auto-enrichment is a separate post-capture flow.

**Where lead lands:** `leads` table, status='new'. Draft generation auto-triggers (5 concurrent workers, 3 positions x 2 channels).

#### Mode B: Automated Research + Manual Review

| Aspect | Detail |
|--------|--------|
| **Trigger** | Weekly cron, low-water-mark auto-trigger (queue < 5), or operator "Run research" button |
| **Mechanism** | Claude web_search, 12-week state directory rotation (CAI chapters + FCAP) |
| **Quality bar** | Rejects mega-firms, leads without verifiable hook, leads without email/linkedin |
| **Output** | 15-20 leads per run, status='new' |

**Post-capture enrichment:**

| Step | File | LOC | Mechanism |
|------|------|-----|-----------|
| Enrichment script | `scripts/enrich-leads.ts` | 288 | CLI, Claude web_search for missing LinkedIn URLs |
| Enrichment review | `components/admin/EnrichmentReviewBoard.tsx` | 442 | Operator approve/reject/edit before apply |
| Staging table | `lead_enrichment_candidates` | — | Stores proposals until operator approval |

**Total capture LOC:** ~1,840 (capture UX + research + enrichment review)

#### Schema: `leads` table

Key fields: id, tier (tier1/2/3), first_name, last_name, organization, hook, source_url, email, linkedin_url, channel_preference (linkedin/email/both), status (new/researched/ready/sent/replied/demo/customer/dead), message_draft_opener/followup1/followup2 (JSONB), credentials, firm_size_estimate.

**Notable:** No explicit `source` or `capture_method` field. Origin inferred from context (operator paste vs research_runs audit table).

---

### 2.2 PPCRM — Teams Chat + Kanban Form

**Capture modes:** 3 (Sales Chat AI extraction, Kanban form, SalesBot commands)

#### Mode A: Sales Chat "Add to Pipeline"

| Aspect | Detail |
|--------|--------|
| **UI entry point** | Sales Chat workspace in dashboard |
| **UX** | Operator describes prospect in chat → AI extracts company names → dedup/blacklist check → "Add to Pipeline" button |
| **Entity extraction** | 3-layer dedup: full name ilike, no-spaces join, CamelCase split |
| **Blacklist check** | Queries MOP master blacklist |
| **Auto-enrichment** | Async: role/vertical detection + web research for website/LinkedIn |

**Key files:**

| File | LOC | Purpose |
|------|-----|---------|
| `dashboard/workspaces/saleschat.js` | 1,225 | Sales Chat workspace (entity extraction, vetting, pipeline add) |
| `supabase/functions/salesbot/index.ts` | 1,657 | SalesBot agent (vet, signal, blacklist-add) |
| `supabase/functions/teams-bot/index.ts` | 1,083 | Teams Bot handler (routes to SalesBot) |

**Operator provides:** Company name (extracted by AI from chat text), optional contact_name.

**Auto-enriched:** source='sales-chat', role (publisher/buyer), verticals, website, LinkedIn URL (all via async AI research).

**Where lead lands:** `leads` table, outreach_status='new', research_status='pending' → 'complete'. Auto-assigned to Cam/Vee.

**Payload shape:**
```json
{
  "company_name": "Acme Media",
  "source": "sales-chat",
  "outreach_status": "new",
  "research_status": "pending",
  "metadata": {
    "source_context": "sales-chat",
    "source": "sales-intelligence-hub",
    "role": "publisher",
    "verticals": []
  }
}
```

#### Mode B: Kanban "Add New Lead" Form

| Aspect | Detail |
|--------|--------|
| **UI entry point** | "Add New Lead" button on Lead Kanban board |
| **Fields** | Company Name (required), Type (Publisher/Buyer), Vertical dropdown, Contact Name, Email, Assign To (Cam/Vee/Unassigned), Notes |
| **Auto-enriched** | Nothing |
| **Where lands** | `leads` table, outreach_status='new' |

**Key file:** `dashboard/index.html` (lines 7885-7907 handler, 8776-8850 form)

#### Mode C: SalesBot Commands (Teams)

Operators invoke via Teams chat:
- `vet [company]` → blacklist check, scam signal assessment
- `signal [company]` → log scam intelligence
- `blacklist-add` → add to master blacklist

Not direct lead creation, but part of the capture workflow (vetting before adding).

#### Auto-management:

- **Auto-reassign:** Cam → Vee after 3 days inactivity (daily cron at 7:00 AM Denver)
- **Activity log:** `lead_activity_log` tracks reassignments, status changes

#### Schema: `leads` table

Key fields: id, company_name, contact_name, contact_email, contact_phone, linkedin_url, website, source (manual/sales-chat/linkedin_scrape/referral), outreach_status (new/contacted/replied/qualified/rejected/converted/blacklisted), assigned_to (Cam/Vee/Unassigned), verticals[], role, metadata (JSONB), lead_score.

**Notable:** Company-centric (company_name is primary, contact_name secondary). Split from mysecretary's person-centric model.

**Total capture LOC:** ~3,000+ (Sales Chat + Kanban + SalesBot + Teams Bot)

---

### 2.3 milo-ops — CSV + Conversation Capture + Manual Form + LinkedIn Research

**Capture modes:** 4 (CSV import, conversation capture, manual prospect form, LinkedIn research)

#### Mode A: CSV Bulk Import

| Aspect | Detail |
|--------|--------|
| **UI entry point** | Drag CSV anywhere on dashboard (UniversalDropZone) |
| **Detection** | Auto-detects CSV with "name" + ("email" OR "company") headers |
| **Processing** | Column auto-detection, titleCase normalization, phone +1 format, blacklist screening per lead, dedup against contacts + prospect_pipeline |
| **Results** | 3-tab modal: NEW (green), EXISTING (orange), BLACKLISTED (red) |

**Key files:**

| File | LOC | Purpose |
|------|-----|---------|
| `src/app/api/leads/import/route.ts` | 459 | CSV processing, blacklist screen, dedup, insert |
| `src/components/LeadImportModal.tsx` | 430 | Results display (3 tabs) |
| `src/components/UniversalDropZone.tsx` | 628 | Global drag-drop detection |

**Operator provides:** CSV with any combination of: name, email, phone, company, job title, LinkedIn, platform/source, contact method, profile notes, entity type.

**Auto-enriched:** Blacklist screening result, pipeline stage assignment, MOP sync (fire-and-forget).

**Where leads land:** `contacts` table (contact record) + `prospect_pipeline` table (pipeline entry, stage='outreach'). Sponsors skip pipeline.

#### Mode B: Conversation Capture

| Aspect | Detail |
|--------|--------|
| **UI entry point** | CommandCenter — toggle between Text Paste and Screenshot |
| **Text paste** | Operator pastes raw conversation from Teams/Telegram/PPC Chat/Email |
| **Screenshot** | Drag-drop image, Claude Vision OCR + smart relevance filter |
| **AI extraction** | Participants, companies, action items, decisions, numbers, sentiment, urgency, topics |
| **Scam screening** | 12+ red flag patterns (impersonation, payment refusal, pressure tactics, below-market rates, sensitive verticals without compliance mention) |
| **Risk levels** | HIGH (blacklist hit, 2+ high flags) → MEDIUM (1 high flag) → LOW → CLEAN |
| **Auto-actions** | HIGH RISK → "Blacklist?" task to Mark. MEDIUM → "Investigate" task to Mark. CLEAN+positive → "Reach out?" task to Cam. NEW ENTITY → "Research" task to Cam. |

**Key file:** `src/app/api/capture/route.ts` (865 LOC)

**Where lands:** `conversation_captures` table (raw capture). Auto-creates contacts for unmapped companies (stage='qualifying'). Enriches existing contacts with new phone/email.

#### Mode C: Manual Prospect Add

| Aspect | Detail |
|--------|--------|
| **UI entry point** | "Add Prospect" on pipeline page |
| **Fields** | Contact name (req), Business name (req), Email (req), Country, Path mode (Vetting/Trusted) |
| **Trusted path** | Skip vetting, direct to onboarding (requires skip reason) |
| **Optional terms** | Billing type, payout rate, verticals, call types |

**Key file:** `src/components/AddProspectModal.tsx` (240+ LOC)

#### Mode D: LinkedIn Research

| Aspect | Detail |
|--------|--------|
| **UX** | Operator manually copies LinkedIn profile text (headline, about, experience) and pastes into research tool |
| **Hard contract** | No scraping — only operator-pasted text (enforced at code level) |
| **AI classification** | BUYER/PUBLISHER/BROKER, company size, decision authority, likely verticals, green/red flags, outreach angle |
| **Blacklist screen** | Parallel with LLM analysis |
| **Confidence gate** | 0.85+ → auto-research. 0.6-0.84 → research + flag. <0.6 → return matches for human selection |

**Key file:** `src/lib/tools/research-prospect.ts` (498 LOC)

#### Schema: `contacts` + `prospect_pipeline`

**contacts:** id, entity_name, contact_name, contact_email, phone, role, preferred_channel, linkedin_url, personal_notes (JSONB), last_contact_at.

**prospect_pipeline:** id, entity_name, stage (outreach/qualifying/onboarding/active/dormant), entity_type (publisher/buyer), billing_type, terms.

**Notable:** Dual-table model unlike single-table in other repos.

**Total capture LOC:** ~3,120 (CSV import + capture + form + LinkedIn research, API routes only)

---

### 2.4 milo-outreach — API Only, No Operator UI

**Capture modes:** 1 (JSON API)

| Aspect | Detail |
|--------|--------|
| **Entry point** | `POST /api/admin/leads` (JSON body) |
| **Auth** | API key (tenant-scoped) or HMAC cookie |
| **Fields** | first_name, last_name, organization, hook (required), tier (required). Optional: email, linkedin_url, title, state, city, website, source_url, channel_preference, notes |
| **Auto-enriched** | Nothing at capture |
| **Where lands** | `leads` table with tenant_id, status='new' |
| **UI** | "Admin UI coming soon" placeholder |

**Key files:**

| File | LOC | Purpose |
|------|-----|---------|
| `app/api/admin/leads/route.ts` | 134 | POST create, GET list |
| `app/api/admin/research/trigger/route.ts` | 46 | Manual research trigger |
| `lib/research-agent.ts` | ~400 | Claude web_search lead discovery |

**No CSV import, no bulk endpoint, no UI forms, no paste flow.** External verticals push leads in via API. Migration script exists for one-shot transfer from mysecretary.

**Schema:** Mirrors mysecretary `leads` table but adds `tenant_id` (TEXT) for multi-tenant isolation.

**Total capture LOC:** ~580 (API + research agent)

## 3. Data Shape Comparison

### 3.1 Required Fields at Capture

| Field | mysecretary | PPCRM | milo-ops | milo-outreach |
|-------|-------------|-------|----------|---------------|
| **Person name** | first_name + last_name | contact_name (optional) | contact_name | first_name + last_name |
| **Company** | organization | company_name (required) | business_name (required) | organization |
| **Email** | optional | optional | required (form), optional (CSV) | optional |
| **Hook/reason** | hook (required) | notes (optional) | — | hook (required) |
| **Source URL** | source_url (required) | — | — | source_url (optional) |
| **Tier/priority** | tier (tier1/2/3, required) | — | — | tier (tier1/2/3, required) |
| **Entity type** | — | type (Publisher/Buyer) | entity_type (publisher/buyer/sponsor) | — |
| **Vertical** | — | vertical (dropdown) | verticals (comma-sep) | — |

**Observation:** mysecretary and milo-outreach are person-centric (name + hook required). PPCRM and milo-ops are company-centric (company_name/business_name required). This reflects the underlying business models — mysecretary targets individual property managers, while PPCRM/milo-ops target companies.

### 3.2 Auto-Enrichment Comparison

| Enrichment | mysecretary | PPCRM | milo-ops | milo-outreach |
|------------|-------------|-------|----------|---------------|
| **At capture** | None | Role/vertical + web research (async) | Blacklist screen + pipeline stage | None |
| **Post-capture** | Claude web_search for LinkedIn URL | — | Entity verification + web research (confidence-gated) | — |
| **Operator review gate** | Yes (EnrichmentReviewBoard) | No (auto-applied) | Partial (confidence < 0.6 needs human) | No |
| **Scam/risk screening** | No | Blacklist check | 12+ red flag patterns + blacklist | No |

### 3.3 Lead Status Pipeline

| Status | mysecretary | PPCRM | milo-ops | milo-outreach |
|--------|-------------|-------|----------|---------------|
| new | new | new | outreach | new |
| researched | researched | — | qualifying | researched |
| ready | ready | — | — | ready |
| contacted | — | contacted | — | — |
| sent | sent | — | — | sent |
| replied | replied | replied | — | replied |
| qualified | — | qualified | onboarding | — |
| active | — | — | active | — |
| converted | — | converted | — | — |
| demo | demo | — | — | demo |
| customer | customer | — | — | customer |
| rejected | — | rejected | — | — |
| dead | dead | — | dormant | dead |
| blacklisted | — | blacklisted | — | — |

**Observation:** No two repos share the same status set. mysecretary/milo-outreach are closest (outreach-funnel model). PPCRM has sales-pipeline stages. milo-ops uses prospect_pipeline stages.

### 3.4 Destination Tables

| Repo | Lead Table | Pipeline Table | Contact Table |
|------|-----------|----------------|---------------|
| mysecretary | `leads` | — | — |
| PPCRM | `leads` | — | — |
| milo-ops | — | `prospect_pipeline` | `contacts` |
| milo-outreach | `leads` | — | — |

## 4. Horizontal vs Vertical Split Analysis

### 4.1 Horizontal (shared across 2+ repos)

| Component | Where Found | Extractable LOC | Notes |
|-----------|-------------|-----------------|-------|
| **CSV parsing + column auto-detection** | mysecretary (PasteImport), milo-ops (leads/import) | ~200 | Both detect name/email/company/phone columns, normalize. milo-ops is more sophisticated (12+ column types). |
| **Field normalization** | milo-ops (titleCase, phone +1, email lowercase) | ~50 | mysecretary does basic trim. milo-ops does full normalization. |
| **Dedup matching** | PPCRM (3-layer: ilike, no-spaces, CamelCase), milo-ops (email + company match) | ~80 | Different strategies but same problem. |
| **Blacklist screening integration** | PPCRM, milo-ops | ~30 | Both call @milo/blacklist (or equivalent). Already extracted. |
| **AI entity extraction from text** | PPCRM (Sales Chat), milo-ops (conversation capture) | ~150 | Both use Claude to extract companies/people from unstructured text. Prompts are domain-specific. |
| **Web research enrichment** | mysecretary (enrich-leads.ts), milo-ops (web-research.ts) | ~200 | Both use Claude web_search. Search strategies differ (LinkedIn lookup vs entity verification). |
| **Lead status machine** | All 4 repos | ~40 | Different statuses, same pattern (linear pipeline with terminal states). |

### 4.2 Vertical (repo-specific)

| Component | Repo | Why Vertical |
|-----------|------|-------------|
| **HOA directory rotation** | mysecretary | CAI chapters, FCAP, state-by-state rotation — specific to HOA property management |
| **Tier system (tier1/2/3)** | mysecretary, milo-outreach | Specific to outreach prioritization model |
| **Hook field** | mysecretary, milo-outreach | "Reason to reach out" — specific to cold outreach |
| **Teams/SalesBot integration** | PPCRM | Teams bot, Adaptive Cards, conversation references — specific to PPCRM's Teams-first workflow |
| **VA assignment (Cam/Vee)** | PPCRM | Named VAs with auto-reassignment — specific to PPCRM's ops model |
| **PPC vertical detection** | PPCRM, milo-ops | SSDI, Medicare, Auto Insurance, etc. — performance marketing verticals |
| **Scam screening red flags** | milo-ops | 12+ patterns specific to PPC industry (below-market rates, exclusive traffic claims, pressure tactics) |
| **Conversation capture + OCR** | milo-ops | CommandCenter UX, screenshot relevance filter — specific to milo-ops monitoring workflow |
| **LinkedIn research contract** | milo-ops | "No scraping, operator-pasted only" — specific to milo-ops compliance posture |
| **3-message draft pipeline** | mysecretary | opener/followup1/followup2 — specific to mysecretary's outreach cadence |
| **Billing type terms** | milo-ops | CPA/RTB/CPL/CPQL — PPC-specific |

### 4.3 Assessment

**Horizontal surface is thin.** The shared patterns (CSV parsing, normalization, dedup) are utility functions, not a cohesive domain. The vertical logic (enrichment prompts, screening rules, pipeline stages, UX) dominates each implementation.

**There is no "lead capture lifecycle"** that's shared. Each repo has a fundamentally different capture philosophy:
- mysecretary: batch import + AI research (outreach-focused)
- PPCRM: chat-first AI extraction (intelligence-focused)
- milo-ops: multi-modal capture + risk assessment (ops-focused)
- milo-outreach: API receiver (engine-focused)

## 5. Recommendation

### Do NOT extract `@milo/lead-capture` as a standalone primitive.

**Why:**

1. **Too thin.** The horizontal surface is ~550 LOC of utility functions (CSV parsing, normalization, dedup). A package with 550 LOC of utilities and a README is not a meaningful primitive — it's a utils folder.

2. **Too coupled.** The valuable parts (enrichment, screening, entity extraction) require domain-specific prompts and rules. A "generic" lead capture primitive would either be too generic to be useful or would accumulate vertical-specific options until it becomes a config monster.

3. **milo-outreach already IS the engine.** milo-outreach's API-first design means verticals push leads into a shared outreach engine. The capture UX stays vertical; the engine is horizontal. This is the right architecture.

### Instead, do this:

| Action | Where | LOC | Rationale |
|--------|-------|-----|-----------|
| **Add CSV import helpers to `@milo/crm`** | `@milo/crm/import` | ~150 | Column auto-detection, format parsing (CSV/TSV/Markdown), field normalization. Used by any vertical's import UI. |
| **Add dedup utilities to `@milo/crm`** | `@milo/crm/dedup` | ~80 | Multi-strategy matching (exact, fuzzy, normalized). Already needed for CRM merge. |
| **Keep enrichment vertical** | Each repo | 0 | Enrichment prompts, research strategies, and confidence thresholds are domain-specific. Not worth abstracting. |
| **Keep capture UX vertical** | Each repo | 0 | Every repo has a different capture philosophy. Forcing a shared UX would make all of them worse. |
| **Standardize lead → crm_leads migration** | `@milo/crm` schema | ~50 | Ensure crm_leads table has `source` and `capture_method` fields so all verticals can track origin. |

### Priority: Low

This is not blocking any current work. The capture flows work today in each repo. The CSV/dedup utilities are nice-to-have for reducing duplication but not urgent.

## 6. Specific Question Answers

### Q: What does mysecretary's "Add leads" button actually do?

Opens AddLeadsPanel modal → PasteImport component → operator pastes CSV/TSV/Markdown → format auto-detected → 5-row preview with validation → "Confirm — add N leads" → POST /api/admin/leads/bulk → insert to `leads` table with status='new' → auto-triggers draft generation (5 concurrent workers, 3 positions x 2 channels). Fallback: CsvImport component at /admin/import for file upload.

### Q: How did PPCRM operators trigger Teams-chat-driven lead capture?

Operator opens Sales Chat workspace in PPCRM dashboard → types/pastes prospect info (company name, screenshot, etc.) → AI Claude extracts company names from text/images → 3-layer dedup check → blacklist status check → for companies "NOT IN SYSTEM", operator clicks "Add to Pipeline" button → creates lead with source='sales-chat', outreach_status='new' → async enrichment (role/vertical detection + web research). The 6 leads mapped from 'sales-chat' to 'inbound' during @milo/crm migration came from this flow.

### Q: Is there a "paste LinkedIn URL, operator fills rest, system researches" pattern?

**Partially in milo-ops only.** The LinkedIn research tool (`research-prospect.ts`) accepts a linkedin_profile_url as reference, but the operator must manually paste the profile text (headline, about, experience) — no scraping. The system then classifies the prospect (buyer/publisher/broker), assesses decision authority, detects verticals, checks for red flags, and routes to appropriate pipeline stage. No other repo has this pattern.

### Q: Does any capture flow integrate with enrichment?

Yes, all three non-API repos:

| Repo | Integration | Timing |
|------|-------------|--------|
| mysecretary | Claude web_search for LinkedIn URLs → staging table → operator review | Post-capture (separate CLI script + review board) |
| PPCRM | AI role/vertical detection + web research | At capture (async, non-blocking) |
| milo-ops | Entity verification + web research + LinkedIn analysis | At capture (confidence-gated) + post-capture |

## 7. Open Questions for Mark

1. **Should `crm_leads` schema include `source` and `capture_method` fields?** Currently mysecretary has no explicit source field. PPCRM has `source` (manual/sales-chat/linkedin_scrape/referral). milo-ops tracks via `conversation_captures` table. Standardizing in @milo/crm would enable cross-vertical capture analytics.

2. **Is the milo-ops conversation capture flow (paste/screenshot → AI entity extraction → scam screening → auto-action-items) intended to be shared with future verticals?** It's the most sophisticated capture system (~865 LOC) and could serve as a template, but the scam screening rules are PPC-specific.

3. **Should milo-outreach grow an operator UI for lead capture?** Currently API-only. If verticals are expected to push leads via API (the current design), no UI needed. But if operators need to add leads directly to milo-outreach (bypassing vertical apps), a UI would be needed.

4. **Are the CSV import helpers worth extracting now, or should this wait until a 3rd vertical needs them?** mysecretary and milo-ops both have column auto-detection, but milo-ops's is significantly more sophisticated. Extracting now means choosing one implementation; waiting means a cleaner merge later.

5. **What's the intended relationship between `leads` (mysecretary/PPCRM/milo-outreach) and `contacts` + `prospect_pipeline` (milo-ops)?** Are these converging into `crm_leads` from @milo/crm, or do contacts/pipeline stay separate from leads?
