# Multi-Tenant + Tool Architecture Audit

**Date:** 2026-05-06
**Author:** Coder-3 (Research)
**Type:** Read-only audit. No code changes.
**Scope:** milo-for-ppc, milo-engine packages, milo-outreach

---

## Section 1 -- Coupling Analysis + Recommended Phasing

### Current Single-Tenant Coupling Map

The codebase has **three distinct coupling tiers** that determine migration difficulty:

**Tier 1 -- Loosely Coupled (cheap to decouple)**

These already accept `tenantId` as a parameter. Migration is config change + UUID swap:

- `@milo/crm` -- `createCrmClient({ tenantId })` in `packages/crm/src/client.ts`
- `@milo/blacklist` -- `createBlacklistClient({ tenantId })` in `packages/blacklist/src/client.ts`
- `@milo/onboarding` -- `createOnboardingClient({ tenantId })` in `packages/onboarding/src/client.ts`
- `@milo/contract-signing` -- `createSigningClient({ tenantId })` in `packages/contract-signing/src/client.ts`
- `milo-outreach` -- Full tenant model already exists: `tenants` table, `getTenant()`, `authenticateTenantApiKey()`, per-tenant send gates, per-tenant API keys. This is the most multi-tenant-ready component.

**Tier 2 -- String-Hardcoded (moderate refactor)**

These have `'tlp'` hardcoded at the call site. Requires threading `session.organization_id` through:

- `src/lib/blacklist-client.ts` line 34: `tenantId: 'tlp'`
- `src/lib/crm-client.ts` line 32: `tenantId: 'tlp'`
- `src/lib/crm-client.ts` line 343: `opts.tenantId ?? 'tlp'`
- `src/lib/onboarding-client.ts` line 34: `tenantId: 'tlp'`
- `src/lib/vertical.config.ts` line 10: `tenantId: 'tlp'`
- `src/lib/ppc/engine.ts` line 28: `tenantId: 'tlp'`
- `src/lib/ppc/index.ts` line 12: `tenantId: 'tlp'`
- 20+ API routes with inline `tenant_id: 'tlp'` or `.eq('tenant_id', 'tlp')` (full list: `ask/route.ts`, `ask/vet-action/route.ts`, `ask/entity-add/route.ts`, `ask/outreach-log/route.ts`, `ask/suggestions/route.ts`, `outreach/record-manual-send/route.ts`, `operator/pill/counts/route.ts`, `operator/pill/route.ts`, `entities/aliases/route.ts`, `ask/entity-vet-inline/route.ts`)

**Tier 3 -- Structurally Single-Tenant (deep refactor)**

These have no `tenant_id` column at all in the database:

- `call_records` -- No tenant_id. 30+ query paths assume single tenant.
- `campaigns`, `campaign_routes`, `campaign_qualifiers` -- No tenant_id. Tied to TLP's TrackDrive account.
- `invoices` -- No tenant_id. Invoice workflow (Jen/Tiffani views) assumes single company.
- `alerts` -- No tenant_id. Alert routing hardcodes TLP team members (Fab, Jackie, Jen, etc.)
- `blacklist` (legacy foundation table) -- No tenant_id. Replaced by `blacklist_entries` which has tenant_id.
- `contacts` (foundation) -- No tenant_id. Different from `crm_contacts` which has tenant_id.
- `contract_documents` -- No tenant_id. Has `signing_token` for public access.
- `reconciliation_runs` -- No tenant_id.
- `ai_action_log` -- No tenant_id. Audit trail references TLP-specific entities.
- `milo_conversations`, `milo_knowledge`, `milo_decisions` -- No tenant_id.
- `entity_knowledge` -- No tenant_id. Stores per-entity operational intelligence.
- `evaluation_queue` -- No tenant_id.
- `td_sync_state` -- No tenant_id.
- `user_profiles` -- No org_id / organization_id. Users are not linked to organizations.

The following **already have tenant_id** (mostly from @milo/* packages):
- `vet_results` (TEXT, hardcoded 'tlp')
- `entity_aliases` (TEXT, default 'tlp')
- `outreach_log` (TEXT, default 'tlp')
- `crm_counterparties`, `crm_contacts`, `crm_activities`, `crm_leads` (from @milo/crm)
- `blacklist_entries` (from @milo/blacklist)
- `onboarding_flows`, `onboarding_runs`, `onboarding_steps` (from @milo/onboarding)
- `signing_documents`, `signing_library` (from @milo/contract-signing)
- `tenants` table (exists in shared Supabase, used by milo-outreach)
- `send_log`, `leads`, `outreach_templates`, `opt_outs` (milo-outreach tables)

### Recommended Phasing

#### Phase A -- Foundational (Unblocks Everything)

**Scope:** Organizations table, user-to-org link, tenant_id UUID migration, auth context update, RLS policies.

**Dependencies:** None. This is the root.

**Work items:**
1. Create `organizations` table with UUID PK, slug, display_name, domain_config, created_at, etc.
2. Insert TLP row with a specific UUID. Record the mapping: `'tlp'` string -> TLP UUID.
3. Add `organization_id UUID` to `user_profiles` with FK to `organizations.id`.
4. Backfill all existing user_profiles with `organization_id = TLP_UUID`.
5. Update `handle_new_user()` trigger to accept org context (or default to null for anonymous signups).
6. Add `tenant_id UUID` column to every Tier 3 table listed above (nullable first, then backfill + NOT NULL).
7. Migrate all existing `tenant_id TEXT = 'tlp'` columns to `tenant_id UUID` referencing organizations.id.
8. Update `getEffectiveUser()` in `auth-server.ts` to return `organizationId` from user_profiles.
9. Update middleware to resolve org from subdomain (`{slug}.justmilo.app`) or session.
10. Create invite token table (`org_invites`) for shareable signup links.
11. Wire magic-link flow to associate new users with the org from their invite token.

**Risk areas:**
- The `handle_new_user()` trigger auto-creates user_profiles without org context. New users signing up via anonymous magic link need a path to associate with an org.
- RLS policies are currently "anon_read_all" USING (true) on 15 tables. These must be replaced with tenant-scoped policies.
- Middleware currently resolves auth via Supabase session only. Subdomain-to-org mapping is new logic.

**What this unblocks:** Every subsequent phase depends on `organizations` table + `session.organization_id` being available.

#### Phase B -- Cross-Tenant Shared Layer

**Scope:** Shared entities registry, cross-tenant vet results, blacklist visibility flags, entity aliases.

**Dependencies:** Phase A complete (organizations table exists, tenant_id UUID columns exist).

**Work items:**
1. Create `entities` master table -- one row per real-world entity, canonical facts only.
2. Create `entity_research` table -- versioned vet findings at shared tier, with per-risk-tier freshness cutoff.
3. Modify `blacklist_entries` -- add `reason_visibility` flag (public/private), remove attribution from read paths.
4. Add legal disclaimer infrastructure (see Section 4).
5. Modify `entity_aliases` -- make globally shared (remove tenant_id scoping or add `is_global` flag).
6. Implement conflict resolution for divergent vet verdicts (see Section 11).

**Risk areas:**
- Current `vet_results` table stores everything in one row (facts + recommendation + verdict). Splitting into shared vs tenant-scoped requires restructuring `verdict_data` JSONB.
- `entity_knowledge.hot_knowledge` mixes shared facts with tenant-specific intelligence in a single TEXT field. Requires parsing/splitting strategy.

#### Phase C -- Tool Capability Matrix

**Scope:** `tenant_capabilities` schema, per-tool wiring, tenant onboarding wizard.

**Dependencies:** Phase A complete. Independent of Phase B.

**Work items:**
1. Create `tenant_capabilities` table (see Section 5).
2. Wire each tool's API routes to check capability before executing.
3. Build tenant admin UI for toggling tools.
4. Update crons to iterate active tenants with the relevant capability.

**Risk areas:**
- 15 Vercel crons currently run for TLP only. Each must become tenant-aware (iterate `organizations` where the relevant capability is enabled).
- Cron execution time. Currently ~60s budget per cron. Iterating N tenants may exceed Vercel function limits.

---

## Section 2 -- Existing Multi-Tenant Infrastructure Inventory

### Organizations/Tenants Table

**EXISTS.** The `tenants` table exists in the shared Supabase project (`tappyckcteqgryjniwjg`). Schema from `milo-outreach/lib/types.ts`:

```
tenants:
  id TEXT PK (currently string slugs like 'demo-ppc', 'milo-internal')
  display_name TEXT
  domain TEXT
  from_address TEXT
  reply_to TEXT
  physical_address TEXT
  voice_config JSONB
  research_config JSONB
  real_sends_enabled BOOLEAN
  send_mode TEXT ('manual' | 'auto')
  api_key_hash TEXT
  api_key_prefix TEXT
  active BOOLEAN
  is_demo BOOLEAN
  demo_expires_at TIMESTAMPTZ
  created_at / updated_at
```

**Gap:** Uses TEXT id (string slugs), not UUID. Mark's decision specifies UUID. Migration path: create `organizations` table with UUID PK alongside existing `tenants`, then migrate references.

### User Profiles Relationship to Org

**DOES NOT EXIST.** `user_profiles` has no `organization_id` column. Schema:

```
user_profiles:
  id UUID PK
  user_id UUID FK -> auth.users
  name TEXT
  roles TEXT[]
  team TEXT (always 'operations')
  is_admin BOOLEAN
  setup_complete BOOLEAN
  focus_areas TEXT[]
  ...
```

No FK or column linking a user to a tenant/org. Currently all 11 team members implicitly belong to TLP.

### Team Members Table Scoping

**NOT SCOPED.** The `TEAM_ROSTER` in `team-roster.ts` hardcodes 11 TLP operators. `getCanonicalTeamRoster()` queries `user_profiles` filtered by the hardcoded roster names. No concept of team-per-org.

### Magic-Link Auth -- Org Association

**DOES NOT EXIST.** The middleware (`middleware.ts`) resolves auth via Supabase session cookie only. The `handle_new_user()` trigger creates a bare `user_profiles` row with no org linkage. The `/welcome/[slug]` flow is a one-time claim link per hardcoded team member, not a generic org-aware invite.

### Invite Token / Shareable Link Logic

**DOES NOT EXIST.** No `org_invites` table, no invite token generation, no invite claim flow. The `/welcome/[slug]` system is closest but is hardcoded to the 11 TLP slugs and is one-time-use.

Verification tokens exist (`20260506000004_verification_tokens.sql`) but these are for email verification, not org invites.

### @milo/onboarding DAG -- Org Creation Coverage

**NO.** The onboarding DAG (`@milo/onboarding`) handles publisher onboarding (12-step MSA/W9/IO flow). It does not cover org creation, team invites, or role assignment. The `OnboardingFlow` is tenant-scoped (has `tenant_id` on flows, runs, and steps) but assumes the tenant already exists.

### Existing Role/Permission Concept

**MINIMAL.** Only `is_admin: boolean` on `user_profiles`. Two tiers: admin (Mark, Tiffani) and operator (everyone else). No fine-grained permissions. `focus_areas` array drives which operator pills are visible but is not a permission system -- it is a UI filter only.

---

## Section 3 -- Per-Table Tenant Scoping Recommendation

### Tables WITH Existing tenant_id

| Table | Current tenant_id | Recommendation | Notes |
|---|---|---|---|
| `vet_results` | TEXT 'tlp' | **Split** | Verdict + facts = shared. Recommendation text = tenant-scoped. See Section 4. |
| `entity_aliases` | TEXT 'tlp' | **Shared** | Aliases are universal. Remove tenant scoping. One alias -> one canonical entity globally. |
| `outreach_log` | TEXT 'tlp' | **Tenant-scoped** | Outreach activity is tenant-private. |
| `crm_counterparties` | TEXT (via @milo/crm) | **Tenant-scoped** | Each tenant's CRM is independent. |
| `crm_contacts` | TEXT | **Tenant-scoped** | Same as counterparties. |
| `crm_activities` | TEXT | **Tenant-scoped** | Activity log is tenant-private. |
| `crm_leads` | TEXT | **Tenant-scoped** | Lead pipeline is tenant-private. |
| `blacklist_entries` | TEXT (via @milo/blacklist) | **Split** | See Section 4 for shared blacklist design. |
| `onboarding_flows` | TEXT | **Tenant-scoped** | Each tenant defines their own onboarding flow. |
| `onboarding_runs` | TEXT | **Tenant-scoped** | Onboarding progress is tenant-private. |
| `onboarding_steps` | TEXT | **Tenant-scoped** | Step state is tenant-private. |
| `signing_documents` | TEXT | **Tenant-scoped** | Contracts are tenant-private. |
| `signing_library` | TEXT | **Tenant-scoped** | Document library is tenant-private. |
| `tenants` | is the tenant table | Becomes `organizations` | Migrate to UUID PK. |
| `send_log` | TEXT | **Tenant-scoped** | Email send history is tenant-private. |
| `leads` (outreach) | TEXT | **Tenant-scoped** | Outreach leads are tenant-private. |

### Tables WITHOUT tenant_id (Foundation Schema)

| Table | Recommendation | RLS Feasibility | Gotchas |
|---|---|---|---|
| `call_records` | **Tenant-scoped** | High | Largest table. 30+ query paths in `milo-queries.ts`, `stats`, `reconciliation`, `sync`, `predictions`, `buyer-health`, `publisher-grades`. All must add `.eq('tenant_id', orgId)`. Crons iterate per-tenant. |
| `campaigns` | **Tenant-scoped** | High | Tied to TrackDrive offers. Each tenant's TD account = separate campaigns. |
| `campaign_routes` | **Tenant-scoped** | High | FK to campaigns. Inherits tenant scope. |
| `campaign_qualifiers` | **Tenant-scoped** | High | FK to campaigns. Inherits tenant scope. |
| `invoices` | **Tenant-scoped** | High | Invoice workflow (Jen/Tiffani views) must become per-tenant. Status workflow names are TLP-specific (`pending_jen`, `jen_approved`). |
| `alerts` | **Tenant-scoped** | High | `target_user_id` references TLP users. Alert routing (`alert-routing.ts`) hardcodes TLP team members. Must become per-org. |
| `ai_action_log` | **Tenant-scoped** | Medium | Audit trail. Add tenant_id for scoping. Historical rows get TLP UUID backfill. |
| `contacts` (foundation) | **Tenant-scoped** | Medium | Different from crm_contacts. Used by pipeline, captures, relationships. |
| `prospect_pipeline` | **Tenant-scoped** | High | Already has tenant_id in some code paths (`findOrCreatePipelineEntry` defaults to 'tlp'). Add proper column. |
| `drip_campaigns` | **Tenant-scoped** | High | FK to prospect_pipeline. Inherits tenant scope. |
| `conversation_captures` | **Tenant-scoped** | Medium | Capture screenshots/pastes. Tenant-private. |
| `contract_documents` | **Tenant-scoped** | Medium | Legacy contract table (predates @milo/contract-signing). Some rows have public signing tokens. |
| `blacklist` (legacy) | **Deprecate** | N/A | Replaced by `blacklist_entries`. Legacy table should be migrated and dropped. |
| `reconciliation_runs` | **Tenant-scoped** | High | Per-tenant reconciliation against per-tenant TrackDrive. |
| `relationship_profiles` | **Tenant-scoped** | Low | FK to contacts. Small table. |
| `milo_conversations` | **Tenant-scoped** | Medium | Chat history. Tenant-private. |
| `milo_knowledge` | **Tenant-scoped** | Low | Milo's learned knowledge. Tenant-private. |
| `milo_decisions` | **Tenant-scoped** | Low | Decision tracking. Tenant-private. |
| `entity_knowledge` | **Split** | Medium | `hot_knowledge` mixes shared facts with tenant intelligence. See Section 4. |
| `evaluation_queue` | **Tenant-scoped** | Medium | Knowledge engine queue. Process per-tenant. |
| `td_sync_state` | **Tenant-scoped** | Low | Sync state per TD account = per tenant. |
| `chat_logs` | **Tenant-scoped** | High | /ask query history. Has `ip_hash` for anonymous. Anonymous queries get `tenant_id = NULL`. |
| `chat_logs_archive` | **Tenant-scoped** | Medium | Archive of above. Same treatment. |
| `ops_error_log` | **Tenant-scoped** | Low | Error telemetry. Scope per tenant. |
| `user_profiles` | **Tenant-scoped** (add org_id) | High | Link users to organizations. |
| `eod_reports` | **Tenant-scoped** | Medium | End-of-day reports. Tenant-private. |
| `billing_schedules` | **Tenant-scoped** | Medium | Billing config. Per-tenant. |
| `bank_transactions` | **Tenant-scoped** | Medium | Bank reconciliation. Per-tenant. |
| `milo_learned_patterns` | **Tenant-scoped** | Low | Pattern learning for bank categorization. Per-tenant. |
| `creative_submissions` | **Tenant-scoped** | Low | Publisher creative review. Per-tenant. |
| `support_tickets` | **Tenant-scoped** | Medium | Support system. Per-tenant. |
| `buyer_reports` | **Tenant-scoped** | Medium | Buyer settlement reports. Per-tenant. |
| `buyer_report_rows` | **Tenant-scoped** | Medium | FK to buyer_reports. Inherits scope. |
| `buyer_report_templates` | **Tenant-scoped** | Medium | Column mapping templates. Per-tenant. |
| `disputes` | **Tenant-scoped** | Medium | Billing disputes. Per-tenant. |
| `agreed_terms` | **Tenant-scoped** | Medium | IO terms tracking. Per-tenant. |
| `action_items` | **Tenant-scoped** | Medium | Task system. Per-tenant. |
| `snoozed_items` | **Tenant-scoped** | Low | User-scoped snooze state. |
| `ping_posts` | **Tenant-scoped** | Medium | Ping/post records from TD. Per-tenant. |
| `ping_source_map` | **Tenant-scoped** | Low | TD source mapping. Per-tenant. |
| `milo_activity_log` | **Tenant-scoped** | Low | Activity tracking. Per-tenant. |
| `milo_feedback` | **Tenant-scoped** | Low | Feedback records. Per-tenant. |

### Special Handling -- Entity Tables

**vet_results:** Currently stores everything in one row. For cross-tenant sharing:
- **Shared columns:** entity_name, entity_name_normalized, entity_type, verdict, and within verdict_data: facts array, verdict_summary, flag categories.
- **Tenant-scoped columns:** recommendation text, assigned_to, created_by.
- **Implementation:** Keep vet_results as-is (tenant-scoped). Create separate `entity_research` shared table that extracts canonical facts from verdict_data on vet completion.

**entity_knowledge:** Currently a single row per entity with hot/warm/cold knowledge tiers.
- hot_knowledge contains both universal facts ("company founded 2018, based in Miami") and tenant intelligence ("our IO with them is $225 CPA, last payment was late").
- **Implementation:** Split into `entity_knowledge` (tenant-scoped, operational intelligence) and contribute canonical facts to the shared `entities` + `entity_research` tables.
- **Gotcha:** The knowledge engine cron (`/api/cron/evaluate`) processes events and rewrites hot_knowledge. Must scope event processing per tenant.

**prospect_pipeline:** Purely tenant-scoped. Each tenant has their own pipeline. The `findOrCreatePipelineEntry()` function in crm-client.ts already passes tenantId (defaulting to 'tlp').

---

## Section 4 -- Cross-Tenant Shared Layer Schema Proposal

### `entities` -- Master Entity Registry

```sql
CREATE TABLE entities (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  canonical_name TEXT NOT NULL,
  canonical_name_normalized TEXT NOT NULL,
  entity_type TEXT NOT NULL CHECK (entity_type IN ('publisher', 'buyer', 'affiliate', 'network', 'individual', 'company')),
  website TEXT,
  linkedin_url TEXT,
  industry TEXT,
  location TEXT,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE(canonical_name_normalized, entity_type)
);

CREATE INDEX idx_entities_name ON entities (canonical_name_normalized);
CREATE INDEX idx_entities_type ON entities (entity_type);
```

**Write path:** Created/updated when any tenant vets an entity, adds to CRM, or creates a pipeline entry.
**Read path:** Queried during vet dedup, /ask entity lookup, cross-tenant entity resolution.
**Conflict resolution:** UNIQUE on (normalized_name, entity_type). First vet creates the row. Subsequent vets update facts via entity_research.

### `entity_research` -- Shared Vet Findings

```sql
CREATE TABLE entity_research (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  entity_id UUID NOT NULL REFERENCES entities(id),
  researched_by_tenant_id UUID REFERENCES organizations(id),  -- NULL for anonymous /ask
  verdict TEXT NOT NULL CHECK (verdict IN ('blacklisted', 'likely_scam', 'likely_real', 'unclear')),
  verdict_summary TEXT NOT NULL,
  facts JSONB NOT NULL DEFAULT '[]',        -- [{label, value}] canonical facts
  flag_categories JSONB NOT NULL DEFAULT '[]', -- [{severity, title}] without tenant detail
  freshness_tier TEXT NOT NULL DEFAULT 'standard',  -- risk-tier key for cutoff lookup
  researched_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  research_method TEXT DEFAULT 'web_research',
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_entity_research_entity ON entity_research (entity_id, researched_at DESC);
CREATE INDEX idx_entity_research_verdict ON entity_research (verdict);
```

**Per-risk-tier freshness cutoff schema:**

```sql
CREATE TABLE entity_freshness_config (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  verdict TEXT NOT NULL,
  freshness_days INTEGER NOT NULL,
  description TEXT,
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE(verdict)
);

-- Default configuration per Mark's spec:
INSERT INTO entity_freshness_config (verdict, freshness_days, description) VALUES
  ('likely_real', 5, 'Re-vet after 5 days'),
  ('unclear', 2, 'Re-vet after 2 days'),
  ('likely_scam', 0, 'Always re-vet immediately'),
  ('blacklisted', 0, 'Always re-vet immediately');
```

**Read path for freshness check:**
```sql
SELECT er.* FROM entity_research er
JOIN entity_freshness_config efc ON efc.verdict = er.verdict
WHERE er.entity_id = $entity_id
  AND er.researched_at > now() - (efc.freshness_days || ' days')::interval
ORDER BY er.researched_at DESC
LIMIT 1;
```

If this returns NULL, the entity needs re-vetting. If it returns a row, use the cached result.

**Conflict resolution -- two tenants vet same entity with different verdicts:**
The `entity_research` table stores ALL vet results (one row per vet action). The most recent row is canonical for freshness purposes. If Tenant A says "likely_real" and Tenant B says "likely_scam" within the cutoff window, the LATER verdict wins for the shared layer. Both rows are preserved for audit. Recommend surfacing "Verdict: likely_scam (conflicting assessments exist)" in the shared read path so consumers know to exercise caution.

### `blacklist_entries` -- Cross-Tenant Blacklist

The existing `blacklist_entries` table (from @milo/blacklist) already has the right structure. Additions needed:

```sql
ALTER TABLE blacklist_entries
  ADD COLUMN IF NOT EXISTS reason_visibility TEXT NOT NULL DEFAULT 'private'
    CHECK (reason_visibility IN ('public', 'private')),
  ADD COLUMN IF NOT EXISTS legal_disclaimer_accepted BOOLEAN NOT NULL DEFAULT false,
  ADD COLUMN IF NOT EXISTS legal_disclaimer_accepted_at TIMESTAMPTZ;
```

**Read path (cross-tenant):**
```sql
-- Any tenant checking if an entity is blacklisted:
SELECT
  company_name,
  CASE WHEN reason_visibility = 'public' THEN reason
       ELSE 'private'
  END AS displayed_reason,
  severity,
  is_active
FROM blacklist_entries
WHERE company_name ILIKE $entity_name
  AND is_active = true;
-- NO tenant_id in WHERE -- reads across all tenants.
-- NO attribution columns returned -- no "who blacklisted" information.
```

**Write path (blacklist submission):**
```sql
INSERT INTO blacklist_entries (tenant_id, company_name, reason, reason_visibility, ...)
VALUES ($submitting_tenant_uuid, ...);
```

**Soft warning implementation:** When Tenant B engages with a blacklisted entity:
- Display: "This entity is on the blacklist. Reason: [public text or 'private']"
- Do NOT block the action. Log the engagement to `ai_action_log` with `action_type = 'blacklist_override_engagement'` for audit trail.

### Legal Disclaimer Recommendation

**Research finding:** Platforms sharing user-contributed compliance data have precedent liability under Section 230 of the Communications Decency Act (US), but this protection has limits when the platform curates/endorses the content. The safest pattern used by compliance platforms (e.g., TCPA Litigator List, DNC registries, industry blacklists):

**Recommended Terms of Service clause:**

> **User-Contributed Compliance Data.** When you submit blacklist entries, reasons, or compliance flags ("User Compliance Data"), you represent and warrant that: (a) the information is truthful and based on your direct business experience; (b) you have a good-faith basis for the claims made; (c) the submission does not constitute defamation, tortious interference, or unfair business practice under applicable law. Milo provides this data as-is, without endorsement, verification, or guarantee of accuracy. Milo is not liable for business decisions made based on User Compliance Data contributed by other users. You agree to indemnify Milo against claims arising from your submissions.

**Where this disclaimer surfaces:**
1. **Blacklist submission form** -- checkbox: "I confirm this information is truthful and based on my direct business experience" (required before submission).
2. **ToS acceptance** -- at tenant signup (Phase A).
3. **Blacklist read view** -- footer text: "Blacklist data is contributed by network participants and not verified by Milo."

**Recommendation: reasons should default to 'private' with opt-in to public.** This is the most protective stance. Tenants must affirmatively check "make reason visible to other network members" when submitting. The legal disclaimer checkbox is required regardless of visibility choice.

### `entity_aliases` -- Globally Shared

```sql
-- Migration: remove tenant scoping
ALTER TABLE entity_aliases DROP CONSTRAINT entity_aliases_alias_normalized_tenant_id_key;
ALTER TABLE entity_aliases ADD CONSTRAINT entity_aliases_alias_normalized_key UNIQUE (alias_normalized);
-- Or: add is_global boolean, keep tenant_id for tenant-specific aliases
```

**Recommended approach:** Make aliases global. If two tenants provide conflicting aliases (e.g., Tenant A: "NLD" = "NextLevel Direct", Tenant B: "NLD" = "National Lead Direct"), the FIRST alias wins. The second tenant sees a disambiguation prompt: "NLD is already mapped to NextLevel Direct. Create a tenant-specific override?" Tenant-specific overrides stored in a separate `tenant_entity_alias_overrides` table.

---

## Section 5 -- Tool Inventory + Capability Matrix

### Proposed Coarse Tool Groupings (10 tools)

#### Tool 1: Core Operations Dashboard
- **Description:** Command center, stat bar, morning briefing, predictions, team status
- **Tables owned:** N/A (reads from call_records, alerts, invoices)
- **Crons:** `/api/cron/morning-assignments`, `/api/cron/health-check`
- **Surfaces:** /, /operator, StatBar, BriefingPanel, MorningBriefing
- **Integrations:** TrackDrive (read), Teams (write)
- **Independent:** No -- requires Call Sync at minimum
- **TLP migration impact:** None (already exists)

#### Tool 2: Call Sync + TrackDrive Integration
- **Description:** Pulls calls, campaigns, pings from TrackDrive. Reconciliation. ConvoQC enrichment.
- **Tables owned:** call_records, campaigns, campaign_routes, campaign_qualifiers, ping_posts, ping_source_map, reconciliation_runs, td_sync_state
- **Crons:** `/api/cron/sync` (every 15m), `/api/cron/reconciliation` (daily), `/api/cron/td-change-detection` (every 15m)
- **Surfaces:** /campaigns, /pings, /ping-post, /reconciliation
- **Integrations:** TrackDrive API (per-tenant credentials), ConvoQC API
- **Independent:** Yes (but most other tools depend on this)
- **TLP migration impact:** Requires per-tenant TD credentials

#### Tool 3: Billing + Invoicing
- **Description:** Invoice generation, approval workflow, aging, payment tracking, bank reconciliation
- **Tables owned:** invoices, billing_schedules, bank_transactions, milo_learned_patterns, buyer_reports, buyer_report_rows, buyer_report_templates, disputes
- **Crons:** `/api/cron/billing-prepare` (daily weekdays), `/api/cron/detect-disputes` (hourly)
- **Surfaces:** /invoices, /jen, /tiffani, /reports, /disputes
- **Integrations:** None external
- **Independent:** Yes (needs call data for invoice generation)
- **TLP migration impact:** Invoice status names reference TLP people (pending_jen, jen_approved). Must be generalized.

#### Tool 4: Entity Vetting + Research (Public /ask)
- **Description:** Entity vetting, web research, entity verification, structured response cards, spin-to-reveal
- **Tables owned:** vet_results, chat_logs, chat_logs_archive, ops_error_log
- **Crons:** `/api/cron/error-digest` (daily)
- **Surfaces:** /ask, EntityVetCard, StructuredResponseCard, TriageDrawer
- **Integrations:** Anthropic Claude, web research
- **Independent:** Yes (standalone public surface + authenticated vet)
- **TLP migration impact:** None -- already supports anonymous + tenant

#### Tool 5: CRM + Pipeline
- **Description:** Counterparty management, contacts, pipeline stages, relationship profiles
- **Tables owned:** crm_counterparties, crm_contacts, crm_activities, crm_leads, prospect_pipeline, drip_campaigns, contacts (foundation), relationship_profiles
- **Crons:** `/api/cron/stale-relationships` (daily)
- **Surfaces:** /pipeline, /pipeline-board, /partners, EntityDetailPanel, DrillDownDrawer
- **Integrations:** @milo/crm
- **Independent:** Yes
- **TLP migration impact:** Already has tenant_id in CRM tables

#### Tool 6: Contracts + Signing
- **Description:** Contract analysis, redlining, digital signing, document library, creative review
- **Tables owned:** contract_documents, signing_documents, signing_library, analysis_records, creative_submissions, agreed_terms
- **Crons:** `/api/cron/stale-waiting` (implied from route existence)
- **Surfaces:** /contract-queue, /contract-review/[id], ContractAnalysisDisplay, InlineRedlineEditor
- **Integrations:** @milo/contract-analysis, @milo/contract-signing, @milo/contract-negotiation, Resend (signing emails)
- **Independent:** Yes
- **TLP migration impact:** Already has tenant_id in signing tables

#### Tool 7: Outreach + Messaging
- **Description:** Publisher/buyer recruitment, email sends, LinkedIn drafts, reply classification
- **Tables owned:** outreach_log, send_log, leads (outreach), outreach_templates, opt_outs
- **Crons:** `/api/cron/linkedin-draft` (weekly), `/api/cron/publisher-discovery` (daily)
- **Surfaces:** Outreach drawers within /operator
- **Integrations:** milo-outreach service, Resend, Cloudflare email worker
- **Independent:** Yes
- **TLP migration impact:** milo-outreach already multi-tenant

#### Tool 8: Knowledge Engine + AI Operations
- **Description:** Entity knowledge profiles, event evaluation, anomaly detection, pattern recognition
- **Tables owned:** entity_knowledge, evaluation_queue, milo_knowledge, milo_decisions, milo_conversations
- **Crons:** `/api/cron/evaluate` (every 5m), `/api/cron/mediarite-xref` (daily)
- **Surfaces:** Observatory (/observatory)
- **Integrations:** Anthropic Claude (Sonnet for evaluation)
- **Independent:** Requires Call Sync for event feed
- **TLP migration impact:** No tenant_id on any owned table. Full backfill needed.

#### Tool 9: Alert System + Quality Enforcement
- **Description:** Automated alerts, quality enforcement, publisher grades, buyer health scores
- **Tables owned:** alerts, action_items, snoozed_items
- **Crons:** Alert generation runs as step 5 of sync orchestrator
- **Surfaces:** Alerts drawer, notification count badge
- **Integrations:** TrackDrive (publisher pause/unpause -- Decision 175.1: mutations forbidden for now)
- **Independent:** Requires Call Sync
- **TLP migration impact:** Alert routing hardcodes TLP team members

#### Tool 10: Publisher Onboarding
- **Description:** 12-step DAG onboarding flow for new publishers
- **Tables owned:** onboarding_flows, onboarding_runs, onboarding_steps
- **Crons:** None (event-driven)
- **Surfaces:** /onboard, /onboard/publisher/[token]
- **Integrations:** @milo/onboarding, @milo/contract-signing
- **Independent:** Yes
- **TLP migration impact:** Already has tenant_id

### `tenant_capabilities` Schema

```sql
CREATE TABLE tenant_capabilities (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  organization_id UUID NOT NULL REFERENCES organizations(id),
  tool_key TEXT NOT NULL,  -- matches tool keys below
  enabled BOOLEAN NOT NULL DEFAULT false,
  enabled_at TIMESTAMPTZ,
  disabled_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE(organization_id, tool_key)
);

CREATE INDEX idx_tenant_caps_org ON tenant_capabilities (organization_id);
CREATE INDEX idx_tenant_caps_tool ON tenant_capabilities (tool_key, enabled) WHERE enabled = true;
```

Tool keys: `co[REDACTED_KEY]`, `call_sync`, `billing`, `entity_vetting`, `crm_pipeline`, `contracts_signing`, `outreach`, `knowledge_engine`, `alerts_quality`, `publisher_onboarding`.

---

## Section 6 -- Onboarding Flow Infrastructure Audit

### Existing Signup Surface

- `/login` -- Magic-link login page. Sends Supabase magic link to email.
- `/onboard` -- Cinematic onboarding sequence (shown to users with `has_completed_onboarding = false`).
- `/welcome/[slug]` -- One-time-use claim link per team member. Hardcoded to 11 TLP slugs.
- `/setup` -- Setup page for users with `setup_complete = false`.

**Gap:** No self-service tenant signup. No "create organization" step. All users are assumed to be TLP operators.

### @milo/onboarding DAG -- Tenant Creation

**DOES NOT COVER.** The onboarding DAG is exclusively for publisher onboarding (MSA > W9 > IO > Go Live). There is no flow for:
- Organization creation
- Team invite
- Role assignment
- Tool selection
- Integration credential setup

**Recommendation:** Build a separate `TenantOnboardingWizard` component (not a DAG flow) that covers:
1. Create organization (name, slug, domain)
2. Admin user association
3. Tool selection (binary on/off toggles)
4. Integration credential input (TrackDrive, Resend, Teams)
5. First team invite

### Whether Infrastructure Precludes Future Sub-Users

**No structural blocker.** The `user_profiles.organization_id` FK allows linking multiple users to one org. Adding a future `user_profiles.role_within_org` or a separate `org_members` junction table with scoped permissions is straightforward. The existing `is_admin` boolean plus `focus_areas` array is a rough version of role-based access that can be evolved.

**Caveat:** The middleware impersonation system (`milo_impersonating` cookie) assumes admin-to-non-admin impersonation within a single org. Cross-org impersonation would require additional guards.

---

## Section 7 -- Per-Tenant Integration Credentials

### TrackDrive

**Current state:** Single TLP account. Credentials in env vars: `TD_API_BASE_URL`, `TD_API_AUTH` (HTTP Basic). Hardcoded in `src/lib/trackdrive.ts` lines 34-35.

**Per-tenant schema:**

```sql
CREATE TABLE tenant_integrations (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  organization_id UUID NOT NULL REFERENCES organizations(id),
  integration_key TEXT NOT NULL,  -- 'trackdrive', 'resend', 'teams', 'convoqc'
  credentials_encrypted BYTEA NOT NULL,  -- encrypted at rest
  credentials_iv BYTEA NOT NULL,         -- initialization vector
  config JSONB DEFAULT '{}',             -- non-secret config (base_url, etc.)
  status TEXT NOT NULL DEFAULT 'active' CHECK (status IN ('active', 'inactive', 'error')),
  last_verified_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE(organization_id, integration_key)
);
```

**Encryption:** Use Node.js `crypto.createCipheriv('aes-256-gcm', ...)` with a master key from env var `INTEGRATION_ENCRYPTION_KEY`. Decrypt at runtime when creating integration clients.

**Rotation:** Add `rotated_at TIMESTAMPTZ` column. Admin UI allows updating credentials. Old credentials are overwritten (not versioned -- no reason to keep old TD auth).

### Resend

**Current state:** Single `RESEND_API_KEY` env var. Used by @milo/contract-signing (signing emails) and milo-outreach (outreach emails).

**Per-tenant:** Each tenant needs their own sending domain (SPF/DKIM/DMARC). Resend supports domain verification via API. Schema: store `resend_domain_id` and `resend_api_key` in `tenant_integrations`. The `from_address` and `reply_to` are already per-tenant in the `tenants` table.

**DNS requirements per tenant:** Each tenant must add:
- MX record for their domain
- SPF record including Resend
- DKIM records (provided by Resend domain verification)
- Optional: DMARC record

### Teams (Microsoft)

**Current state:** Sends via PPCRM Supabase Edge Function relay. Channel webhooks are hardcoded in the edge function.

**Per-tenant:** Two options:
1. **Webhook per channel per tenant:** Store Teams incoming webhook URLs in `tenant_integrations.config`. Each tenant provides their own webhooks. Simple but limited (no DMs).
2. **M365 OAuth per tenant:** Full Teams integration with app-level permissions. Allows DMs, adaptive cards, channel management. More complex setup.

**Recommendation:** Start with option 1 (webhook URLs). Graduate to M365 OAuth only if tenants need DMs.

### Telegram

**Deferred per Mark's decision.** When implemented: per-tenant bot token stored in `tenant_integrations`. Each tenant creates their own Telegram bot via @BotFather.

### Supabase

**Recommendation: Single project + RLS.** Reasons:
- All @milo/* packages already use a single Supabase project (tappyckcteqgryjniwjg).
- Milo-outreach uses the same project.
- Schema migrations are atomic across all tenants.
- RLS is the native Supabase multi-tenancy pattern.
- Cross-tenant shared layer (entity research, blacklist) requires same-project access.

**Flag for potential separation:** If a tenant has regulatory requirements (data residency, HIPAA), a separate Supabase project may be needed. This is not the v1 design -- flag it as a future consideration.

---

## Section 8 -- Auth Model + Resilience Track 2 Reconciliation

### Per-Route Auth Tier Recommendation

| Route Pattern | Current Auth | Recommended Tier |
|---|---|---|
| `/api/ask` (POST) | None | **Public anonymous** -- tenant_id from session if logged in, NULL if not |
| `/api/ask/rating` | None | **Public anonymous** |
| `/api/ask/explain-further` | None | **Public anonymous** |
| `/api/ask/classify-file` | None | **Public anonymous** |
| `/api/ask/vet-action` | None | **Authenticated tenant-scoped** (writes to CRM/pipeline) |
| `/api/ask/entity-add` | None | **Authenticated tenant-scoped** |
| `/api/ask/entity-check` | None | **Authenticated cross-tenant** (checks shared entities) |
| `/api/ask/suggestions` | None | **Authenticated tenant-scoped** |
| `/api/ask/outreach-log` | None | **Authenticated tenant-scoped** |
| `/api/blacklist` | None | **Authenticated cross-tenant** (read), **Authenticated tenant-scoped** (write) |
| `/api/blacklist/screen` | None | **Authenticated cross-tenant** (reads across tenants) |
| `/api/alerts` | None | **Authenticated tenant-scoped** |
| `/api/operator/*` | session check | **Authenticated tenant-scoped** |
| `/api/admin/*` | is_admin check | **Admin-only** |
| `/api/cron/*` | CRON_SECRET | **System (cron secret)** |
| `/api/sync/*` | None | **System or Admin-only** (sensitive -- triggers TD sync) |
| `/api/invoices/*` | None | **Authenticated tenant-scoped** |
| `/api/stats` | None | **Authenticated tenant-scoped** |
| `/api/briefing/*` | None | **Authenticated tenant-scoped** |
| `/api/live-stats/*` | API key | **API key authenticated** (cross-project) |
| `/api/demo/*` | demo session | **Public anonymous** (demo tenant) |
| `/api/feedback/record` | None | **Public anonymous** |
| `/api/verify/*` | token | **Token-authenticated** (contract signing) |
| `/api/webhooks/resend` | webhook secret | **Webhook secret** |

### Alerts Drawer 401 Case

**Current problem:** The alerts drawer loads on every page for all users, including the public /ask surface. When not authenticated, the alert fetch returns 401.

**Right fix:** Alerts are only visible when logged in to a tenant. The component should check `useAuth().user` before fetching. The API route should return 401 (current behavior is correct) -- the fix is on the client side to not fetch when unauthenticated.

### Track 2 (Auth Removal) Reconciliation

Track 2 originally proposed removing auth gates from certain routes. With multi-tenant, the guidance reverses:

**When Track 2 resumes:**
- Routes that were candidates for auth removal should instead get **three-tier auth**: public anonymous (no session), authenticated cross-tenant (session exists, reads shared data), authenticated tenant-scoped (session exists + org, reads/writes tenant data).
- The /ask surface stays public anonymous for read-only operations.
- Any route that writes tenant data MUST require authenticated tenant-scoped session.
- The resilience improvement from Track 2 (no 401 breaks on public surfaces) is achieved by making public surfaces genuinely work without auth, not by removing auth from tenant surfaces.

---

## Section 9 -- Migration Shape

### TLP Becomes Tenant 1

**Step 1: Create organizations table + TLP row**
```sql
CREATE TABLE organizations (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  slug TEXT NOT NULL UNIQUE,
  display_name TEXT NOT NULL,
  legal_name TEXT,
  domain TEXT,
  logo_url TEXT,
  settings JSONB DEFAULT '{}',
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

INSERT INTO organizations (id, slug, display_name, legal_name, domain)
VALUES ('aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee', 'tlp', 'The Lead Penguin', 'The Lead Penguin LLC', 'tlp.justmilo.app');
```

**Step 2: Add organization_id to user_profiles**
```sql
ALTER TABLE user_profiles ADD COLUMN organization_id UUID REFERENCES organizations(id);
UPDATE user_profiles SET organization_id = 'aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee';
ALTER TABLE user_profiles ALTER COLUMN organization_id SET NOT NULL;
```

**Step 3: Add tenant_id UUID to Tier 3 tables**
For each table without tenant_id: add column, backfill with TLP UUID, set NOT NULL.
Tables: call_records, campaigns, campaign_routes, campaign_qualifiers, invoices, alerts, ai_action_log, contacts (foundation), contract_documents, reconciliation_runs, milo_conversations, milo_knowledge, milo_decisions, entity_knowledge, evaluation_queue, td_sync_state, chat_logs, chat_logs_archive, ops_error_log, eod_reports, billing_schedules, bank_transactions, milo_learned_patterns, creative_submissions, support_tickets, buyer_reports, buyer_report_rows, buyer_report_templates, disputes, agreed_terms, action_items, snoozed_items, ping_posts, ping_source_map, relationship_profiles, milo_activity_log, milo_feedback.

**Step 4: Migrate TEXT tenant_id to UUID**
For tables with `tenant_id TEXT = 'tlp'`: add new `tenant_id_uuid UUID` column, backfill, drop old column, rename new column.
Tables: vet_results, entity_aliases, outreach_log, plus all @milo/* tables (crm_counterparties, crm_contacts, crm_activities, crm_leads, blacklist_entries, onboarding_flows, onboarding_runs, onboarding_steps, signing_documents, signing_library).

**Step 5: Wire RLS policies**
Replace all `anon_read_all USING (true)` policies with:
```sql
CREATE POLICY "tenant_isolation" ON <table>
  FOR ALL
  USING (tenant_id = (SELECT organization_id FROM user_profiles WHERE user_id = auth.uid()));
```

**Step 6: Update crons to iterate tenants**
Each cron route gets:
```typescript
const { data: orgs } = await supabaseAdmin
  .from('organizations')
  .select('id')
  .eq('active', true);
for (const org of orgs) {
  await processForTenant(org.id);
}
```

**Step 7: Update query paths**
Every `supabaseAdmin.from('table').select(...)` query that currently has no tenant filter gets `.eq('tenant_id', orgId)`.

### Scope Estimate

| Category | Count |
|---|---|
| Tables needing new tenant_id column | ~37 |
| Tables needing TEXT -> UUID migration | ~14 |
| API route files with hardcoded 'tlp' | ~12 |
| API route files needing tenant scoping | ~130+ |
| Cron routes needing tenant iteration | 15 |
| Lib files with hardcoded 'tlp' | ~8 |
| RLS policies to replace | 15 |
| @milo/* packages (already tenant-aware) | 7 |

### Rollback Plan

1. All migrations use `IF NOT EXISTS` / `ADD COLUMN IF NOT EXISTS`.
2. New UUID columns are added alongside existing TEXT columns (no drop until verified).
3. RLS policies are added with `CREATE POLICY IF NOT EXISTS` -- old policies can coexist.
4. Code changes behind feature flag: `if (process.env.MULTI_TENANT === 'true')` for the org resolution path, falling back to TLP UUID.
5. If migration fails: revert code, old TEXT columns still work, old RLS policies still work.

---

## Section 10 -- Frozen-Repo Prior Art

### MOP (milo-ops, frozen)

`~/Documents/GitHub/milo-ops/` -- **does not exist** at this path. Cannot audit.

### PPCRM

Not accessible at a known path. Based on SYSTEMS.md references: PPCRM uses a separate Supabase project (`jdzqkaxmnqbboqefjolf`) and communicates with milo-for-ppc only via the Teams messaging Edge Function relay. No multi-tenant infrastructure to port.

### mysecretary

`~/mysecretary/` exists but contains only a `.next` directory (build output, no source). Based on conductor-mark docs: mysecretary uses Supabase project `nuequnfpnvbryvhuyuvz`. The `DOMAIN-STRINGS.md` catalog shows 139 domain-specific strings across 5 categories, but no multi-tenant infrastructure.

**Verdict:** No reusable multi-tenant infrastructure in frozen repos. The milo-outreach tenant model is the only existing prior art worth extending.

---

## Section 11 -- Risk Areas + Open Questions for Mark

### Questions Needing Mark's Decision

1. **Conflict resolution for divergent vet verdicts:** Proposed: latest verdict wins for shared layer, with "conflicting assessments exist" flag. Alternative: lower-trust verdict always wins (conservative). Which approach?

2. **Blacklist expiration:** Should blacklist entries expire? Proposed: `expires_at` column (already exists on `blacklist_entries`). TCPA violations might have a 4-year statute of limitations. Fraud entries might be permanent. Should expiration be per-severity (permanent = never, temporary = configurable, watch = 90 days)?

3. **Anonymous /ask contributors:** Currently anonymous /ask queries contribute to `chat_logs` with `ip_hash` but no tenant association. For the shared layer: should anonymous vet results feed into `entity_research`? If yes, they would be attributed to `researched_by_tenant_id = NULL` (no attribution). If no, only authenticated tenants contribute to the shared layer.

4. **Entity aliases disambiguation:** Proposed: first alias wins globally, subsequent conflicting aliases get tenant-specific override. Alternative: all aliases are tenant-scoped (no global sharing). Which approach?

5. **Role permissions across tools:** Currently `is_admin` + `focus_areas`. For multi-tenant: should role definitions be per-org (each tenant defines their own roles) or global (Milo defines standard roles)? Recommended: global role templates with per-org overrides.

6. **Invoice workflow naming:** Current statuses reference TLP people: `pending_jen`, `jen_approved`, `jen_flagged`. For multi-tenant these must be generalized. Proposed: `pending_reviewer`, `reviewer_approved`, `reviewer_flagged`. The actual reviewer identity comes from the assigned user, not the status name. Confirm this is acceptable.

7. **TrackDrive web base URL:** Currently hardcoded as `https://the-lead-penguin.trackdrive.com` in briefing/action/route.ts for deep links. Each tenant with TD integration has a different subdomain. Store in `tenant_integrations.config.web_base_url`.

8. **Single Supabase project confirmation:** Recommendation is single project + RLS for v1. Are there any known tenants with data residency or regulatory requirements that would force project separation?

9. **Tenant self-service vs admin-provisioned:** Should new tenants be able to self-service signup (create org, configure tools), or should Mark/admin provision each tenant manually? Recommendation: admin-provisioned in v1, self-service in v2.

10. **Cross-tenant entity_knowledge sharing:** The knowledge engine's `hot_knowledge` contains both universal facts and tenant-specific operational intelligence. The proposed split (shared `entities` + `entity_research` for facts, tenant-scoped `entity_knowledge` for intelligence) requires the knowledge engine cron to write to both tiers. Confirm this approach.

### Risks Identified

- **Cron execution budget:** 15 crons currently run for 1 tenant. With N tenants, the every-5-minute knowledge evaluation cron could exceed Vercel's 60s function limit. May need to queue tenants across invocations rather than processing all in one call.

- **Invoice status names:** `pending_jen`, `jen_approved` are embedded in the database CHECK constraint, the invoice generation code, and the Jen/Tiffani view pages. Renaming requires a coordinated migration + code update.

- **Teams messaging relay:** Currently routes through PPCRM's Supabase Edge Function. Each new tenant needs their own Teams webhook or OAuth app. The relay pattern may not scale -- consider a Milo-owned Teams integration service.

- **`supabaseAdmin` bypasses RLS:** All API routes use `getSupabaseAdmin()` which uses the service role key. RLS policies only protect against anon/authenticated client access. Server-side tenant isolation depends on code correctly filtering by tenant_id, not RLS. This is the correct pattern for Next.js API routes but means RLS is a safety net, not the primary isolation mechanism.

- **Legacy `blacklist` table:** The foundation schema `blacklist` table (no tenant_id) coexists with `blacklist_entries` (from @milo/blacklist, has tenant_id). Code references both. The legacy table must be migrated to `blacklist_entries` and dropped as part of Phase A cleanup.

---

Report at: https://raw.githubusercontent.com/moscaleflow/conductor-mark-public/main/reports/multi-tenant-architecture-audit-2026-05-06.md
