---
directive: "Morgan schema reconciliation prep doc"
lane: research
coder: Coder-3
started: 2026-04-19 ~23:00 MDT
completed: 2026-04-19 ~23:45 MDT
---

# Schema Reconciliation: milo-engine vs tlp-platform — 2026-04-19

**Purpose:** Ready-for-Mark-and-Morgan conversation doc. Identifies conflicts, proposes resolutions, surfaces open questions.

## Executive Summary

milo-engine has 7 tables across 3 primitives (CRM, blacklist, contract-analysis). tlp-platform has 32 tables in a unified migration set. There is **direct overlap on 9 of the 32 tables**, with the deepest conflict being the CRM entity model: milo-engine unifies publishers/buyers into one `counterparties` table with a `counterparty_type` column, while tlp-platform keeps separate `publishers` (63 columns) and `buyers` (67 columns) tables — a fundamental architectural disagreement. Blacklist schemas are nearly identical and can converge easily. Contract schemas have moderate conflict: tlp-platform merged 4 MOP tables into one mega `contracts` table (50+ columns) while milo-engine kept a focused `contract_analyses` table. The multi-tenancy key is also mismatched: milo-engine uses `tenant_id TEXT`, tlp-platform uses `organization_id UUID FK`. Resolution requires 3-4 product decisions from Mark and Morgan.

## Side-by-Side Table Inventory

### Overlapping Tables (9 of 32)

| milo-engine Primitive | milo-engine Table | tlp-platform Table(s) | Relationship |
|---|---|---|---|
| @milo/crm | `crm_counterparties` | `publishers` + `buyers` + `company_groups` | **CONFLICT** — milo-engine unifies via counterparty_type; tlp splits into separate tables with 63-67 columns each |
| @milo/crm | `crm_leads` | `entity_pipeline` | **Partial overlap** — similar concept (pre-entity tracking) but different scope: milo leads are pre-counterparty records; tlp entity_pipeline is full prospect lifecycle with stage machine |
| @milo/crm | `crm_contacts` | `contacts` | **Near-identical** — both are person-level models with optional entity FK |
| @milo/crm | `crm_activities` | (no direct equivalent) | **Gap** — tlp-platform has per-domain audit logs (invoice_audit_log, contract_template_audit_log, user_role_history) but no unified activity log |
| @milo/blacklist | `blacklist_entries` | `blacklist` | **Near-identical** — same core design, minor column differences |
| @milo/contract-analysis | `contract_analyses` | `contracts` | **CONFLICT** — tlp merged 4 MOP tables (contract_analyses + signing_documents + finalized_documents + partner_documents) into one mega table; milo-engine kept analysis-only |
| @milo/contract-analysis | `contract_analysis_jobs` | `contract_analysis_jobs` | **Near-identical** — same purpose, milo-engine replaced 3 PPC columns with generic `analysis_config` JSONB |
| (future @milo/contract-*) | — | `contract_negotiations` | milo-engine hasn't extracted this yet; tlp-platform has it |
| (future @milo/contract-*) | — | `negotiation_rounds` + `negotiation_links` | Same — not yet extracted |

### Non-Overlapping Tables in tlp-platform (23 of 32)

See Section 6 below.

## Column-Level Conflicts

### Conflict 1: CRM Entity Model (CRITICAL)

**milo-engine `crm_counterparties`** (25 columns):

| Column | Type | Notes |
|--------|------|-------|
| tenant_id | TEXT | Multi-tenancy key |
| counterparty_type | TEXT | CHECK: publisher, buyer, vendor, partner, client, prospect |
| company | TEXT | Company name |
| status | TEXT | 9 values: lead, contacted, qualified, onboarding, active, paused, churned, blacklisted, archived |
| contact_name/email/phone/telegram | TEXT | Single contact fields |
| linkedin_url, website | TEXT | |
| address/city/state/zip/country | TEXT | |
| source, referred_by, assigned_to | TEXT | |
| notes | TEXT | |
| metadata | JSONB | Catch-all |

**tlp-platform `publishers`** (63 columns) + **`buyers`** (67 columns):

Everything milo-engine has, PLUS:

| Category | Columns (not in milo-engine) | Count |
|----------|------------------------------|-------|
| Billing | billing_type, billing_cycle, net_terms, min_invoice_threshold, payment_model, payment_terms, payment_method, payout, payout_type, duration | 10 |
| Vertical/Geo | vertical, verticals[], geos[], states, center_type, center_location, transfer_type, transfer_types[], lead_type, lead_type_category, inbound_source, call_type | 12 |
| Operations | hoo, campaign_name, did_rtb_id, rep_name, ad_lander, teams_chat_link, jornaya_tf, posting_instructions_url, io_url | 9 |
| Compliance | has_msa, has_io, msa_signed_date, io_signed_date, contract_expires_at, chargeback_cap_percent, chargeback_window_days, governing_law_state, term_length_months, non_circumvention, tax_doc_type | 11 |
| TrackDrive | td_traffic_source_id, td_user_id | 2 |
| External IDs | ghl_contact_id, quickbooks_id | 2 |
| Assignment | assigned_va (UUID FK), assigned_agent | 2 |
| Blacklist inline | blacklist_status, blacklist_reason, blacklist_date | 3 |
| company_group_id | UUID FK | 1 |
| Buyer-only | call_types_accepted[], cc, daily_cap, ping_post_instructions, convoqc_enabled, ad_lander_url, td_buyer_id | 7 |

**Key differences:**

| Aspect | milo-engine | tlp-platform |
|--------|-------------|--------------|
| Table count | 1 unified table | 2 separate tables + company_groups link table |
| Column count | 25 | 63 (pub) + 67 (buy) |
| Multi-tenancy key | `tenant_id TEXT` | `organization_id UUID FK` |
| Contact model | Single contact fields inline | Single contact fields inline + separate `contacts` table |
| Status values | 9 (lead through archived) | 5 (active, paused, blacklisted, archived, inactive) |
| Billing fields | None (deferred to vertical) | 10 columns inline |
| Compliance fields | None (deferred to @milo/contracts) | 11 columns inline |
| Vertical/geo fields | None (deferred to vertical) | 12 columns inline |
| Operations fields | None (deferred to vertical) | 9 columns inline |
| Blacklist handling | Separate @milo/blacklist table + 'blacklisted' status | Inline blacklist fields + separate blacklist table |

### Conflict 2: Leads vs Entity Pipeline

**milo-engine `crm_leads`** (24 columns):
- Pre-counterparty records with research tracking
- Status: new, researched, contacted, replied, qualified, rejected, converted
- Has `lead_score` (0-100), `research_status`, `converted_to_counterparty_id`
- Converts to counterparty via FK

**tlp-platform `entity_pipeline`** (30 columns):
- Prospect lifecycle with stage machine
- Stage: outreach, qualifying, drip, onboarding, activation, active, dormant, blacklisted
- Has `intelligence_brief`, `risk_flags`, `margin_analysis`, `broker_status/depth`
- Links to both `publisher_id` and `buyer_id` FKs
- Has `verticals`, `call_types`, `coverage_states`, `payout_discussed`, `billing_type` — PPC-specific fields

| Aspect | milo-engine crm_leads | tlp-platform entity_pipeline |
|--------|----------------------|------------------------------|
| Purpose | Generic lead tracking | PPC prospect lifecycle |
| Stages | 7 (new → converted) | 8 (outreach → blacklisted) |
| Conversion target | counterparty_id FK | publisher_id + buyer_id FKs |
| PPC fields | None | verticals, call_types, coverage_states, payout_discussed, billing_type, broker_status/depth |
| Intelligence | lead_score (int 0-100) | intelligence_brief (JSONB), risk_flags (JSONB), margin_analysis (JSONB) |
| Research tracking | research_status field | None |
| Contact tracking | first_contact_at, last_contact_at | contact_id FK |

### Conflict 3: Contacts — Near Identical

| Column | milo-engine crm_contacts | tlp-platform contacts | Match? |
|--------|-------------------------|----------------------|--------|
| tenant key | tenant_id TEXT | organization_id UUID FK | **Mismatch** |
| entity FK | counterparty_id UUID | (none — uses entity_type TEXT) | **Structural** |
| lead FK | lead_id UUID | (none) | **Missing in tlp** |
| name | TEXT NOT NULL | TEXT NOT NULL | Match |
| company | (via counterparty FK) | TEXT | **Different approach** |
| email | TEXT | TEXT | Match |
| phone | TEXT | TEXT | Match |
| role | TEXT | TEXT | Match |
| entity_type | (inferred from FK) | TEXT CHECK (publisher, buyer) | **Different approach** |
| is_primary | BOOLEAN | (none) | **Missing in tlp** |
| linkedin_url | TEXT | TEXT | Match |
| source | TEXT | TEXT | Match |
| tags | JSONB | JSONB | Match |
| last_contacted_at | TIMESTAMPTZ | TIMESTAMPTZ | Match |
| notes | TEXT | TEXT | Match |

### Conflict 4: Blacklist — Near Identical

| Column | milo-engine blacklist_entries | tlp-platform blacklist | Match? |
|--------|------------------------------|----------------------|--------|
| tenant key | tenant_id TEXT | organization_id UUID FK | **Mismatch** |
| company_name | TEXT NOT NULL | TEXT NOT NULL | Match |
| company_aliases | JSONB | JSONB | Match |
| contact_names | JSONB | JSONB | Match |
| email | TEXT (single) | (none — uses email_domains) | **Different** |
| email_domains | JSONB | JSONB | Match |
| phone | TEXT (single) | (none — uses phone_numbers) | **Different** |
| phone_numbers | (none) | JSONB | **Missing in milo** |
| ip_addresses | (none) | JSONB | **Missing in milo** |
| linkedin_urls | (none) | JSONB | **Missing in milo** |
| registration_states | (none) | JSONB | **Missing in milo** |
| reason | TEXT NOT NULL | TEXT NOT NULL | Match |
| severity | TEXT CHECK | TEXT CHECK | Match (same values) |
| evidence | JSONB | JSONB | Match |
| source | TEXT | TEXT | Match |
| source_reference | TEXT | (none) | **Missing in tlp** |
| added_by | TEXT | (blacklisted_by TEXT) | **Renamed** |
| reviewed_by | TEXT | (none) | **Missing in tlp** |
| expires_at | TIMESTAMPTZ | (none) | **Missing in tlp** |
| is_active | BOOLEAN | (none) | **Missing in tlp** |
| deleted_at | TIMESTAMPTZ | (none) | **Missing in tlp** |
| verticals | (none) | TEXT | **Missing in milo** |
| blacklisted_at | (via created_at) | TIMESTAMPTZ explicit | **Different naming** |
| notes | TEXT | TEXT | Match |

### Conflict 5: Contract Analysis

**milo-engine `contract_analyses`** (26 columns) — analysis-focused:
- Has `tenant_id TEXT`, `score JSONB` (persisted), `metadata JSONB`
- Removed MOP's `publisher_id`/`buyer_id` FKs, `negotiation_stage`, `is_finalized`
- No signing columns

**tlp-platform `contracts`** (50+ columns) — merged mega-table:
- Has `organization_id UUID FK`
- Includes ALL of: analysis (raw_text, summary, issues, analysis_result, risk_level, risk_score), signing (signing_token, signature_data/hash/ip, counterparty_* fields, viewed_at, signed_at), versioning (contract_group_id, parent_version_id, version_number), storage (storage_path, file_size, mime_type), templates (template_version_id)
- Keeps PPC-specific: `pass_through_terms`, `changes_from_prior`

| Aspect | milo-engine | tlp-platform |
|--------|-------------|--------------|
| Scope | Analysis only | Analysis + signing + storage + versioning |
| Table count | 1 (analysis) | 1 (mega merged) |
| Signing columns | None (deferred to @milo/contract-signing) | 15+ signing columns inline |
| Entity FKs | None (uses counterparty_info JSONB) | publisher_id, buyer_id FKs |
| Template FK | None | template_version_id FK |
| PPC columns | None (uses analysis_config JSONB) | playbook_preset, threshold_profile, analysis_profile |
| Score | JSONB (persisted) | (computed from issues) |

### Conflict 6: Contract Analysis Jobs

| Column | milo-engine | tlp-platform | Match? |
|--------|-------------|--------------|--------|
| tenant key | tenant_id TEXT | organization_id UUID FK | **Mismatch** |
| PPC config | analysis_config JSONB (generic) | playbook_preset + threshold_profile + analysis_profile (3 TEXT columns) | **Structural** |
| system_prompt | TEXT (injectable) | (none) | **Missing in tlp** |
| contract FK | analysis_id UUID FK contract_analyses | contract_id UUID FK contracts | **Renamed** |

### Conflict 7: Multi-Tenancy Key (FUNDAMENTAL)

| Aspect | milo-engine | tlp-platform |
|--------|-------------|--------------|
| Column name | `tenant_id` | `organization_id` |
| Type | `TEXT NOT NULL` | `UUID NOT NULL` |
| FK target | None (freeform) | `organizations(id)` |
| Root table | None | `organizations` (id, name, slug, settings) |

This affects **every table** in both schemas. Resolution required before any migration.

## Proposed Resolutions

### Resolution 1: CRM Entity Model — Recommend Mark decides

**Option A: tlp-platform adopts milo-engine's unified `counterparties` approach.**
- Pro: Simpler schema, no company_groups link table, extensible counterparty_type
- Con: Loses 40+ PPC-specific columns (billing, compliance, vertical, ops, TrackDrive). These would need to move to a vertical extension table or JSONB metadata.
- Risk: Major restructuring of Morgan's schema. All 32 tables that FK to publishers/buyers would need updating.

**Option B: milo-engine amends to match tlp-platform's split publishers/buyers.**
- Pro: Zero disruption to Morgan's schema. PPC-specific columns stay type-safe.
- Con: Violates Thesis 2.5 (horizontal primitive should work for any vertical). Future verticals would need their own entity tables.
- Risk: Would trigger Decision 38 schema amendment in milo-engine.

**Option C: Hybrid — milo-engine's `counterparties` as horizontal base, tlp-platform's publishers/buyers as vertical extension views or tables that FK to counterparties.**
- Pro: Both schemas survive. Horizontal primitive stays clean. Vertical gets its columns.
- Con: More complex join patterns. Migration requires adding counterparty_id to publishers/buyers.
- Recommended path if neither side wants to fully yield.

**Recommendation for conversation:** Start with Option C. It's the Thesis 2.5 approach — horizontal base + vertical extension.

### Resolution 2: Leads vs Entity Pipeline — Both stay, different purposes

milo-engine's `crm_leads` is a generic pre-counterparty record (any vertical). tlp-platform's `entity_pipeline` is a PPC prospect lifecycle tracker. These serve different purposes. entity_pipeline could become a PPC vertical consumer of crm_leads, with its PPC-specific fields (verticals, call_types, broker_status, margin_analysis) in the vertical layer.

**Recommendation:** milo-engine keeps crm_leads as the horizontal primitive. tlp-platform's entity_pipeline becomes a PPC extension that FKs to crm_leads or crm_counterparties.

### Resolution 3: Contacts — Converge on milo-engine's richer model

milo-engine's `crm_contacts` has `is_primary`, `lead_id FK`, and `counterparty_id FK`. tlp-platform's `contacts` has `company TEXT` inline and `entity_type TEXT` instead of FKs. milo-engine's design is stronger (FK-enforced relationships, multi-entity support).

**Recommendation:** tlp-platform adopts milo-engine's contact model, adds `counterparty_id` and `lead_id` FKs, drops inline `company` and `entity_type`.

### Resolution 4: Blacklist — Converge with minor additions

Both schemas are 90% identical. Differences are additive, not conflicting.

**Recommendation:** milo-engine adds tlp-platform's extra signal arrays: `phone_numbers JSONB`, `ip_addresses JSONB`, `linkedin_urls JSONB`, `registration_states JSONB`, `verticals TEXT`. tlp-platform adopts milo-engine's lifecycle columns: `source_reference`, `reviewed_by`, `expires_at`, `is_active`, `deleted_at`. Rename: milo's `added_by` = tlp's `blacklisted_by` (pick one). milo's `email TEXT` can coexist with `email_domains JSONB`.

### Resolution 5: Contract Analysis — Recommend splitting tlp's mega-table

tlp-platform merged 4 MOP tables into one 50+ column `contracts` mega-table. milo-engine kept analysis-focused. The mega-table combines unrelated concerns (analysis + signing + storage + versioning) that have different lifecycles.

**Recommendation:** tlp-platform splits `contracts` back into domain-aligned tables matching milo-engine's extraction plan:
- `contract_analyses` (from @milo/contract-analysis)
- `signing_documents` (future @milo/contract-signing)
- `contract_negotiations` + `negotiation_rounds` + `negotiation_links` (future @milo/contract-negotiation)
- Keep `contract_templates` + `contract_template_versions` + `contract_template_audit_log` as-is (no milo-engine equivalent yet)

This aligns with the extraction architecture: each `@milo/*` package owns its own tables.

### Resolution 6: Contract Analysis Jobs — Converge on milo-engine's generic design

milo-engine replaced 3 PPC-specific TEXT columns with `analysis_config JSONB` and added `system_prompt TEXT` for injectable prompts. This is the horizontal design.

**Recommendation:** tlp-platform adopts milo-engine's generic shape. PPC-specific config (playbook_preset, threshold_profile, analysis_profile) goes into `analysis_config` JSONB.

### Resolution 7: Multi-Tenancy Key — Must align

**Option A: Both use `tenant_id TEXT`** (milo-engine's approach)
- Pro: Simpler, no FK constraint, works before organizations table exists
- Con: No referential integrity, freeform text can drift

**Option B: Both use `organization_id UUID FK organizations`** (tlp-platform's approach)
- Pro: Referential integrity, UUID performance, Morgan's schema is already built
- Con: milo-engine would need to ship an `organizations` table or accept the FK dependency

**Option C: milo-engine uses `tenant_id TEXT`, consumers map it**
- Pro: Horizontal primitive stays simple. Each consumer (including tlp-platform) maps tenant_id to their own multi-tenancy model.
- Con: No system-level enforcement of tenant isolation in milo-engine

**Recommendation:** Option B with a compromise — milo-engine primitives define `tenant_id UUID NOT NULL` (change type from TEXT to UUID) but do NOT define the FK constraint. The consumer's migration adds the FK to their organizations/tenants table. This gives UUID type safety without coupling milo-engine to a specific organizations schema.

## Non-Overlapping Tables in tlp-platform

### Tables that map to future milo-engine primitives

| tlp-platform Table | Future Primitive | Notes |
|---|---|---|
| `contract_negotiations` | @milo/contract-negotiation | Already audited in Coder-3 negotiation report |
| `negotiation_rounds` | @milo/contract-negotiation | Same |
| `negotiation_links` | @milo/contract-negotiation | Same |
| `contract_templates` | @milo/contract-templates (future) | Per-org template definitions with type constraints |
| `contract_template_versions` | @milo/contract-templates (future) | Immutable versions with `{{variable}}` syntax |
| `contract_template_audit_log` | @milo/contract-templates (future) | Template change audit |
| `contract_issue_decisions` | @milo/contract-analysis (extend) | Per-issue accept/reject — currently in MOP, not yet in milo-engine SCHEMA.md |
| `contract_issue_resolutions` | @milo/contract-analysis (extend) | Resolution details — same note |
| `invoices` | @milo/invoice-gen (future) | 55+ column mega-invoice table |
| `invoice_line_items` | @milo/invoice-gen (future) | Per-DID line items |
| `invoice_audit_log` | @milo/invoice-gen (future) | Invoice change trail |

### Tables with no planned milo-engine equivalent

| tlp-platform Table | Category | Purpose | Notes |
|---|---|---|---|
| `organizations` | System | Multi-tenancy root | milo-engine has no equivalent (uses freeform tenant_id) |
| `users` | System | Operator/user table (11 seeded TLP team members) | Not a horizontal primitive — per-org |
| `user_role_history` | System | Role change audit | Not a horizontal primitive |
| `company_groups` | CRM | Links publisher + buyer records for same company | milo-engine handles via counterparty_type (same record serves both roles) |
| `call_logs` | Operations | 50+ column unified call data | PPC-specific (TrackDrive, ConvoQC, conversion tracking) |
| `campaigns` | Operations | 45+ column campaign definitions | PPC-specific |
| `campaign_routes` | Operations | Publisher/buyer routing per campaign | PPC-specific |
| `campaign_qualifiers` | Operations | Per-campaign qualification criteria | PPC-specific |
| `milo_conversations` | Operations | Daily chat threads | Candidate for @milo/ai-client extension |
| `verticals` | Reference | Vertical definitions | PPC reference data |
| `vertical_synonyms` | Reference | Synonym mapping | PPC reference data |
| `ping_source_map` | Reference | Ping source to publisher mapping | PPC-specific |
| `call_disputes` | Disputes | CPA conversion disputes | PPC-specific |
| `report_disputes` | Disputes | Buyer report financial disputes | PPC-specific |

### Alternative schema (ScaleFlow Architecture, older)

tlp-platform also has an older 21-table alternative schema (`docs/scaleflow-architecture/`) with different design decisions:
- Uses unified `entities` table (like milo-engine's counterparties)
- Adds `insertion_orders`, `routes`, `workflow_definitions/instances`, `pipeline_stages`, `alerts`, `milo_actions`, `review_queue`, `milo_memory` (with pgvector)
- Uses Supabase Auth (`auth.users` FK) instead of standalone users table

**Key question for Morgan:** Which schema generation is authoritative? The migration-scripts (32-table, separate publishers/buyers) or the scaleflow-architecture (21-table, unified entities)? If the latter, the entity model conflict with milo-engine largely disappears.

## Open Questions for Mark and Morgan

**Q1: Unified entities or separate publishers/buyers?**
Context: This is the single biggest schema conflict. milo-engine's `counterparties` unifies all entity types. tlp-platform's migration-scripts split publishers (63 cols) and buyers (67 cols). But tlp-platform's older scaleflow-architecture schema ALSO uses a unified `entities` table — suggesting Morgan may have already considered and rejected (or deferred) unification.
Recommended path: Discuss Option C hybrid (horizontal counterparties base + vertical extension tables).

**Q2: Which tlp-platform schema generation is authoritative?**
Context: Two schema sets exist in the repo — `docs/migration-scripts/` (32 tables, split entities) and `docs/scaleflow-architecture/schema/` (21 tables, unified entities). They share concepts but differ structurally. The migration-scripts version appears newer and more complete.
Recommended path: Morgan confirms which is the build target. If migration-scripts, the entity model conflict is real. If scaleflow-architecture, it's largely resolved.

**Q3: Multi-tenancy key type — TEXT or UUID?**
Context: milo-engine uses `tenant_id TEXT`, tlp-platform uses `organization_id UUID FK organizations(id)`. Every table in both schemas is affected.
Recommended path: Agree on UUID type (for performance and consistency), with FK constraint optional per consumer.

**Q4: Should tlp-platform's mega `contracts` table be split?**
Context: tlp-platform merged 4 MOP tables into one 50+ column table. milo-engine's extraction plan creates separate packages for analysis, signing, and negotiation — each with its own tables. A mega-table fights the package architecture.
Recommended path: Split contracts back to domain-aligned tables matching milo-engine primitives.

**Q5: What about the 40+ PPC columns on publishers/buyers?**
Context: If the entity model unifies, these columns (billing, compliance, vertical, ops, TrackDrive) need a home. Options: JSONB metadata, vertical extension table, or inline nullable columns on counterparties.
Recommended path: Vertical extension table (`ppc_publisher_details`, `ppc_buyer_details`) that FKs to counterparties.

**Q6: entity_pipeline vs crm_leads — coexist or merge?**
Context: Different abstractions: crm_leads is generic pre-entity tracking; entity_pipeline is PPC prospect lifecycle with intelligence_brief, risk_flags, margin_analysis. Both track prospect-to-entity conversion but with different schemas.
Recommended path: entity_pipeline becomes a PPC consumer of crm_leads, with PPC-specific fields in the pipeline layer.

**Q7: Contract template versioning ownership.**
Context: tlp-platform has `contract_templates` + `contract_template_versions` + `contract_template_audit_log` (3 tables). milo-engine has no template primitive yet. Should this become a @milo/contract-templates package, or stay in the PPC vertical?
Recommended path: Template versioning is horizontal (any vertical needs versioned templates). Extract as @milo/contract-templates. The `{{variable}}` syntax and version lifecycle are domain-agnostic.

**Q8: Issue decisions and resolutions — extend milo-engine's contract-analysis or separate package?**
Context: tlp-platform has `contract_issue_decisions` and `contract_issue_resolutions`. milo-engine's contract-analysis SCHEMA.md doesn't include them. They're needed for the negotiation workflow.
Recommended path: Add to @milo/contract-analysis v0.2.0, since they're tightly coupled to analysis results.

## Recommended Conversation Sequence

If Mark and Morgan sit down tomorrow, resolve in this order:

1. **Which tlp-platform schema is authoritative?** (Q2) — If the scaleflow-architecture schema is the target, several conflicts dissolve. 5 minutes.

2. **Multi-tenancy key alignment** (Q3) — Affects every table. Agree on UUID type, FK optional. 5 minutes.

3. **Entity model: unified or split** (Q1) — The biggest decision. Option C hybrid takes 20 minutes to design. Drives everything else.

4. **PPC column placement** (Q5) — Once entity model is decided, figure out where billing/compliance/ops columns live. 10 minutes.

5. **Mega contracts table: split or keep** (Q4) — Aligns tlp-platform with milo-engine's package-per-domain architecture. 10 minutes.

6. **Leads vs entity_pipeline** (Q6) — Lower stakes, can coexist. 5 minutes.

7. **Template versioning ownership** (Q7) — Straightforward: horizontal or vertical? 5 minutes.

8. **Issue decisions/resolutions scope** (Q8) — Extend contract-analysis or new package. 5 minutes.

**Total estimated conversation time: ~65 minutes.**
