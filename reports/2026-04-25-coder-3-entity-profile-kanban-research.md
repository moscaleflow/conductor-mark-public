# Entity Profile + Kanban + Draft Persistence Research — Existing Infra Mapped

**Coder-3 Research | 2026-04-25**
**Scope:** Pure audit. No code. Map existing infrastructure for vet-spawned entity profiles, kanban deal pipeline, and draft persistence across milo-for-ppc, milo-engine, milo-ops, milo-outreach.
**Cross-references:** D116 (Vet Part 1), D117 (CRM context), D106 (voice audit), /ask workflow audit (2026-04-24).

---

## 1. Entity Profile Data Model

### WHAT EXISTS

**Two parallel entity systems with NO cross-references:**

| System | Table | Primary key | Used by |
|---|---|---|---|
| **@milo/crm** | `crm_counterparties` | UUID `id`, tenant-scoped | /ask CRM context injection, operator pill, capture route |
| **Pipeline** | `prospect_pipeline` | UUID `id`, entity-name based | Pipeline pages, EntityDetailPanel, stage management, TD provisioning, contract flow |

**`crm_counterparties` fields (22 columns):**
`id`, `tenant_id`, `counterparty_type` (publisher/buyer/vendor/partner/client/prospect), `company`, `status` (lead/contacted/qualified/onboarding/active/paused/churned/blacklisted/archived), `contact_name`, `contact_email`, `contact_phone`, `contact_telegram`, `linkedin_url`, `website`, `address`, `city`, `state`, `zip`, `country`, `source` (manual/outreach/referral/inbound), `referred_by`, `assigned_to`, `notes`, `metadata` JSONB, `archived_at`, `created_at`, `updated_at`

**`prospect_pipeline` fields (not on crm_counterparties):**
`entity_name`, `entity_type`, `stage`, `billing_type`, `verticals`, `call_types`, `coverage_states`, `broker_status`, `risk_flags` (string[]), `td_traffic_source_id`, `td_buyer_id`, `stage_changed_at`, `contact_id` (FK to `contacts`), `source`, `intelligence_brief` JSONB

**`crm_counterparties` fields NOT on prospect_pipeline:**
`contact_name`, `contact_email`, `contact_phone`, `contact_telegram`, `address`, `city`, `state`, `zip`, `country`, `referred_by`, `assigned_to`, `archived_at`

**Entity detail UI:** No dedicated `/counterparties/[id]` or `/entities/[id]` route. Instead:
- `EntityDetailPanel` — slide-out tray component at `src/components/shared/EntityDetailPanel.tsx`. Fetches from `/api/entities/[name]` and aggregates: prospect_pipeline + contacts + call_records + campaign_routes + conversation_captures + blacklist screening + invoices + creative_submissions + entity verification.
- `/api/entities/[name]/route.ts` — entity intelligence API. Queries `prospect_pipeline` (NOT `crm_counterparties`).

**CRM context injection (`crm-context.ts`):** Only uses 6 of 22 crm_counterparties columns: `id`, `company`, `counterparty_type`, `status`, `created_at`, `notes`. The other 16 are fetched but ignored.

**Entity verification:** Stored in `contacts.personal_notes.entity_verification` JSONB — single latest-only record with `confidence`, `tier` (verified/likely/unknown), `unique_match`, `needs_human_review`. This is identity verification ("is this the right Acme LLC?"), NOT a vet verdict.

### WHAT'S MISSING

| Required field | Exists? | Where? | Gap |
|---|---|---|---|
| `verdict` (pass/fail/caution) | NO | — | Vet output is freeform prose in chat_logs.response_text |
| `flags` JSONB (red/green) | PARTIAL | `prospect_pipeline.risk_flags` (string[]) | Risk-only, no green flags, no structured split |
| `confidence_score` | PARTIAL | `contacts.personal_notes.entity_verification.confidence` | Identity confidence, NOT vet confidence |
| `source_screenshot` | NO | — | conversation_captures stores captures but no link to entity profile |
| `vet_history` (past vets) | NO | — | No vet log table exists anywhere |
| `last_vetted_at` | NO | — | Not tracked |
| `vetted_by` | NO | — | Not tracked |
| Cross-reference between the two entity tables | NO | — | No FK, no name-based join |

### RECOMMENDED PATH

**Extend, don't build new.** The two-entity-system problem is the elephant:

1. **Short-term:** Add a `vet_results` table with nullable FKs to BOTH `prospect_pipeline.id` AND `crm_counterparties.id`, plus `chat_logs.id` and `blacklist_entries.id`. Denormalize `entity_name` for querying. This avoids forcing a merge of the two entity systems while allowing vet results to link to whichever system the entity lives in.

2. **Medium-term:** The two entity systems need reconciliation. `prospect_pipeline` is where the UI lives; `crm_counterparties` is where the extracted @milo/crm package operates. A sync mechanism or merge is needed but is a separate directive.

**Effort:** S (vet_results table) / L (entity system reconciliation)

---

## 2. Kanban / Pipeline Infrastructure

### WHAT EXISTS

**This is NOT greenfield. A full production kanban board exists.**

| Layer | Location | Status |
|---|---|---|
| **Database table** | `prospect_pipeline` (Supabase) | Production |
| **8 pipeline stages** | outreach, qualifying, drip, onboarding, activation, active, dormant, blacklisted | Hardcoded in API + UI |
| **Full kanban board** | `/pipeline-board` (milo-for-ppc, 1251 lines) | Production, `@hello-pangea/dnd` |
| **Reusable KanbanBlock** | `src/components/shared/KanbanBlock.tsx` (492 lines) | Production, native HTML5 DnD |
| **Pipeline list + board toggle** | `/pipeline` (milo-for-ppc, 1448 lines) | Production |
| **Stage change API** | `PATCH /api/pipeline/[id]` | Production |
| **Gated stage change API** | `POST /api/pipeline/[id]/stage` | Production, with activation gate (requires signed MSA + tax doc + IO) |
| **Audit logging** | `ai_action_log` table | Production |
| **DnD library** | `@hello-pangea/dnd ^18.0.1` | Both milo-for-ppc and milo-ops |
| **Billing pipeline** | `BillingPipelineBlock` (milo-ops) | Production, separate 5-column board |

**Pipeline board columns (milo-for-ppc `/pipeline-board`):**

| Column | Stages | Color |
|---|---|---|
| Outreach | `['outreach']` | Gray |
| Qualifying | `['qualifying', 'drip']` | Blue |
| Onboarding | `['onboarding']` | Purple |
| Activation | `['activation']` | Amber |
| Active | `['active']` | Green |

Dormant + blacklisted shown as toggleable count badges below the board.

**KanbanBlock (reusable component) columns:**

4 columns (no Active — graduated entities shown as summary count): Outreach / Qualifying (includes drip) / Onboarding / Activation

**milo-ops KanbanBlock — diverged:** Has extra stages `trusted_shortcut` and `inbound_onboarding` in the onboarding column that milo-for-ppc does not. Column definitions are hardcoded in 3 separate files with no shared config.

**Valid stages enum (API):**
```
outreach | qualifying | drip | onboarding | activation | active | dormant | blacklisted
```

### State machines in milo-engine

**@milo/onboarding:**
- Step-based DAG state machine: `pending → active → completed/failed/skipped`
- Run statuses: `active → completed/cancelled/expired`
- 6 step integrations: blacklist_screen, contract_signing, crm_update, file_upload, webhook, custom
- Event bus: `run.started | run.completed | step.completed | step.stalled` etc.
- **Verdict:** Could drive auto-advance from onboarding → activation but is NOT itself a deal pipeline. The pipeline board's `stage` column is the pipeline — onboarding tracks granular steps within that stage.

**@milo/contract-negotiation:**
- `draft → broker_review ↔ counterparty_review → finalized/cancelled`
- Back-and-forth negotiation loop. Not a deal pipeline, but negotiation status could be metadata on a pipeline card.

**@milo/crm CounterpartyStatus enum:**
```
lead | contacted | qualified | onboarding | active | paused | churned | blacklisted | archived
```
This is a superset that almost maps to pipeline stages but with different naming and no sync mechanism. Changing one does NOT update the other.

### WHAT'S MISSING

| Gap | Detail |
|---|---|
| **No "vetted" stage** | Current pipeline starts at outreach. A vet-spawned entity has no natural entry point — it would need a pre-outreach stage or a "vetted" column. |
| **No pipeline type differentiation** | Only one pipeline exists. If vet results need a separate deal flow, the table needs a `pipeline_type` column or new table. |
| **No onboarding → pipeline auto-advance** | @milo/onboarding events don't trigger pipeline stage changes. |
| **No CRM status ↔ pipeline stage sync** | Two divergent enums, no bridge. |
| **Column config not centralized** | Hardcoded in 3 files with drift already occurring. |

### RECOMMENDED PATH

**Extend the existing pipeline board.** Do NOT build a new kanban:

1. **Add a "Vetted" column** (or "New" pre-outreach column) to the pipeline board. A vet result that identifies a new entity creates a `prospect_pipeline` row at stage `vetted`. The operator drags it to `outreach` when ready.
2. **Add `VALID_STAGES` entry** for `vetted` in the stage change API.
3. **Column config extraction** — move column definitions to a shared config (closes the 3-file drift). This is a quick win.
4. **Event bridge** (later) — @milo/onboarding `run.completed` → auto-advance pipeline stage.

**Effort:** S (add vetted column + stage) / M (column config extraction) / L (event bridge)

---

## 3. Draft Outreach Persistence

### WHAT EXISTS

**Four separate draft systems, none connected to /ask:**

| System | Location | Storage | Connected to /ask? |
|---|---|---|---|
| **chat_logs.response_text** | milo-for-ppc /ask route | `chat_logs` table, capped at 10K chars | YES — but write-only, no UI to browse, no entity FK, IP-keyed not user-keyed |
| **ai_action_log** | milo-for-ppc operator tools | `ai_action_log` table with `data_snapshot` JSONB | NO — only for tool-invoked drafts (`draft_first_outreach`), not /ask |
| **milo-outreach leads.message_draft_*** | milo-outreach | 3 JSONB columns on `leads` table: `message_draft_opener`, `message_draft_followup1`, `message_draft_followup2` | NO — purpose-built for lead sequences |
| **disputes.evidence.draft_text** | milo-for-ppc disputes | `disputes` table with `evidence` JSONB containing `is_draft`, `draft_text` | NO — dispute-specific |

**chat_logs schema (full, assembled from 6 migrations):**
`id` UUID, `created_at`, `pill` TEXT, `input_text` TEXT, `attachments` JSONB, `response_text` TEXT, `model_used` TEXT, `input_tokens` INT, `output_tokens` INT, `cache_read_tokens` INT, `cache_creation_tokens` INT, `latency_ms` INT, `ip_hash` TEXT, `error` TEXT, `user_rating` TEXT, `rating_at` TIMESTAMPTZ, `admin_flag` BOOLEAN, `batch_id` TEXT, `persona` TEXT, `query_id` TEXT, `expected_pill` TEXT, `run_mode` TEXT, `context_metadata` JSONB, `analysis_id` TEXT, `classifier_decision` JSONB, `operator_override` BOOLEAN, `final_destination` TEXT

**@milo/crm crm_activities:**
- Actions: `status_change | note | contact_added | promoted | assigned | created | updated | archived`
- NO draft-related action. No `outreach_drafted`, `message_drafted`, `email_sent`.
- Activity shape: `{ id, tenant_id, counterparty_id, lead_id, action, detail, previous_value, new_value, performed_by, created_at }`

**milo-outreach draft lifecycle:**
1. `draft-generator.ts` calls Claude with voice-config template → writes to `leads.message_draft_<position>` JSONB
2. `draft-fanout.ts` fans out leads × positions × channels into generation jobs
3. Drafts sit in lead row, reviewable
4. `email-sender.ts` reads draft, 4 gate checks (blacklist, env, tenant, manual-confirmation), sends via Resend
5. Logged to `send_log` table with status: sent/delivered/bounced/failed/dry_run/blocked_*

**milo-for-ppc /api/outreach/draft/route.ts:** Generates outreach using real performance data. Returns JSON `{subject, body, tone, platform, data_used}`. Does NOT persist — response only.

### WHAT'S MISSING

| Gap | Detail |
|---|---|
| **No saved_drafts or outreach_drafts table** | Sales pill drafts exist only in chat_logs.response_text (unstructured, no entity FK) and React state |
| **No entity linkage on chat_logs** | Cannot query "show me all drafts for MediaBridge" without NLP extraction from input_text |
| **No user_id on chat_logs** | Drafts cannot be attributed to a specific operator (IP-keyed only, no auth) |
| **No "act on this later" workflow** | chat_logs is an audit log, not an action queue |
| **No bridge between /ask Sales pill and milo-outreach send pipeline** | Zero shared persistence |
| **No UI to browse past drafts** | chat_logs is write-only from the operator's perspective |

### RECOMMENDED PATH

**Extend @milo/crm activities for lightweight draft logging, build a proper drafts table for actionable drafts:**

1. **Immediate (S):** Add `outreach_drafted` to `ACTIVITY_ACTIONS` enum in @milo/crm. When Sales pill produces a draft in /ask, log an activity against the matched `counterparty_id` (if CRM context found one). This gives draft history on the entity timeline with zero new tables.

2. **Short-term (M):** Add an `ask_drafts` table: `{ id, chat_log_id FK, counterparty_id FK nullable, lead_id FK nullable, entity_name TEXT, pill TEXT, draft_text TEXT, draft_type (outreach/dispute/followup), status (draft/sent/discarded), created_by TEXT, created_at, acted_at }`. The /ask route writes here when pill is `sales` and response contains an outreach draft. The pipeline board's EntityDetailPanel can show recent drafts.

3. **Later (L):** Bridge to milo-outreach send pipeline — convert an ask_draft into a milo-outreach `leads.message_draft_opener` for sequenced delivery.

**Effort:** S (activity enum) / M (ask_drafts table) / L (milo-outreach bridge)

---

## 4. Vet Result Persistence

### WHAT EXISTS

**No structured vet result storage anywhere.** Vet output is freeform prose in `chat_logs.response_text`.

| What exists | Where | Usable as vet history? |
|---|---|---|
| `chat_logs` filtered by `pill='vet'` | milo-for-ppc Supabase | PARTIALLY — has `response_text` but verdict/entity/flags are unstructured prose. No entity FK. 90-day archive rotation splits history across tables. |
| `contacts.personal_notes.entity_verification` | milo-for-ppc foundation | NO — this is identity verification (confidence/tier/matches), not vet verdict |
| `prospect_pipeline.risk_flags` (string[]) | milo-for-ppc foundation | PARTIALLY — stores risk flag strings but no green flags, no verdict, no history |
| `blacklist_entries` | @milo/blacklist | NO — negative-only. Cannot represent "vetted clean" or "caution". |
| `analysis_records` (contract analysis) | @milo/contract-analysis | NO — document-centric, not entity-centric. But good structural template. |

**Vet prompt output format (from vet.ts):**
- Entity name + type
- RED FLAGS (bulleted)
- GREEN FLAGS (bulleted)
- CONFIDENCE RATING: X out of 5
- Closing verdict (3-4 lines for contract, full paragraph for entity)
- Suggested blacklist note if negative

This is all unstructured markdown inside `response_text`. Nothing is parsed into structured fields at write time.

### WHAT'S MISSING

| Gap | Detail |
|---|---|
| **No vet_results table** | No structured storage for verdict, flags, confidence, entity link |
| **No entity FK on chat_logs** | Cannot query "all vets for entity X" without text search |
| **No parsed vet output** | Verdict/flags/confidence buried in freeform prose |
| **No vet history** | Only latest entity_verification (identity, not vet) stored per contact |
| **No vet → pipeline bridge** | Vet result doesn't create or update a prospect_pipeline row |
| **No vet → blacklist bridge** | "Add to blacklist" from vet output requires manual operator action |

### RECOMMENDED PATH

**New `vet_results` table + structured output parsing at write time:**

1. **New table `vet_results`:**

```
id UUID PK
tenant_id TEXT
entity_name TEXT (denormalized for querying)
entity_type TEXT (publisher/buyer/vendor/etc.)
verdict TEXT (clean/caution/high_risk/blocked/inconclusive)
confidence_rating INT (1-5, matches Milo output format)
red_flags JSONB (structured array)
green_flags JSONB (structured array)
blacklist_hit BOOLEAN
blacklist_entry_id UUID FK nullable → blacklist_entries
prospect_pipeline_id UUID FK nullable → prospect_pipeline
counterparty_id UUID FK nullable → crm_counterparties
chat_log_id UUID FK nullable → chat_logs
analysis_id UUID FK nullable → analysis_records (if contract was part of vet)
vet_source TEXT (milo_ask/scam_chat/manual/web_research)
response_summary TEXT (extracted verdict prose, ~50-100 words)
vetted_by TEXT
created_at TIMESTAMPTZ
```

2. **Structured output parsing:** After the vet pill streams its response, parse the freeform output into structured fields. Two options:
   - **Option A (cheap):** Regex extraction — RED FLAGS section → `red_flags` JSONB, CONFIDENCE RATING line → `confidence_rating` INT, etc. Fragile but fast.
   - **Option B (robust):** Add a structured JSON output step after the vet streams — a second lightweight LLM call (Haiku) that takes the vet response and returns `{ verdict, confidence, red_flags[], green_flags[], summary }`. More reliable, adds ~500ms latency.

3. **Vet → pipeline bridge:** When a vet_result is created for an entity NOT already in prospect_pipeline, offer "Add to pipeline?" action. Creates a prospect_pipeline row at stage `vetted` (new stage from Section 2).

4. **Vet → blacklist bridge:** When verdict is `blocked`, surface "Add to blacklist?" action button (already identified in D107 spec for scam-chat screenshots).

**Effort:** M (vet_results table + Option A parsing) / L (Option B parsing + pipeline/blacklist bridges)

---

## 5. Recommended Directive Sequence

| Priority | Directive | Effort | What it closes |
|---|---|---|---|
| **1** | **`vet_results` table + structured parsing** | M | Vet result persistence, entity-linked vet history, queryable verdicts |
| **2** | **Add "vetted" stage to pipeline board** | S | Vet → pipeline entry point, new kanban column |
| **3** | **Draft logging via crm_activities** | S | Draft history on entity timeline, `outreach_drafted` action type |
| **4** | **`ask_drafts` table for actionable drafts** | M | Browse/act-on past Sales pill drafts, entity-linked |
| **5** | **Entity system reconciliation** | L | Merge or sync crm_counterparties ↔ prospect_pipeline |
| **6** | **Pipeline column config extraction** | M | Close 3-file column drift (milo-for-ppc × 2 + milo-ops) |
| **7** | **Onboarding event → pipeline auto-advance** | L | @milo/onboarding run.completed → stage change |
| **8** | **milo-outreach send bridge** | L | Convert /ask drafts to sequenced delivery |

Directives 1-3 are the minimum for "vet results spawn an entity profile that flows into a kanban board." Directive 4 adds draft persistence. Directives 5-8 are infrastructure cleanup.

---

## Architecture Decision: The Two-Entity Problem

The biggest finding is the **two parallel entity systems**:

```
@milo/crm: crm_counterparties (extracted package, tenant-scoped, CRM lifecycle)
     ↕ NO LINK
Pipeline:  prospect_pipeline (TLP-specific, stage-based, call/campaign data)
```

Both represent the same real-world entities. Both have `entity_name`/`company` as their natural key. Neither references the other.

**Impact on vet-spawned profiles:**
- If a vet result creates a `prospect_pipeline` row, the /ask CRM context won't see it (reads `crm_counterparties`)
- If a vet result creates a `crm_counterparties` row, the pipeline board won't see it (reads `prospect_pipeline`)
- The `vet_results` table must FK to BOTH and the UI must decide which system to show

**Recommended interim approach:** `vet_results` links to both via nullable FKs. When a vet result spawns a new entity:
1. Create `prospect_pipeline` row at stage `vetted` (powers the kanban board)
2. Create `crm_counterparties` row at status `lead` (powers /ask CRM context)
3. Store both IDs on the vet_result row
4. Accept the duplication until the entity reconciliation directive (Priority 5) ships

This is pragmatic duplication — worse than a unified entity model, but shippable without a multi-week migration.

---

Cross-references: D116 (Vet Part 1), D117 (CRM context), D106 (voice consistency), D107 (Surface 18 spec), D112 (Surface 18 locked decisions), /ask workflow audit (2026-04-24), @milo/crm types.ts, @milo/onboarding state-machine.ts, @milo/contract-negotiation state-machine.ts, @milo/blacklist types.ts, @milo/contract-analysis types.ts.
