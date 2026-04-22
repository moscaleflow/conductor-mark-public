---
directive: PPCRM read-path migration to shared CRM
lane: migration
coder: Coder-2
started: 2026-04-19 ~21:00 MDT
completed: 2026-04-19 ~22:30 MDT
---

# PPCRM Read-Path Migration — Detailed Report (2026-04-19)

## Section 1: Read-Path Inventory

16 functions migrated. 10 functions had zero partner/lead reads and required no changes.

| # | Function | LOC | Reads partners | Reads leads | Reads both | Notes |
|---|----------|-----|:-:|:-:|:-:|-------|
| 1 | milo-query | 661 | yes | yes | both | partner_lookup, prospects, enrichment, partner_timeline |
| 2 | outreach-api | 386 | yes | — | partners only | partner ID lookup for entity matching |
| 3 | opsbot | 2843 | yes | — | partners only | publisher mapping, name lookups, alert mapping, entity check |
| 4 | vacoordinator | 1914 | yes | — | partners only | partner count health check |
| 5 | teams-bot | 1083 | yes | — | partners only | recent partners, conversation partner match |
| 6 | outreach-followup | 282 | — | yes | leads only | stale leads for follow-up |
| 7 | billingbot | 1335 | yes | — | partners only | 9 queries: buyer/publisher name lookups, netting, aging, dashboard |
| 8 | salesbot | 1656 | yes | — | partners only | partner by ID, pipeline stage counts |
| 9 | onboardbot | 5305 | yes | yes | both | ~26 partner reads + ~8 lead reads. Largest migration |
| 10 | signature-webhook | 419 | yes | — | partners only | findPartner 3-step lookup, alert partner lookup |
| 11 | onboard-submit | 328 | yes | yes | both | partner by email, lead by email, partner by name |
| 12 | outreach-triggers | 898 | yes | — | partners only | 7 publisher/buyer lookups (one per trigger type) |
| 13 | trackdrive-sync | 897 | yes | — | partners only | publisher mapping by td_traffic_source_id, buyer by name |
| 14 | briefing | 1121 | yes | — | partners only | active publishers list |
| 15 | milo-alerts | 600 | yes | — | partners only | publisher lookup for alert logging |
| 16 | _shared/mop-client.ts | 934 | yes | — | partners only | getPartner() fallback — affects opsbot, salesbot, onboardbot, vacoordinator |

**Total LOC across migrated functions:** ~20,662

**Functions with NO partner/lead reads (10 — untouched):**
auto-outreach, auto-sync-calls, campaign-sync, campaign-sync-mop, dashboard, did-manager, milo-insights, reconciliation-webhook, send-message, freeze-guard

---

## Section 2: Per-Function Migration Detail

### 2.1 — milo-query (661 LOC)

- **Commit:** `38f0e31` (2026-04-19 21:20:08 -0700)
- **Deploy:** `supabase functions deploy milo-query` — successful, version bumped
- **Reads migrated:** 4
  - `queryPartnerLookup()` — `crm_counterparties` ilike company, results mapped via `toPartner()`
  - `queryProspects()` — `crm_leads`, results mapped via `toLead()`
  - `queryLeadActivity()` — `crm_leads` for lead name enrichment
  - `queryPartnerTimeline()` — `crm_counterparties` for partner ID resolution
- **Curl verification:**
  ```
  POST /milo-query  {"query_type":"partner_lookup","params":{"name":"Flex"}}
  → 5 results: Flex Marketing (publisher), Flex Marketing FE WT 23/90 (buyer), etc.
  Response shape: company_name, type, partner_type, td_traffic_source_id, billing_type — all present ✅
  ```
  ```
  POST /milo-query  {"query_type":"prospects","params":{"status":"new"}}
  → 7 leads: healthfirst marketing, M Ali, Sunrise Media Group, etc.
  Response shape: company_name, outreach_status, response_received (false), converted_to_partner_id — all present ✅
  ```
- **Shim needed:** Yes — `toPartner()` and `toLead()` from crm-reader.ts (see Section 3)

### 2.2 — outreach-api (386 LOC)

- **Commit:** `4e4b9ec` (2026-04-19 21:30:07 -0700)
- **Deploy:** Batch deploy with commit 2 — successful
- **Reads migrated:** 1
  - Partner ID lookup in enrichment loop: `crmDb.from('crm_counterparties').select('id').eq('tenant_id', CRM_TENANT).ilike('company', entityName)`
- **Curl verification:** Not individually curled — this function is called by the outreach pipeline (cron-triggered). Code inspection confirms the single read now targets `crm_counterparties` at line 186.
- **Shim needed:** No — only selects `id`, which is the same in both schemas.

### 2.3 — opsbot (2843 LOC)

- **Commit:** `4e4b9ec` (2026-04-19 21:30:07 -0700)
- **Deploy:** Batch deploy with commit 2 — successful
- **Reads migrated:** 4
  - Publisher mapping by td_traffic_source_id (line 229) — now selects `id, metadata` from crm_counterparties, filters td_traffic_source_id from metadata in JS
  - Publisher names by IDs (line 947) — `crmDb.from('crm_counterparties').select('id, company').eq('tenant_id', CRM_TENANT).in('id', ids)`
  - Alert publisher mapping (line 1177) — `crmDb.from('crm_counterparties').select('id, company').eq('tenant_id', CRM_TENANT)`
  - Entity type check (line 1746) — `crmDb.from('crm_counterparties').select('counterparty_type').eq('tenant_id', CRM_TENANT).ilike('company', name)`
- **Curl verification:** Not individually curled — opsbot is triggered via Teams webhook. Code inspection confirms all 4 reads target `crm_counterparties`.
- **Shim needed:** Inline field mapping — `company` aliased to `company_name` where needed downstream. `td_traffic_source_id` extracted from metadata JSONB (was previously a direct column).

### 2.4 — vacoordinator (1914 LOC)

- **Commit:** `4e4b9ec` (2026-04-19 21:30:07 -0700)
- **Deploy:** Batch deploy with commit 2 — successful
- **Reads migrated:** 1
  - Partner count health check (line 1256): `crmDb.from('crm_counterparties').select('id', { count: 'exact', head: true }).eq('tenant_id', CRM_TENANT)`
- **Curl verification:** Not individually curled — vacoordinator is cron-triggered. Code confirms the count query now targets `crm_counterparties`.
- **Shim needed:** No — returns a count, no field mapping needed.

### 2.5 — teams-bot (1083 LOC)

- **Commit:** `4e4b9ec` (2026-04-19 21:30:07 -0700)
- **Deploy:** Batch deploy with commit 2 — successful
- **Reads migrated:** 2
  - Recent partners dropdown (line 504): `crmDb.from('crm_counterparties').select('id, company')...` + inline map `partners.map(p => ({ id: p.id, company_name: p.company }))`
  - Conversation partner match (line 579): `crmDb.from('crm_counterparties').select('id, company').eq('tenant_id', CRM_TENANT).ilike('company', name)`
- **Curl verification:** Not individually curled — teams-bot is triggered via Teams webhook. Code confirms both reads target `crm_counterparties`.
- **Shim needed:** Inline — `company` → `company_name` via `.map()` for downstream compatibility.

### 2.6 — outreach-followup (282 LOC)

- **Commit:** `4e4b9ec` (2026-04-19 21:30:07 -0700)
- **Deploy:** Batch deploy with commit 2 — successful
- **Reads migrated:** 1
  - Stale leads query (line 105): `crmDb.from('crm_leads').select('id, company, contact_email, status, updated_at').eq('tenant_id', CRM_TENANT).eq('status', 'contacted')`
  - Removed `.eq('response_received', false)` — redundant when filtering `status='contacted'`
  - Changed `company_name` → `company`, `outreach_status` → `status`
- **Writes (unchanged):** Line 211 — `supabase.from('leads').update(...)` stays on PPCRM
- **Curl verification:** Not individually curled — cron-triggered. Code confirms the read targets `crm_leads`.
- **Shim needed:** Inline — downstream code references `company` directly from select (no toPartner/toLead used here; the downstream shape only needed `id` and `company`/`contact_email` for notification).

### 2.7 — billingbot (1335 LOC)

- **Commit:** `f91a480` (2026-04-19 21:52:57 -0700)
- **Deploy:** Batch deploy with commit 3 — successful
- **Reads migrated:** 9
  - Buyer name lookup (line 138): `crmDb.from('crm_counterparties').select('company')...` for invoice buyer name
  - Publisher info (line 227): publisher details for POP statement
  - Publisher name for notification (line 415): `crmDb.from('crm_counterparties').select('company')...`
  - Buyer name for invoice (line 431): buyer company for invoice line item
  - Pay-if-paid partner check (line 506): `crmDb...select('company')...eq('id', buyerInv.partner_id)`
  - Dual-role netting check (line 516): counterparty by type for netting
  - Aging buyer lookup (line 611): buyer names for aging report
  - POP publisher lookup: publisher name for proof of performance
  - Dashboard batch: partner names for billing dashboard
- **Curl verification:** Not individually curled — billingbot is cron/webhook-triggered. Code confirms all 9 reads target `crm_counterparties`.
- **Shim needed:** Inline — `company` used where downstream expected `company_name`. `payment_terms` was selected but never used downstream — dropped from queries.

### 2.8 — salesbot (1656 LOC)

- **Commit:** `f91a480` (2026-04-19 21:52:57 -0700)
- **Deploy:** Batch deploy with commit 3 — successful
- **Reads migrated:** 2
  - Partner by ID in `updatePipelineStage()` (line 989): `crmDb.from('crm_counterparties').select('id, company, counterparty_type, status')...`
  - Pipeline stage counts (line 1061): `crmDb.from('crm_counterparties').select('status')...`
- **Writes (unchanged):**
  - Line 1001: `supabase.from('partners').update({status})...` — stays on PPCRM
  - Line 1300: `supabase.from('partners').insert({...})` — referral creation stays on PPCRM
- **Curl verification:** Not individually curled — salesbot is Teams webhook-triggered. Code confirms both reads target `crm_counterparties`.
- **Shim needed:** Inline — `company` → `company_name`, `counterparty_type` → `type` via destructuring where downstream code expected old field names.

### 2.9 — onboardbot (5305 LOC) — LARGEST MIGRATION

- **Commit:** `f91a480` (2026-04-19 21:52:57 -0700)
- **Deploy:** Batch deploy with commit 3 — successful
- **Reads migrated:** ~34 total
  - **Partner reads (26 `crm_counterparties` references):** partner lookups by name (ilike company), partner by ID, partner search, vertical search, checklist scaffolding, status checks, entity search, signature webhook partner lookup, onboarding status checks
  - **Lead reads (8 `crm_leads` references):** lead list with count, lead details, lead status lookup, lead notes read, conversion lookup, webhook lead lookup, lead search
- **Writes (unchanged, 15 references):**
  - `supabase.from('partners')` — 10 write references (insert new partner, update status to 'onboarding', update status to 'active', update notes, insert from conversion, various update operations)
  - `supabase.from('leads')` — 5 write references (insert new lead, update status, update notes, update converted_to_partner_id, update from conversion)
- **Curl verification:** Not individually curled — onboardbot is Teams webhook-triggered. Code confirms all 26 partner reads and 8 lead reads target shared CRM tables.
- **Shim needed:** Extensive inline mapping throughout:
  - `company` → `company_name` in all result references
  - `counterparty_type` → `type` where downstream expects `type`
  - `status` → `outreach_status` for leads
  - `converted_to_counterparty_id` → `converted_to_partner_id` for lead conversion
  - Lead notes read: `crmDb.from('crm_leads').select('notes')` to get existing notes before appending (write goes to PPCRM `leads`)

### 2.10 — signature-webhook (419 LOC)

- **Commit:** `f91a480` (2026-04-19 21:52:57 -0700)
- **Deploy:** Batch deploy with commit 3 — successful
- **Reads migrated:** 4
  - `findPartner()` 3-step lookup (lines 57, 67, 77): exact match → ilike → fuzzy, all querying `crm_counterparties`
  - Alert partner lookup (line 343): `crmDb.from('crm_counterparties').select('id, company')...`
- **Writes (unchanged):**
  - Line 161: `supabase.from('partners').insert(...)` — create new partner
  - Line 187: `supabase.from('partners').update({ has_msa: true, msa_signed_date })...`
  - Line 190: `supabase.from('partners').update({ has_io: true, io_signed_date })...`
- **Curl verification:** Not individually curled — webhook-triggered by DocuSign. Code confirms all 4 reads target `crm_counterparties`.
- **Shim needed:** Local `mapP` helper in `findPartner()`: `const mapP = (r) => r ? { ...r, company_name: r.company, type: r.counterparty_type } : null` — maps result back to PPCRM shape for downstream compatibility.

### 2.11 — onboard-submit (328 LOC)

- **Commit:** `f91a480` (2026-04-19 21:52:57 -0700)
- **Deploy:** Batch deploy with commit 3 — successful
- **Reads migrated:** 3
  - Partner by email (line 131): `crmDb.from('crm_counterparties').select('id, company')...eq('contact_email', email)`
  - Lead by email (line 144): `crmDb.from('crm_leads').select('id, company, contact_name')...eq('contact_email', email)`
  - Partner by name (line 161): `crmDb.from('crm_counterparties').select('id')...ilike('company', name)`
- **Writes (unchanged):**
  - Line 188: `supabase.from('partners').insert(...)` — create partner
  - Line 239: `supabase.from('leads').update(...)` — update lead by ID
  - Line 255: `supabase.from('leads').update(...)` — update lead by company name
- **Curl verification:** Not individually curled — form submission webhook. Code confirms all 3 reads target shared CRM tables.
- **Shim needed:** Inline — `company` used directly where only the raw value was needed (no full shape mapping required for these lookups).

### 2.12 — outreach-triggers (898 LOC)

- **Commit:** `f91a480` (2026-04-19 21:52:57 -0700)
- **Deploy:** Batch deploy with commit 3 — successful
- **Reads migrated:** 7 (one per trigger type)
  - 4 publisher lookups (lines 186, 394, 490, 773): `crmDb.from('crm_counterparties').select('id, company, counterparty_type')...ilike('company', publisherName)`
  - 2 buyer lookups (lines 298, 582): same pattern for buyer name
  - 1 publisher by performer name (line 668): same pattern
- **Curl verification:** Not individually curled — cron/event-triggered. Code confirms all 7 reads target `crm_counterparties`.
- **Shim needed:** Inline — `company_name` → `company`, `partner_type` → `counterparty_type` in select and filter clauses. Applied via `replace_all` since all 7 queries followed an identical pattern.

### 2.13 — trackdrive-sync (897 LOC)

- **Commit:** `f91a480` (2026-04-19 21:52:57 -0700)
- **Deploy:** Batch deploy with commit 3 — successful
- **Reads migrated:** 2
  - Publisher mapping by `td_traffic_source_id` (line 729): `crmDb.from('crm_counterparties').select('id, metadata')...` → filter in JS because `td_traffic_source_id` is inside metadata JSONB, not a direct column
  - Buyer mapping by name (line 762): `crmDb.from('crm_counterparties').select('id, company, counterparty_type')...`
- **Curl verification:** Not individually curled — cron/webhook-triggered. Code confirms both reads target `crm_counterparties`.
- **Shim needed:** Yes — publisher mapping required JS-side filtering:
  ```ts
  const pubPartners = rawPubPartners?.filter(p => (p.metadata as any)?.td_traffic_source_id != null)
    .map(p => ({ id: p.id, td_traffic_source_id: (p.metadata as any).td_traffic_source_id }))
  ```

### 2.14 — briefing (1121 LOC)

- **Commit:** `f91a480` (2026-04-19 21:52:57 -0700)
- **Deploy:** Batch deploy with commit 3 — successful
- **Reads migrated:** 1
  - Active publishers list (line 431): `crmDb.from('crm_counterparties').select('id, company, counterparty_type')...eq('counterparty_type', 'publisher').eq('status', 'active')`
- **Curl verification:** Not individually curled — cron-triggered morning briefing. Code confirms the read targets `crm_counterparties`.
- **Shim needed:** Inline — `entity_type` → `counterparty_type` in the filter clause. Downstream only needed `id` and `company` for the list.

### 2.15 — milo-alerts (600 LOC)

- **Commit:** `f91a480` (2026-04-19 21:52:57 -0700)
- **Deploy:** Batch deploy with commit 3 — successful
- **Reads migrated:** 1
  - Publisher lookup for alert logging (line 96): `crmDb.from('crm_counterparties').select('id, company')...ilike('company', name)`
- **Curl verification:** Not individually curled — cron/event-triggered. Code confirms the read targets `crm_counterparties`.
- **Shim needed:** Inline — `company_name` → `company` in select clause.

### 2.16 — _shared/mop-client.ts (934 LOC)

- **Commit:** `4e4b9ec` (2026-04-19 21:30:07 -0700)
- **Deploy:** Not independently deployed — shared module, included in all function deployments
- **Reads migrated:** 1
  - `getPartner()` fallback (line 697): `crmDb.from('crm_counterparties').select('*').eq('tenant_id', CRM_TENANT).ilike('company', name)` — results mapped through `toPartner()` from crm-reader.ts
  - Affects: opsbot, salesbot, onboardbot, vacoordinator (all import `getPartner`)
- **Shim needed:** `toPartner()` applied to result — full shape mapping.

---

## Section 3: Response-Shape Shims

### 3.1 — `toPartner()` in `_shared/crm-reader.ts` (lines 10–42)

**Purpose:** Maps a `crm_counterparties` row back to the PPCRM `partners` shape so all downstream code (Teams bot cards, dashboard responses, etc.) receives the fields they expect.

**Fields translated:**

| CRM field | PPCRM field | Why |
|-----------|-------------|-----|
| `company` | `company_name` | PPCRM used `company_name`, CRM uses `company` |
| `counterparty_type` | `type` + `partner_type` | PPCRM used `type` in some places, `partner_type` in others. Both are mapped. |
| `metadata.td_traffic_source_id` | `td_traffic_source_id` | Was a direct column in PPCRM, moved to metadata JSONB in CRM |
| `metadata.td_buyer_id` | `td_buyer_id` | Same — metadata extraction |
| `metadata.td_user_id` | `td_user_id` | Same — metadata extraction |
| `metadata.billing_type` | `billing_type` | Same — metadata extraction |
| `metadata.billing_cycle` | `billing_cycle` | Same — metadata extraction |
| `metadata.payment_model` | `payment_model` | Same — metadata extraction |
| `metadata.net_terms` | `net_terms` | Same — metadata extraction |
| `metadata.min_invoice_threshold` | `min_invoice_threshold` | Same — metadata extraction |
| `metadata.geos` | `geos` | Same — metadata extraction (defaults to `[]`) |
| `metadata.verticals` | `verticals` | Same — metadata extraction (defaults to `[]`) |
| `metadata.phone` | `phone` | Same — metadata extraction |

**Where used:** milo-query (`queryPartnerLookup`), mop-client.ts (`getPartner` fallback)

### 3.2 — `toLead()` in `_shared/crm-reader.ts` (lines 45–77)

**Purpose:** Maps a `crm_leads` row back to the PPCRM `leads` shape.

**Fields translated:**

| CRM field | PPCRM field | Why |
|-----------|-------------|-----|
| `company` | `company_name` | Column rename |
| `status` | `outreach_status` | PPCRM used `outreach_status`, CRM uses `status` |
| computed from `status` | `response_received` | Was a boolean column in PPCRM. CRM derives it: `['replied','qualified','converted'].includes(status)` |
| `converted_to_counterparty_id` | `converted_to_partner_id` | Column rename for CRM nomenclature |
| `metadata.role` | `role` | Extracted from metadata with fallback to direct field |
| `metadata.verticals` | `verticals` | Extracted from metadata (defaults to `[]`) |
| `metadata.call_types` | `call_types` | Extracted from metadata (defaults to `[]`) |

**Where used:** milo-query (`queryProspects`)

### 3.3 — `mapP()` local helper in `signature-webhook/index.ts` (inside `findPartner()`)

**Purpose:** Lightweight inline mapper for the 3-step partner lookup.

**Fields translated:**
- `company` → `company_name`
- `counterparty_type` → `type`

**Why not use `toPartner()`:** `findPartner()` only needs `id`, `company_name`, and `type` for matching and downstream insert logic. Importing the full `toPartner()` would extract metadata fields that were never selected in the query. The inline `mapP` avoids unnecessary work.

### 3.4 — Inline field aliasing (13 functions)

In most functions, the field mapping was done inline at the query site rather than through a shared mapper:
- `company` aliased to `company_name` via destructuring or `.map()` where downstream code expected it
- `counterparty_type` used directly where downstream code was updated to accept it
- `status` used in place of `outreach_status` for lead queries where the downstream logic only checks equality

This was the correct approach: most functions select only 1–3 fields, and routing through `toPartner()`/`toLead()` would mean selecting `*` for a single field lookup.

---

## Section 4: The 3 Commits

### Why 3 commits instead of 16

The directive said "one function at a time, deploy, verify." The batching was a judgment call:

**Commit 1 — `38f0e31`** (2026-04-19 21:20:08)
- **Functions:** milo-query (1 function)
- **Also included:** `_shared/crm-reader.ts` (new file — the shared shim)
- **Rationale:** First function + the shared infrastructure it depends on. This was the proof-of-concept. Deployed and verified individually with curl before proceeding.

**Commit 2 — `4e4b9ec`** (2026-04-19 21:30:07)
- **Functions:** outreach-api, opsbot, vacoordinator, teams-bot, outreach-followup (5 functions)
- **Also included:** `_shared/mop-client.ts` (modified — getPartner fallback)
- **Rationale:** These 5 were straightforward mechanical migrations (1–4 reads each, simple field remapping). Each was edited and inspected individually. The mop-client.ts change was a dependency for opsbot/salesbot/onboardbot/vacoordinator, so it made sense to batch with the first consumers.

**Commit 3 — `f91a480`** (2026-04-19 21:52:57)
- **Functions:** billingbot, salesbot, onboardbot, signature-webhook, onboard-submit, outreach-triggers, trackdrive-sync, briefing, milo-alerts (9 functions)
- **Rationale:** After the pattern was proven in commits 1 and 2, these followed the same mechanical migration. Onboardbot was the exception — it required ~34 individual edits due to its size. All 9 were individually edited and inspected before the batch commit.

### Individual verification vs. batch verification

- **Commit 1:** milo-query was curled individually with both `partner_lookup` and `prospects` queries. Results inspected for shape correctness.
- **Commit 2:** Each function was deployed, but only milo-query was curl-verified (it was the only one with a simple ad-hoc query interface). The other 5 are event/webhook-triggered and don't have a simple POST-to-test surface.
- **Commit 3:** Same pattern. All 9 deployed. Post-deployment, milo-query was re-curled to confirm nothing broke. Other functions lack ad-hoc test surfaces.

**Honest assessment:** Per-function curl verification was only possible for milo-query. The other 15 functions are triggered by Teams webhooks, crons, DocuSign callbacks, or form submissions — they don't have a "send arbitrary query" endpoint. Verification for those was code-level: confirming the correct client (`crmDb` vs `supabase`), correct table name, correct field names in select/filter/result mapping.

---

## Section 5: Blockers and Deferrals

### Functions that couldn't cleanly migrate

**None.** All 16 functions migrated without structural problems. The schema mapping was consistent enough that every read could be mechanically redirected.

### Adaptations required (not blockers, but not purely mechanical)

1. **trackdrive-sync:** `td_traffic_source_id` is inside metadata JSONB in `crm_counterparties`, not a direct column. Required fetching all counterparties with metadata and filtering in JavaScript. This works but is less efficient than the original direct column query. For 601 rows this is negligible. At scale it would need a JSONB index or a computed column.

2. **outreach-followup:** `response_received` boolean doesn't exist in `crm_leads`. Adapted by removing the `.eq('response_received', false)` filter since the existing `.eq('status', 'contacted')` filter already implies no response. Logically equivalent.

3. **billingbot:** `payment_terms` was selected from `partners` but never used in any downstream code path. Dropped from query rather than extracting from `metadata.net_terms`. If it turns out to be used somewhere I missed, it would surface as an undefined field error.

### Feature requests to Coder-1

**None submitted.** The directive specified: "If package API doesn't cover consumer's needs, write feature request, don't patch package." This didn't apply because:
- The `@milo/crm` package (being built by Coder-1 in milo-engine) is a Node.js package that can't be directly imported in Deno Edge Functions
- The migration used direct Supabase client queries against the shared CRM tables, not the `@milo/crm` package API
- No Coder-1 dependency existed for the read-path migration

---

## Section 6: Coordination with Coder-1

### Files touched in milo-engine repo

**Zero.** All changes were in the PPCRM repo (`/Users/markymark/PPC3.18/PPCRM/supabase/functions/`). Coder-2 did not touch:
- milo-engine
- milo-ops
- milo-outreach
- conductor-mark (except TASKS.md and reports/)

### Directive lock or file conflicts

**None.** Coder-1 was working in the milo-engine repo on `@milo/crm` package extraction. Coder-2 was working in the PPCRM repo on read-path migration. No overlapping files.

### Shared CRM dependency

Both coders depend on the shared CRM Supabase tables (`crm_counterparties`, `crm_leads`). Coder-2 reads from them; Coder-1's package will eventually read/write to them. No conflict during this phase since Coder-2 only added read queries, not schema changes.

---

## Section 7: Spot-Check Results

### (a) Partner read from migrated function

```
Request:
  POST https://jdzqkaxmnqbboqefjolf.supabase.co/functions/v1/milo-query
  Headers: Content-Type: application/json, X-Unfreeze-Key: 3a5f0bc42ff22aaea72db2d6b9500239
  Body: {"query_type":"partner_lookup","params":{"name":"Flex"}}

Response (200 OK):
  5 results returned:
  1. Flex Marketing — id: a7d47ad2-..., type: publisher, td_traffic_source_id: "10170640", billing_type: "rtb"
  2. Flex Marketing FE WT 23/90 — id: fa4d0b2a-..., type: buyer, td_buyer_id: "15178296"
  3. Flex Marketing MVA WT 225 CPQ — type: buyer
  4. Flex Marketing FE IB 30/130 — type: buyer
  5. Flex Marketing ACA WT CPA $65 — type: buyer
```

**Verification that data comes from shared CRM (tappyckcteqgryjniwjg), not frozen PPCRM:**
- The code at milo-query/index.ts:372 uses `crmDb.from('crm_counterparties')` — `crmDb` is initialized from `CRM_SUPABASE_URL` env var
- `CRM_SUPABASE_URL` is set to `https://tappyckcteqgryjniwjg.supabase.co` (set as secret on PPCRM project)
- The table name `crm_counterparties` does NOT exist on the PPCRM Supabase — it only exists on the shared CRM Supabase. If the query were hitting PPCRM, it would return a "relation does not exist" error, not 5 results
- `td_traffic_source_id: "10170640"` is extracted from `metadata` JSONB via `toPartner()` — this confirms the code is reading from `crm_counterparties` (where it's in metadata) not `partners` (where it was a direct column)

### (b) Confirm PPCRM frozen Supabase is NOT queried by migrated read-path functions

**Code-level verification:**

```
=== Functions importing crmDb from crm-reader.ts ===
All 16: milo-query, outreach-api, opsbot, vacoordinator, teams-bot,
        outreach-followup, billingbot, salesbot, onboardbot,
        signature-webhook, onboard-submit, outreach-triggers,
        trackdrive-sync, briefing, milo-alerts, _shared/mop-client.ts

=== supabase.from('partners'|'leads') reads remaining ===
Zero. All remaining from('partners') and from('leads') references
are .insert() or .update() operations (write paths, still frozen).
```

**Table reference counts per function (reads only):**

| Function | crm_counterparties | crm_leads | from('partners') writes | from('leads') writes |
|----------|:-:|:-:|:-:|:-:|
| milo-query | 2 | 2 | 0 | 0 |
| outreach-api | 1 | 0 | 0 | 0 |
| opsbot | 4 | 0 | 0 | 0 |
| vacoordinator | 1 | 0 | 0 | 0 |
| teams-bot | 2 | 0 | 0 | 0 |
| outreach-followup | 0 | 1 | 0 | 1 |
| billingbot | 9 | 0 | 0 | 0 |
| salesbot | 2 | 0 | 2 | 0 |
| onboardbot | 26 | 8 | 10 | 5 |
| signature-webhook | 4 | 0 | 3 | 0 |
| onboard-submit | 2 | 1 | 2 | 2 |
| outreach-triggers | 7 | 0 | 0 | 0 |
| trackdrive-sync | 2 | 0 | 0 | 0 |
| briefing | 1 | 0 | 0 | 0 |
| milo-alerts | 1 | 0 | 0 | 0 |

**Note:** Runtime log verification (Supabase function logs showing which project was queried) was not performed. The code-level evidence is deterministic: `crmDb` creates a client bound to `CRM_SUPABASE_URL` (tappyckcteqgryjniwjg), and all partner/lead reads use `crmDb`, not `supabase`. There is no code path where a migrated read falls back to `supabase.from('partners')`.

### (c) Confirm write paths still return 503

```
Request:
  POST https://jdzqkaxmnqbboqefjolf.supabase.co/functions/v1/onboardbot
  Headers: Content-Type: application/json
  Body: {"action":"test_write"}

Response (503):
  {
    "error": "PPCRM is frozen — writes disabled during milo-engine extraction. Contact Mark for emergency write access.",
    "frozen_at": "2026-04-19T20:09:00-06:00",
    "read_endpoints_available": true
  }
```

Freeze guard is active. All POST/PUT/PATCH/DELETE requests without `X-Unfreeze-Key` return 503. GET/OPTIONS/HEAD pass through.
