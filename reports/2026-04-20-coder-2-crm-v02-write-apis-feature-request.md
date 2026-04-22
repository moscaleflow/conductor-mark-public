# Feature Request: @milo/crm Consumer Migration Gap Analysis

**Date**: 2026-04-20
**Lane**: Coder-2 (Migration)
**Task**: Document E3 blocker for Coder-1 prioritization
**Status**: Analysis complete — blocker is schema mapping, not missing APIs

---

## Key Finding: Write APIs Already Exist

**@milo/crm v0.1.x already exports full CRUD:**

| Function | Table | Operation |
|----------|-------|-----------|
| `createCounterparty(client, input, performedBy?)` | crm_counterparties | INSERT |
| `updateCounterparty(client, id, updates, performedBy?)` | crm_counterparties | UPDATE |
| `archiveCounterparty(client, id, performedBy?)` | crm_counterparties | soft-archive |
| `createLead(client, input, performedBy?)` | crm_leads | INSERT |
| `updateLead(client, id, updates, performedBy?)` | crm_leads | UPDATE |
| `promoteLeadToCounterparty(client, leadId, type, performedBy?)` | crm_leads → crm_counterparties | conversion |
| `createContact(client, input, performedBy?)` | crm_contacts | INSERT |
| `updateContact(client, id, updates)` | crm_contacts | UPDATE |
| `deleteContact(client, id)` | crm_contacts | DELETE |
| `logActivity(client, input)` | crm_activities | INSERT |

All functions include tenant isolation (`tenant_id` scoping) and activity logging.

**The wave plan's "blocked on @milo/crm v0.2" was incorrect.** The actual blocker is a structural schema mismatch between milo-ops's local tables and @milo/crm's relational model.

---

## The Real Gap: Schema Mapping

### milo-ops `contacts` (flat) vs @milo/crm `crm_contacts` (relational)

| milo-ops `contacts` column | @milo/crm `crm_contacts` equivalent | Gap |
|---------------------------|-------------------------------------|-----|
| `full_name` | `name` | Column rename |
| `email` | `email` | Match |
| `phone` | `phone` | Match |
| `role` | `role` | Match |
| `linkedin_url` | `linkedin_url` | Match |
| `company_name` | *(none — derived from parent counterparty/lead)* | **Structural** |
| `entity_type` | *(none — derived from counterparty.counterparty_type)* | **Structural** |
| `timezone` | *(none)* | **Missing** |
| `preferred_channel` | *(none)* | **Missing** |
| `website` | *(none — on Lead/Counterparty)* | Moved to parent |
| `facebook_url` | *(none)* | **Missing** |
| `twitter_url` | *(none)* | **Missing** |
| `industry` | *(none — on Lead)* | Moved to parent |
| `company_size` | *(none — on Lead)* | Moved to parent |
| `research_status` | *(none — on Lead)* | Moved to Lead |
| `personal_notes` (JSONB) | `notes` (text) | **Type mismatch** |
| *(none)* | `counterparty_id` (FK) | Contact must link to parent |
| *(none)* | `lead_id` (FK) | Contact must link to parent |
| *(none)* | `is_primary` | Missing in local |
| *(none)* | `tags` | Missing in local |
| *(none)* | `source` | Missing in local |

### Structural Problem

milo-ops `contacts` is a flat table: each contact carries its own `company_name` and `entity_type`. `crm_contacts` is relational: contacts belong to a counterparty or lead, and company info lives on the parent entity.

Migrating contact writes means:
1. Ensure a `crm_counterparties` or `crm_leads` record exists for the company
2. Create the `crm_contacts` record linked to that parent via `counterparty_id` or `lead_id`
3. Move company-level fields (website, industry, company_size) to the parent entity
4. Map `entity_type` to `counterparty_type`

This is a non-trivial structural migration — not just swapping imports.

---

## What milo-ops Writes Today (by table)

### `contacts` — 10 write sites, ~14 operations

| # | File | Operation | Columns | Trigger |
|---|------|-----------|---------|---------|
| 1 | `src/app/api/contacts/route.ts:29` | INSERT | full_name, email, phone, company_name, entity_type | User creates contact |
| 2 | `src/app/api/contacts/[id]/route.ts:36` | UPDATE | full_name, email, phone, role, linkedin_url, preferred_channel, website, facebook_url, twitter_url, industry, company_size | Profile edit |
| 3 | `src/app/api/leads/import/route.ts:350` | INSERT | full_name, email, phone, company_name, entity_type | CSV bulk import |
| 4 | `src/lib/publisher-onboarding.ts:82` | INSERT | full_name, email, role='publisher_contact', company_name, entity_type='publisher', timezone | Publisher onboarding |
| 5 | `src/lib/tools/setup-new-publisher.ts:97` | INSERT | full_name, email, phone, role='publisher_contact', company_name, entity_type='publisher', timezone | AI tool |
| 6 | `src/lib/tools/setup-new-buyer.ts:90` | INSERT | full_name, email, phone, role='buyer_contact', company_name, entity_type='buyer', timezone | AI tool |
| 7 | `src/app/api/capture/route.ts:470` | UPDATE | phone, email (enrichment) | Conversation capture |
| 8 | `src/app/api/capture/route.ts:496` | INSERT | full_name, company_name, role, phone, email, entity_type='unknown' | Auto-create from conversation |
| 9 | `src/lib/web-research.ts:23,55` | UPDATE | research_status, website, linkedin_url, facebook_url, industry, company_size | Async enrichment |
| 10 | `src/app/api/research/entity/route.ts:186` | UPDATE | personal_notes (JSONB) | Entity verification |

### `prospect_pipeline` — 12+ write sites

TLP-specific operational table. Tracks entities through stages: outreach → qualifying → vetting → onboarding → activation → active. Has columns like `td_buyer_id`, `td_traffic_source_id`, `coverage_states`, `intelligence_brief`, `risk_flags` that are deeply TLP-specific.

**No @milo/crm equivalent exists.** This is pipeline/operations data, not CRM data. Would require a new `@milo/pipeline` primitive or stays in milo-ops.

### `campaign_routes` — 1 write site

`src/lib/tools/setup-new-publisher.ts:315` — INSERT with `campaign_id`, `prospect_id`, `publisher_name`, `buyer_name`, `publisher_payout`, `billing_type`, `status`, `td_traffic_source_id`.

**No @milo/crm equivalent.** This is TLP vertical-specific routing configuration.

### `agreed_terms` — 3 write sites

UPSERT/INSERT with terms like `billing_type`, `rate`, `publisher_share`, `duration_seconds`, `states`, `concurrency_cap`, `daily_cap`, `monthly_cap`, `qualifiers`, `schedule_notes`, `special_terms`.

**No @milo/crm equivalent.** These are contractual/financial terms specific to the pay-per-call vertical.

---

## Recommended @milo/crm Schema Amendment (Decision 38 Process)

### Option A: Extend `crm_contacts` metadata (minimal)

Add a JSONB `metadata` column to `crm_contacts` (if not already present) for vertical-specific fields:

```sql
ALTER TABLE crm_contacts
ADD COLUMN IF NOT EXISTS metadata JSONB DEFAULT '{}';
```

Consumer writes would map:
- `timezone` → `metadata.timezone`
- `preferred_channel` → `metadata.preferred_channel`
- `facebook_url` → `metadata.facebook_url`
- `twitter_url` → `metadata.twitter_url`
- `personal_notes` → `metadata.personal_notes` (preserves JSONB type)

### Option B: Add columns to `crm_contacts` (explicit)

```sql
ALTER TABLE crm_contacts
ADD COLUMN timezone TEXT,
ADD COLUMN preferred_channel TEXT,
ADD COLUMN social_urls JSONB DEFAULT '{}';
```

**Recommendation:** Option A (metadata JSONB). These fields are vertical-specific and shouldn't clutter the shared schema. The `metadata` pattern is already used on `crm_counterparties` and `crm_leads`.

---

## Other Consumers That Would Benefit

### mysecretary (E1 — Wave 4)

mysecretary writes to its own local `leads` table (7 write sites). The migration path to @milo/crm requires the same structural mapping:

| mysecretary `leads` column | @milo/crm equivalent | Gap |
|---------------------------|---------------------|-----|
| `name` | `crm_leads.contact_name` | Rename |
| `company` | `crm_leads.company` | Match |
| `email` | `crm_leads.contact_email` | Match |
| `phone` | `crm_leads.contact_phone` | Rename (no prefix) |
| `linkedin_url` | `crm_leads.linkedin_url` | Match |
| `status` | `crm_leads.status` | Match (values may differ) |
| `platform` | `crm_leads.source` | Value mapping needed |
| `notes` | `crm_leads.notes` | Match |
| `enrichment` (JSONB) | `crm_leads.metadata` | Rename |
| `created_at` | `crm_leads.created_at` | Match |

mysecretary's leads migration is more straightforward than milo-ops contacts because:
1. `crm_leads` is a flat table (no parent FK required)
2. Column overlap is higher
3. Write operations (create, update, delete) have direct @milo/crm equivalents

The mysecretary read-path migration (7 functions) was already completed (Decision 36). The write-path migration was deferred to "v0.2 features" but those features already exist.

**Actual blocker for mysecretary E1:** Column name mapping + status value mapping. Not missing APIs.

---

## Migration Path for milo-ops E3

### Phase 1: Create `crm-client.ts` wrapper (like blacklist-client.ts, onboarding-client.ts)

```typescript
// src/lib/crm-client.ts — singleton wrapper for @milo/crm
import { createCrmClient, createContact, updateContact, ... } from '@milo/crm';

function getClient(): CrmClient { ... }

export async function findOrCreateCrmContact(
  companyName: string,
  contactName: string,
  email: string,
  entityType: 'publisher' | 'buyer',
): Promise<string> {
  // 1. Find or create counterparty by company name
  // 2. Create crm_contacts record linked to counterparty
  // 3. Return contact ID
}
```

### Phase 2: Dual-write strategy

Write to BOTH local `contacts` AND `crm_contacts` during transition. Read from `crm_contacts` first (already done for read-paths).

### Phase 3: Cut over

Remove local `contacts` writes, read exclusively from `crm_contacts`.

### Estimated LOC

| Step | LOC | Blocker |
|------|-----|---------|
| crm-client.ts wrapper | ~80 | None |
| Schema amendment (metadata JSONB) | ~5 SQL | Decision 38 approval |
| Dual-write in 10 contact write sites | ~200 | Schema amendment |
| prospect_pipeline / campaign_routes / agreed_terms | 0 | Stay in milo-ops (not CRM) |
| **Total** | ~285 | Schema amendment |

---

## Action Items for Coder-1

1. **Verify**: `crm_contacts` table already has `metadata JSONB` column (check shared Supabase schema)
2. **If not**: Add `metadata JSONB DEFAULT '{}'` via Decision 38 process
3. **Prioritize**: mysecretary E1 write-path migration can proceed immediately — just needs column name mapping, no structural changes
4. **Deprioritize**: milo-ops `contacts` → `crm_contacts` migration requires the dual-write wrapper, which is Coder-2 work after schema amendment

---

## Correction to TASKS.md

The blocked item "Coder-2: milo-ops CRM writes (E3) — blocked on @milo/crm v0.2 write APIs" should be updated to:

> **Coder-2: milo-ops CRM writes (E3)** — blocked on `crm_contacts` metadata JSONB column (Decision 38). Write APIs exist in v0.1.x. `prospect_pipeline`, `campaign_routes`, `agreed_terms` stay in milo-ops (not CRM scope).

---

*Coder-2 — Migration lane*
