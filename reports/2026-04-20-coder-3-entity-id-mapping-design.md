---
directive: "entity_id_mapping table schema design"
lane: research
coder: Coder-3
directive-number: 55
started: 2026-04-20
completed: 2026-04-20
status: DEFERRED — Phase 2 stress-test (D58) found denormalized display names already cover all UI needs. SCHEMA.md amendment reverted (D59). Original design preserved below for Phase 3+ reference.
depends-on:
  - reports/2026-04-20-coder-3-milo-for-ppc-phase2-execution-plan.md (d5244f8 — PRE-1)
  - reports/2026-04-20-coder-3-analysis-migration-audit.md (metadata.source_publisher_id/source_buyer_id)
  - Decision 32 (@milo/crm v0.1.1 — 601 TLP counterparties)
  - Decision 38 (schema change process — SCHEMA.md amendment first)
  - Decision 61 (milo-ops contacts migration — findOrCreateCounterparty)
---

# Entity ID Mapping Table — Schema Design (DEFERRED)

> **DEFERRED per Directive #58 stress-test (Decision 67).** Phase 2 UIs use denormalized display
> names already in migrated data — no ID resolution needed. Original design preserved below for
> Phase 3+ reference when CRM entity detail pages with "related documents" tab are built, or when
> the first external CRM integration (FieldEdge, Clio) requires cross-system entity mapping.

## 1. Problem

33 migrated `analysis_records` carry MOP's `publisher_id` and `buyer_id` in `metadata.source_publisher_id` / `metadata.source_buyer_id` (UUID). These reference MOP's frozen `publishers` / `buyers` tables — separate tables that predate @milo/crm's unified `crm_counterparties`.

Phase 2 UIs (analysis history, negotiation admin, signing dashboard) need to display counterparty names. Without a mapping table, every render requires a cross-database lookup to frozen MOP Supabase.

Decision 32 migrated 601 TLP counterparties into `crm_counterparties`, but the migration used `findOrCreateCounterparty()` which matched by company name and generated **new UUIDs** — it did not preserve MOP's `publishers.id` / `buyers.id`. No persistent mapping was stored.

---

## 2. Placement Recommendation

**Recommended: @milo/crm package** — add `crm_entity_id_mappings` table.

| Option | Pros | Cons |
|--------|------|------|
| **@milo/crm (recommended)** | CRM owns the target FK; future verticals reuse same pattern; SCHEMA.md + Decision 38 process already established; RLS mirrors crm_counterparties | Adds a table to a shared package for what starts as a TLP-only need |
| Milo-for-PPC vertical-owned | PPC-scoped, no shared package impact | Duplicates CRM concerns; mysecretary/PPCRM would need separate tables; FK to crm_counterparties crosses package boundaries |
| Shared Supabase standalone | No package coupling | Orphan table without package ownership; violates @milo/* primitive convention; no SCHEMA.md home |

**Rationale:** Entity reconciliation is a CRM concern — mapping external system IDs to unified counterparties. When PPCRM unfreezes (Wave 3) or mysecretary needs legacy ID mapping, the same table serves all verticals. The `source_system` column makes it extensible without migration.

**Decision 38 implication:** @milo/crm SCHEMA.md must be amended BEFORE implementation.

---

## 3. Full DDL

```sql
-- Migration: packages/crm/migrations/00003_entity_id_mappings.sql
-- Requires: 00001_initial_schema.sql (crm_counterparties)

CREATE TABLE crm_entity_id_mappings (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id TEXT NOT NULL,
  source_system TEXT NOT NULL,
  source_entity_type TEXT NOT NULL,
  source_id UUID NOT NULL,
  target_counterparty_id UUID REFERENCES crm_counterparties(id) ON DELETE CASCADE,
  match_method TEXT NOT NULL DEFAULT 'name_match',
  match_confidence TEXT NOT NULL DEFAULT 'high'
    CHECK (match_confidence IN ('verified', 'high', 'low', 'unresolved')),
  unresolved_reason TEXT,
  notes TEXT,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Primary lookup: "given this MOP publisher_id, who is the counterparty?"
CREATE UNIQUE INDEX idx_crm_entity_mapping_source
  ON crm_entity_id_mappings(tenant_id, source_system, source_id);

-- Reverse lookup: "which legacy IDs map to this counterparty?"
CREATE INDEX idx_crm_entity_mapping_target
  ON crm_entity_id_mappings(target_counterparty_id);

-- Filter by source system + type for batch operations
CREATE INDEX idx_crm_entity_mapping_system_type
  ON crm_entity_id_mappings(tenant_id, source_system, source_entity_type);

-- RLS (matches crm_counterparties policy pattern)
ALTER TABLE crm_entity_id_mappings ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Service role full access" ON crm_entity_id_mappings
  FOR ALL USING (true) WITH CHECK (true);

CREATE POLICY "Authenticated read own tenant" ON crm_entity_id_mappings
  FOR SELECT USING (
    tenant_id = current_setting('app.tenant_id', true)
  );
```

### Column rationale

| Column | Type | Purpose |
|--------|------|---------|
| `tenant_id` | TEXT NOT NULL | Multi-tenant scoping (per Decision 24) |
| `source_system` | TEXT NOT NULL | `'mop'`, `'ppcrm'`, `'mysecretary'`, etc. TEXT not ENUM — extensible without migration |
| `source_entity_type` | TEXT NOT NULL | `'publisher'`, `'buyer'`, `'partner'`, `'lead'`, etc. Preserves the original entity classification |
| `source_id` | UUID NOT NULL | The original ID from the source system |
| `target_counterparty_id` | UUID FK (nullable) | Resolves to `crm_counterparties.id`. NULL when `match_confidence = 'unresolved'` (Q1 locked: nullable FK) |
| `match_method` | TEXT NOT NULL | How the mapping was established: `'name_match'` (automated company name), `'id_preserved'` (UUID carried over, e.g. D32 partners), `'manual'` (human-curated) |
| `match_confidence` | TEXT NOT NULL | `'verified'` (human confirmed), `'high'` (exact name match), `'low'` (fuzzy match), `'unresolved'` (no match found — target_counterparty_id is NULL) |
| `unresolved_reason` | TEXT | Why mapping failed: `'not_found_in_source'`, `'no_crm_match'`, `'ambiguous_match'`, etc. NULL when resolved. (Q1 locked: audit trail) |
| `notes` | TEXT | Optional context for manual or disputed mappings |

### UNIQUE constraint

`UNIQUE(tenant_id, source_system, source_id)` — one source ID maps to exactly one counterparty per tenant. If MOP publisher ABC123 was merged into counterparty XYZ789, that's the canonical mapping. No many-to-many.

### Why no `migrated_from_metadata` column

The directive suggested a boolean to track whether the entry was derived from analysis_records metadata vs manually curated. This is fully captured by `match_method`:
- `'name_match'` = derived from backfill script scanning metadata
- `'manual'` = human-curated
- `'id_preserved'` = UUID carried forward (e.g., D32 partner migrations)

Adding a boolean would duplicate information.

---

## 4. Backfill Strategy

### Data flow

```
analysis_records.metadata.source_publisher_id  ─┐
analysis_records.metadata.source_buyer_id      ─┤
                                                 │
            ┌────────────────────────────────────┘
            ▼
  MOP Supabase (frozen, read-only)
  publishers.id → publishers.company
  buyers.id → buyers.company
            │
            ▼
  crm_counterparties (shared Supabase, tenant='tlp')
  Match on: company name + counterparty_type
            │
            ▼
  INSERT INTO crm_entity_id_mappings
```

### Backfill script (pseudocode)

```typescript
// Step 1: Extract unique source IDs from analysis_records metadata
const analysisRecords = await sharedSupabase
  .from('analysis_records')
  .select('metadata')
  .eq('tenant_id', 'tlp')
  .not('metadata->source_publisher_id', 'is', null);

const uniqueSourceIds = new Map<string, { type: 'publisher' | 'buyer' }>();

for (const record of analysisRecords) {
  const meta = record.metadata;
  if (meta.source_publisher_id) {
    uniqueSourceIds.set(meta.source_publisher_id, { type: 'publisher' });
  }
  if (meta.source_buyer_id) {
    uniqueSourceIds.set(meta.source_buyer_id, { type: 'buyer' });
  }
}

// Step 2: For each source ID, resolve company name from frozen MOP
const mopSupabase = createClient(MOP_URL, MOP_SERVICE_KEY);

const mappings: EntityMapping[] = [];
const unresolved: UnresolvedEntry[] = [];

for (const [sourceId, { type }] of uniqueSourceIds) {
  const table = type === 'publisher' ? 'publishers' : 'buyers';
  
  // Lookup company name in frozen MOP
  const { data: mopEntity } = await mopSupabase
    .from(table)
    .select('id, company')
    .eq('id', sourceId)
    .single();

  if (!mopEntity) {
    unresolved.push({ sourceId, type, reason: 'not_found_in_mop' });
    continue;
  }

  // Match against crm_counterparties
  const { data: counterparty } = await sharedSupabase
    .from('crm_counterparties')
    .select('id, company')
    .eq('tenant_id', 'tlp')
    .eq('counterparty_type', type)
    .ilike('company', mopEntity.company)
    .single();

  if (!counterparty) {
    unresolved.push({
      sourceId, type,
      mopCompany: mopEntity.company,
      reason: 'no_crm_match'
    });
    continue;
  }

  mappings.push({
    tenant_id: 'tlp',
    source_system: 'mop',
    source_entity_type: type,
    source_id: sourceId,
    target_counterparty_id: counterparty.id,
    match_method: 'name_match',
    match_confidence: mopEntity.company === counterparty.company ? 'high' : 'low',
  });
}

// Step 3: Insert mappings
await sharedSupabase
  .from('crm_entity_id_mappings')
  .upsert(mappings, { onConflict: 'tenant_id,source_system,source_id' });

// Step 4: Report
console.log(`Mapped: ${mappings.length}`);
console.log(`Unresolved: ${unresolved.length}`);
for (const u of unresolved) {
  console.log(`  ${u.type} ${u.sourceId}: ${u.reason} (${u.mopCompany || 'unknown'})`);
}
```

### Extended backfill scope (Q2 locked: all three primitives)

The backfill script scans all three contract primitives in one pass:

1. **analysis_records** (33 rows) — `metadata.source_publisher_id` / `metadata.source_buyer_id`
2. **signing_documents** (109 rows) — `metadata.source_publisher_id` / `metadata.source_buyer_id` (if present)
3. **negotiation_records** (2 rows) — linked via `analysis_id` FK → resolve through parent analysis

After mapping, denormalize `counterparty_display_name` + `counterparty_id` into metadata on all three tables.

### Expected dataset size

- 33 analysis_records + 109 signing_documents + 2 negotiation_records = 144 total rows to scan
- Many rows share the same publisher_id or buyer_id
- Estimated unique source IDs: ~50-150 (Mark's estimate)
- CRM counterparties available: 601 (D32 migration, tenant='tlp')
- Expected match rate: high (D61 findOrCreateCounterparty already matched most by name)

---

## 5. Usage Pattern Recommendation

**Recommended: Dual approach — backfill display_name + mapping table for runtime.**

### For 33 historical records: denormalized display_name

During backfill, also write `metadata.counterparty_display_name` on each analysis_record:

```typescript
// During backfill Step 3 (after mapping is established):
for (const record of analysisRecords) {
  const pubMapping = mappings.find(m => m.source_id === record.metadata.source_publisher_id);
  const buyMapping = mappings.find(m => m.source_id === record.metadata.source_buyer_id);

  await sharedSupabase
    .from('analysis_records')
    .update({
      metadata: {
        ...record.metadata,
        counterparty_display_name: pubMapping?.counterpartyName || buyMapping?.counterpartyName || null,
        counterparty_id: pubMapping?.target_counterparty_id || buyMapping?.target_counterparty_id || null,
      }
    })
    .eq('id', record.id);
}
```

**Why denormalize:** The 33 historical records are static — their counterparty relationships will never change. Denormalizing avoids a 3-table JOIN on every history page load for data that's frozen. The analysis history list page (A4) renders a table of records — adding a JOIN to every row query adds latency for no benefit on static data.

### For new Milo-for-PPC analyses: direct counterparty_id

New analyses created through Milo-for-PPC's upload flow (A2) will select the counterparty from @milo/crm directly. The analysis_record gets `metadata.counterparty_id` set at creation time — no mapping table lookup needed.

### For edge cases: mapping table JOIN

If a UI needs to resolve an old MOP ID that wasn't denormalized (e.g., signing_documents or negotiation_records referencing publisher_id), use the mapping table:

```sql
SELECT
  ar.*,
  cp.company AS counterparty_name
FROM analysis_records ar
LEFT JOIN crm_entity_id_mappings m
  ON m.source_id = (ar.metadata->>'source_publisher_id')::UUID
  AND m.source_system = 'mop'
  AND m.tenant_id = ar.tenant_id
LEFT JOIN crm_counterparties cp
  ON cp.id = m.target_counterparty_id
WHERE ar.tenant_id = 'tlp';
```

---

## 6. Unresolved Case Handling

Three categories of unresolved mappings:

| Scenario | Cause | Handling |
|----------|-------|----------|
| MOP ID not found in MOP tables | Data inconsistency in MOP (orphaned FK) | Log as `not_found_in_mop`. Insert mapping with `match_confidence = 'unresolved'`, `target_counterparty_id` = NULL (skip FK constraint via deferred check or separate unresolved table). |
| MOP company found but no CRM match | D32 migration missed this entity, or name mismatch | Log with MOP company name. Create the counterparty in crm_counterparties, then insert mapping with `match_confidence = 'low'`. Report to Mark for review. |
| Multiple CRM matches for one company name | Duplicate counterparties in CRM | Log ambiguity. Pick the most recently active one. Insert mapping with `match_confidence = 'low'` and `notes` explaining the ambiguity. Report to Mark. |

**Unresolved entries must never block the backfill.** The script runs to completion, logs all issues, and Mark reviews the exceptions. Phase 2 UIs handle missing display_name gracefully (show "Unknown" or the raw document_name).

### Alternative for unresolved: skip the mapping, leave metadata as-is

For analysis_records where no counterparty match exists, the history UI falls back to:
1. `metadata.counterparty_display_name` (if backfill populated it)
2. `metadata.extracted_company_name` (from MOP's AI extraction)
3. `document_name` (always present)

This means unresolved mappings don't break any UI — they just show less-specific entity names.

---

## 7. Forward Compatibility

### PPCRM partner_id (Wave 3)

When PPCRM unfreezes, partner_id → crm_counterparty_id mappings use the same table:

```sql
INSERT INTO crm_entity_id_mappings (tenant_id, source_system, source_entity_type, source_id, target_counterparty_id, match_method)
VALUES ('tlp', 'ppcrm', 'partner', $ppcrm_partner_id, $crm_id, 'id_preserved');
```

D32 already verified UUID preservation for partners (`partners.id = crm_counterparties.id`), so these mappings will have `match_method = 'id_preserved'` and `match_confidence = 'verified'`.

### mysecretary legacy IDs

```sql
INSERT INTO crm_entity_id_mappings (tenant_id, source_system, source_entity_type, source_id, target_counterparty_id, match_method)
VALUES ('mysecretary', 'mysecretary', 'lead', $legacy_lead_id, $crm_id, 'name_match');
```

### New source systems

Adding a new source system requires zero schema changes — `source_system` is TEXT, not ENUM. Insert rows with the new system name.

### Extensibility checklist

| Future need | Supported? | How |
|-------------|-----------|-----|
| PPCRM partner_id mapping | Yes | `source_system = 'ppcrm'` |
| mysecretary legacy leads | Yes | `source_system = 'mysecretary'` |
| TrackDrive entity mapping | Yes | `source_system = 'trackdrive'` |
| Multiple target types (not just counterparty) | No — would need new table | Current scope is counterparty mapping only. If we need asset→asset mapping, that's a different table. |
| Bi-directional mapping (CRM → MOP) | Yes | Reverse index on target_counterparty_id already exists |

---

## 8. Decision 38 Process: SCHEMA.md Amendment

Per Decision 38, the following must happen BEFORE implementation:

### Required SCHEMA.md amendment for @milo/crm

Add to `packages/crm/SCHEMA.md`:

```markdown
## crm_entity_id_mappings

Maps legacy entity IDs from source systems to unified crm_counterparties. Supports multi-system reconciliation during migrations.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | UUID | PK, gen_random_uuid() | Row ID |
| tenant_id | TEXT | NOT NULL | Tenant scope |
| source_system | TEXT | NOT NULL | Source system identifier (e.g. 'mop', 'ppcrm') |
| source_entity_type | TEXT | NOT NULL | Original entity type (e.g. 'publisher', 'buyer') |
| source_id | UUID | NOT NULL | Original entity ID in source system |
| target_counterparty_id | UUID | FK crm_counterparties(id) CASCADE, NULLABLE | Resolved CRM counterparty (NULL when unresolved) |
| match_method | TEXT | NOT NULL, DEFAULT 'name_match' | How mapping was established |
| match_confidence | TEXT | NOT NULL, DEFAULT 'high', CHECK IN (verified, high, low, unresolved) | Confidence level |
| unresolved_reason | TEXT | | Why mapping failed (NULL when resolved) |
| notes | TEXT | | Optional context |
| created_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() | Creation timestamp |

**Indexes:**
- UNIQUE(tenant_id, source_system, source_id)
- (target_counterparty_id)
- (tenant_id, source_system, source_entity_type)

**RLS:** Service role full access. Authenticated users read own tenant.
```

### Implementation sequence

1. Commit SCHEMA.md amendment to @milo/crm
2. Create migration file `packages/crm/migrations/00003_entity_id_mappings.sql`
3. Apply DDL to shared Supabase
4. Run backfill script
5. Verify all 33 analysis_records resolve
6. Commit backfill results report

---

## 9. Open Questions — ALL RESOLVED (D57, Decision 66)

| # | Question | Answer | Resolved |
|---|----------|--------|----------|
| Q1 | Nullable FK for unresolved entries? | **YES — nullable FK + add `unresolved_reason TEXT` column.** Audit trail of failed mappings, easier than re-running backfill. | Mark approved (D57) |
| Q2 | Backfill scope — analysis only or all three primitives? | **ALL THREE in one pass.** analysis_records (33) + signing_documents (109) + negotiation_records (2). Single script, single review cycle. ~50-150 mappings expected. | Mark approved (D57) |
| Q3 | Version bump — v0.2.1 patch or v0.3.0 minor? | **v0.3.0 minor.** New table = new feature surface area. Matches v0.2.0 minor bump for crm_contacts metadata. | Mark approved (D57) |

---

## 10. Summary

| Aspect | Decision |
|--------|----------|
| **Placement** | @milo/crm package (`crm_entity_id_mappings` table) |
| **Schema** | 11 columns (+ unresolved_reason), nullable FK, UNIQUE on (tenant_id, source_system, source_id), 3 indexes, RLS |
| **Backfill** | Scan ALL THREE primitives (analysis + signing + negotiation) → lookup MOP frozen tables → match crm_counterparties by name |
| **Usage** | Denormalized display_name for 33 historical records + mapping table for runtime edge cases |
| **Unresolved handling** | Log, don't block. UI falls back to extracted_company_name or document_name |
| **Forward compat** | TEXT source_system supports PPCRM, mysecretary, TrackDrive without migration |
| **Decision 38** | SCHEMA.md amendment required before implementation |
| **Estimated LOC** | ~100 (DDL + backfill script), as estimated in Phase 2 plan |
