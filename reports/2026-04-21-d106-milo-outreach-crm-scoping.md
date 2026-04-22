# D106: milo-outreach → @milo/crm Consumer Migration Scoping

> Coder-3 Research | Directive #106 | Pure scoping — no code changes

---

## 1. Current State

milo-outreach uses a local `leads` table on the **shared Supabase instance** (`tappyckcteqgryjniwjg`) — the same project where `crm_leads` lives. No cross-database blocker.

milo-outreach does **not** consume `@milo/crm` today. It uses direct `.from("leads")` Supabase queries for all lead CRUD. The only @milo/* dependency is `@milo/blacklist` (via `lib/blacklist-gate.ts`), which demonstrates the cross-primitive consumption pattern already works.

---

## 2. Consumer Sites Inventory

15 `.from("leads")` sites across 8 files:

### Read paths (7)

| # | File | Line | Operation | Fields used |
|---|------|------|-----------|-------------|
| R1 | `app/api/admin/leads/check-water-mark/route.ts` | 35 | SELECT count WHERE status='new' | status |
| R2 | `app/api/admin/leads/route.ts` | 32 | SELECT * filter by status/tier, paginated | all fields |
| R3 | `app/api/admin/leads/[id]/route.ts` | 94 | SELECT * by id | all fields |
| R4 | `app/api/cron/check-water-mark/route.ts` | 28 | SELECT count WHERE status='new' | status |
| R5 | `lib/email-sender.ts` | 102 | SELECT * by id for send | all fields |
| R6 | `lib/reply-classifier.ts` | 104 | SELECT * by id for classification | all fields |
| R7 | `lib/draft-generator.ts` | 93 | SELECT * by id for draft gen | all fields |

### Write paths (8)

| # | File | Line | Operation | Fields written |
|---|------|------|-----------|----------------|
| W1 | `app/api/admin/leads/route.ts` | 105 | INSERT new lead | all entity + outreach fields |
| W2 | `app/api/admin/leads/[id]/route.ts` | 144 | UPDATE by id | partial (any field) |
| W3 | `app/api/admin/leads/[id]/route.ts` | 181 | DELETE by id | — |
| W4 | `lib/email-sender.ts` | 272 | UPDATE status after send | status, opener_sent_at / followup*_sent_at, last_contacted_at |
| W5 | `lib/reply-classifier.ts` | 148 | UPDATE after classification | reply_classification, reply_draft_response, reply_classified_at |
| W6 | `lib/draft-generator.ts` | 153 | UPDATE draft column | message_draft_*, template_used |
| W7 | `lib/research-agent.ts` | 270 | INSERT bulk from research | entity fields + tier, hook, source_url, credentials |
| W8 | `lib/leads-queue.ts` | 55 | SELECT * limit 1000 (read, miscategorized — but used to build mutable queue) | all fields |

**Note:** W8 (`leads-queue.ts:55`) is functionally a read (SELECT * LIMIT 1000) used to build an in-memory send queue. No actual writes in that call.

---

## 3. Schema Mapping Audit

### 3a. outreach `leads` DDL (35 columns)

Source: `milo-outreach/supabase/migrations/00001_initial_schema.sql`

### 3b. @milo/crm `crm_leads` DDL (26 columns)

Source: `milo-engine/packages/crm/migrations/00001_initial_schema.sql`

### 3c. Column-by-column mapping

#### Direct/near-direct mappings (15 columns)

| outreach `leads` | crm_leads | Transform | Notes |
|---|---|---|---|
| id | id | Direct | UUID PK |
| tenant_id | tenant_id | Direct | TEXT FK |
| organization | company | Rename | |
| first_name + last_name | contact_name | Concatenate | **Lossy** — split names lost |
| title | role | Rename | |
| email | contact_email | Rename | |
| linkedin_url | linkedin_url | Direct | |
| website | website | Direct | |
| city + state | location | Concatenate | Or store as "city, state" |
| firm_size_estimate | company_size | Rename | |
| assigned_to | assigned_to | Direct | |
| notes | notes | Direct | |
| last_contacted_at | last_contact_at | Rename | |
| created_at | created_at | Direct | |
| updated_at | updated_at | Direct | |

#### Status enum mapping

| outreach status | crm_leads status | Semantic match |
|---|---|---|
| `new` | `new` | Direct |
| `researched` | `researched` | Direct |
| `ready` | — | **No match.** Outreach-only state ("draft ready to send") |
| `sent` | `contacted` | Close match |
| `replied` | `replied` | Direct |
| `demo` | `qualified` | Close match |
| `customer` | `converted` | Close match |
| `dead` | `rejected` | Close match |

**Key insight:** `ready` is purely outreach workflow state (draft prepared, not yet sent). It has no business in a CRM entity lifecycle. This confirms the two status machines serve different purposes — outreach tracks *message workflow*, CRM tracks *entity lifecycle*.

#### Fields that map to crm_leads.metadata JSONB (2 columns)

| outreach `leads` | Target | Rationale |
|---|---|---|
| source_url | metadata.source_url | Research provenance — directory URL where lead was found |
| credentials | metadata.credentials | Professional certifications. Entity data but no dedicated column |

#### Outreach-specific fields — no crm_leads home (15 columns)

| outreach `leads` | Category |
|---|---|
| tier | Outreach prioritization (tier1/tier2/tier3) |
| hook | Personalization note for first contact |
| channel_preference | Preferred outreach channel |
| channel | Actual outreach channel used |
| message_draft_opener | JSONB — AI-generated draft content |
| message_draft_followup1 | JSONB — draft content |
| message_draft_followup2 | JSONB — draft content |
| opener_sent_at | Send timestamp |
| followup1_sent_at | Send timestamp |
| followup2_sent_at | Send timestamp |
| response_notes | Reply notes |
| reply_classification | AI classification of reply |
| reply_draft_response | AI-generated reply draft |
| reply_classified_at | Classification timestamp |
| template_used | Which outreach template was used |

These 15 fields are ALL outreach process data. Per D6: "All outreach stays in milo-outreach."

#### crm_leads columns with no outreach source (8 columns)

| crm_leads column | Notes |
|---|---|
| contact_phone | Outreach doesn't collect phone numbers |
| industry | Not tracked — could infer from tenant vertical |
| source | All outreach leads would be 'research' (fixed value) |
| found_at | created_at is equivalent |
| lead_score | Not tracked — could compute from tier (tier1=80, tier2=50, tier3=20) |
| research_status | Partially maps from outreach status='researched' |
| first_contact_at | opener_sent_at is equivalent |
| converted_to_counterparty_id | Outreach has no conversion tracking |
| converted_at | Outreach has no conversion tracking |

---

## 4. Architecture Assessment

### D6 is the governing decision

SCHEMA.md D6: "@milo/crm does NOT own outreach tables. All outreach stays in milo-outreach. Cross-reference between CRM entities and outreach records uses `tenant_id` + `counterparty_id` or `lead_id` FKs from the outreach side. @milo/crm provides the entity; milo-outreach provides the communication layer."

This means the migration is **NOT** "replace outreach `leads` with `crm_leads`." It is: **outreach keeps its own table for process data, CRM owns entity identity, cross-reference via FK.**

### Three migration architectures

#### Option A: Split table (clean, high effort)

1. Entity fields (name, email, org, website, etc.) migrate to `crm_leads`
2. Outreach fields (drafts, sends, classification) stay in outreach `leads` table (renamed to `outreach_sequences` or similar)
3. Outreach table gets `crm_lead_id UUID REFERENCES crm_leads(id)` FK
4. Entity reads go through @milo/crm API. Outreach reads stay local.

**Effort:** High. 15 consumer sites rewritten. Data migration script. Dual status tracking. All 8 files touched.

#### Option B: FK link + gradual migration (pragmatic, medium effort)

1. Keep outreach `leads` table as-is with all current columns
2. Add `crm_lead_id UUID REFERENCES crm_leads(id)` column
3. On lead INSERT: also create a `crm_leads` row with entity data, store FK
4. Reads that need cross-system context JOIN to `crm_leads`
5. Outreach CRUD stays on local `leads` table — no breaking changes
6. Entity data is dual-written (outreach `leads` + `crm_leads`)

**Effort:** Medium. 2 write paths modified (W1 INSERT, W7 bulk INSERT). 0 read paths changed initially. New migration adds column + backfill script.

#### Option C: Read-only consumer (minimal, low effort)

1. Keep outreach `leads` table unchanged
2. Use @milo/crm for cross-vertical lead aggregation queries only
3. No FK link, no data sync
4. milo-outreach "consumes" @milo/crm by importing types and using the API for new cross-vertical features

**Effort:** Low. Add @milo/crm as dependency. No schema changes. No existing code changes.

### Recommendation: Option B

Option B follows D6 exactly, doesn't break existing code, and establishes the cross-reference FK that enables future features (cross-vertical lead dashboard, dedup, CRM activity log). Option A is the eventual target but requires a coordinated migration that's premature given milo-outreach is the only outreach consumer today (Pattern 12: verify consumers exist before building).

---

## 5. Tenant Assignment

Outreach leads belong to `tenant_id = 'mysecretary'` (the only seeded tenant in milo-outreach). The crm_leads rows created during migration would also use `tenant_id = 'mysecretary'`.

@milo/crm already has 'mysecretary' as a valid tenant on the shared Supabase instance (the `tenants` table is shared).

No tenant conflicts.

---

## 6. Blocker Assessment

### B1: Name split — first_name + last_name → contact_name (MEDIUM)

crm_leads has single `contact_name`. Outreach stores `first_name` and `last_name` separately because email personalization uses first name only ("Hi {first_name}"). Concatenating for crm_leads is lossy — splitting `contact_name` back is unreliable.

**Mitigation options:**
- (a) Store split names in crm_leads.metadata: `{"first_name": "...", "last_name": "..."}`
- (b) Keep first_name/last_name on outreach `leads` table (Option B: no column removal)
- (c) Add first_name/last_name columns to crm_leads (schema change — not recommended for one consumer)

**Recommendation:** (b) for Option B. Outreach keeps its split name columns. crm_leads.contact_name gets the concatenated display name. No data loss, no schema change.

### B2: Status enum incompatibility (MEDIUM)

outreach status tracks message workflow (8 values). crm_leads status tracks entity lifecycle (7 values). `ready` has no CRM equivalent.

**Mitigation:** Maintain both statuses independently. Outreach `leads.status` stays as-is for outreach workflow. `crm_leads.status` gets updated at key lifecycle transitions:
- outreach `new` → crm `new`
- outreach `sent` → crm `contacted`
- outreach `replied` → crm `replied`
- outreach `demo` → crm `qualified`
- outreach `customer` → crm `converted`
- outreach `dead` → crm `rejected`
- outreach `researched` and `ready` → crm `new` (no CRM-visible state change)

**This is architecturally correct per D7:** horizontal primitive owns the state machine, vertical owns the business rules that gate transitions.

### B3: @milo/crm API doesn't support metadata-aware insert (LOW)

`createLead()` accepts `LeadInsert` which includes `metadata: Record<string, unknown>`. Metadata pass-through works. No blocker.

### B4: Bulk insert path (MEDIUM)

`research-agent.ts:270` (W7) does bulk INSERT of leads discovered by the AI research agent. Currently inserts directly via Supabase. To dual-write to crm_leads, each bulk insert would need a corresponding crm_leads bulk insert.

**Mitigation:** @milo/crm doesn't expose a `bulkCreateLeads()` — either loop through `createLead()` (inefficient for 10-50 leads per research run), or add a bulk insert to @milo/crm, or do a direct `.from("crm_leads").insert([...])` batch alongside the outreach insert.

### B5: No `deleteLead()` in @milo/crm API (LOW)

W3 does DELETE by id. @milo/crm exports `listLeads`, `getLead`, `createLead`, `updateLead`, `promoteLeadToCounterparty` — but no `deleteLead`. Would need to either add it to @milo/crm or use direct Supabase delete + ON DELETE CASCADE on the FK (if crm_lead_id FK exists, deleting the outreach lead doesn't auto-delete the crm_leads row — and shouldn't, since other systems may reference it).

**Mitigation:** On outreach lead DELETE, set crm_leads.status = 'rejected' rather than deleting. Or: leave crm_leads row as orphan (acceptable for soft-delete semantics). Decision needed from Mark.

### B6: `promoteLeadToCounterparty()` not wired to outreach (LOW — future)

@milo/crm has `promoteLeadToCounterparty()` which creates a counterparty from a lead. Outreach status `customer` is semantically equivalent to "converted." Currently outreach has no conversion flow. This becomes relevant when milo-outreach gains a "mark as customer" action.

**Not a migration blocker.** Future feature.

---

## 7. Proposed Commit Sequence (Option B)

### Commit 1: Schema migration — add crm_lead_id FK

```sql
-- milo-outreach migration: 00005_crm_lead_link.sql
ALTER TABLE leads ADD COLUMN crm_lead_id UUID REFERENCES crm_leads(id) ON DELETE SET NULL;
CREATE INDEX idx_leads_crm_lead ON leads(crm_lead_id);
```

**Risk:** Low. Nullable column, no existing data affected.

### Commit 2: Backfill script — create crm_leads rows for existing outreach leads

Script reads all outreach `leads`, creates corresponding `crm_leads` rows with mapped entity data, sets `crm_lead_id` FK.

Mapping per row:
```
crm_leads.company           = leads.organization
crm_leads.contact_name      = CONCAT(leads.first_name, ' ', leads.last_name)
crm_leads.contact_email     = leads.email
crm_leads.linkedin_url      = leads.linkedin_url
crm_leads.website           = leads.website
crm_leads.company_size      = leads.firm_size_estimate
crm_leads.location          = CONCAT(leads.city, ', ', leads.state)
crm_leads.source            = 'research'
crm_leads.found_at          = leads.created_at
crm_leads.status            = <mapped from outreach status>
crm_leads.role              = leads.title
crm_leads.first_contact_at  = leads.opener_sent_at
crm_leads.last_contact_at   = leads.last_contacted_at
crm_leads.assigned_to       = leads.assigned_to
crm_leads.notes             = leads.notes
crm_leads.metadata          = {"source_url": leads.source_url, "credentials": leads.credentials}
```

**Risk:** Medium. Requires testing with real data. Run as dry-run first.

### Commit 3: Write-path dual-write — INSERT and bulk INSERT

Modify W1 (`app/api/admin/leads/route.ts:105`) and W7 (`lib/research-agent.ts:270`) to also create crm_leads rows on lead creation.

**Risk:** Medium. Two code paths to maintain. Use a shared helper to avoid divergence.

### Commit 4: Write-path dual-write — UPDATE status sync

Modify W4 (`email-sender.ts:272`) to update crm_leads.status when outreach status changes to a CRM-mapped value (sent→contacted, replied→replied, etc.).

**Risk:** Low. Only fires on status transitions that have CRM mappings.

### Commit 5: Add @milo/crm as package dependency

`npm install @milo/crm` in milo-outreach. Import types for type safety on crm_leads operations.

**Risk:** None.

### Deferred: Read-path migration

No read paths change in this phase. Outreach continues reading from its own `leads` table. Cross-vertical reads (future dashboard) would query crm_leads.

---

## 8. Open Questions for Mark

**Q1.** Option B (FK link + gradual migration) vs Option A (full split) vs Option C (read-only consumer) — which migration depth does Mark want? Recommendation is B.

**Q2.** On outreach lead DELETE: should the linked crm_leads row be (a) soft-deleted via status='rejected', (b) hard-deleted, or (c) left as orphan? Recommendation is (a).

**Q3.** Bulk insert (research-agent): should @milo/crm get a `bulkCreateLeads()` API, or is direct Supabase insert acceptable for the backfill and ongoing research agent writes?

**Q4.** Timeline: should this migration run in parallel with Phase 2 in-flight work (D46, D62, D69, D70), or is it Phase 3?

---

## 9. Effort Estimate (Option B)

| Commit | Effort | Lane |
|---|---|---|
| 1. Schema migration | 1 directive | Coder-2 (Migration) |
| 2. Backfill script | 1 directive | Coder-2 (Migration) |
| 3. INSERT dual-write | 1 directive | Coder-2 (Migration) |
| 4. UPDATE status sync | 1 directive | Coder-2 (Migration) |
| 5. Package dependency | Bundled with #3 | — |
| **Total** | **4 directives** | Coder-2 |

Prerequisite: Commits 1-2 are gated on Mark answering Q1 (Option B confirmation) and Q2 (delete behavior).
