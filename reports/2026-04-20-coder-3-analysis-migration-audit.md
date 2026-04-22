---
directive: "@milo/contract-analysis data migration audit"
lane: research
coder: Coder-3
started: 2026-04-20 ~03:15 MDT
completed: 2026-04-20 ~03:45 MDT
---

# @milo/contract-analysis Data Migration Audit — 2026-04-20

## 1. Executive Summary

| Metric | Value |
|--------|-------|
| Source tables | 4 (contract_analyses, contract_analysis_jobs, contract_issue_decisions, contract_issue_resolutions) |
| Target tables | 2 (analysis_records, analysis_jobs) |
| Rows to migrate (contract_analyses) | 33 |
| Rows to migrate (contract_analysis_jobs) | Low (transient queue) |
| Rows to skip (contract_issue_decisions) | 0 (dead table) |
| Rows to evaluate (contract_issue_resolutions) | Low (vertical-coupled) |
| Storage files to migrate | 22 (contract-originals bucket) |
| Column mapping complexity | MEDIUM — 5 columns fold to metadata JSONB |
| Ordering position | **FIRST** (analysis → signing → negotiation) |
| Target Supabase | tappyckcteqgryjniwjg (shared) |
| Tenant assignment | All rows: tenant_id = 'tlp' |

**Key decisions needed:**
- (c) `contract_issue_decisions`: SKIP — 0 rows, dead table
- (d) `contract_issue_resolutions`: NOT in target schema — stays vertical or deferred to v0.2.0
- (e) `publisher_id`/`buyer_id`: fold to `metadata.source_publisher_id` / `metadata.source_buyer_id` (not @milo/crm counterparty_id — see §5)
- (h) Storage: copy files to shared Supabase bucket, update `original_file_path`

## 2. Source Schema: contract_analyses (MOP frozen)

33 rows. Full column inventory:

| Column | Type | Null | Default | Notes |
|--------|------|------|---------|-------|
| id | UUID | NO | gen_random_uuid() | PK |
| document_name | VARCHAR(255) | NO | — | |
| document_hash | VARCHAR(64) | YES | NULL | SHA-256, unique where not null |
| role | VARCHAR(20) | NO | — | 'publisher' or 'buyer' |
| counterparty_jurisdiction | VARCHAR(100) | YES | NULL | |
| raw_text | TEXT | NO | — | |
| summary | JSONB | NO | '[]' | AnalysisSummary object |
| issues | JSONB | NO | '[]' | ContractIssue[] array |
| created_at | TIMESTAMPTZ | NO | NOW() | |
| updated_at | TIMESTAMPTZ | NO | NOW() | |
| archived_at | TIMESTAMPTZ | YES | NULL | Soft delete |
| publisher_id | UUID | YES | NULL | FK publishers(id) |
| buyer_id | UUID | YES | NULL | FK buyers(id) |
| contract_group_id | UUID | YES | NULL | Version chain grouping |
| version_number | VARCHAR(20) | YES | '1.0' | |
| revision_label | VARCHAR(255) | YES | NULL | |
| negotiation_stage | VARCHAR(20) | YES | 'initial' | 'initial', 'negotiation', 'finalized' |
| parent_analysis_id | UUID | YES | NULL | Self-ref for version chain |
| is_finalized | BOOLEAN | YES | FALSE | |
| extracted_company_name | VARCHAR(255) | YES | NULL | |
| display_name | TEXT | YES | NULL | |
| original_file_path | VARCHAR(500) | YES | NULL | Path in contract-originals bucket |
| original_file_size | INTEGER | YES | NULL | |
| original_file_mime_type | VARCHAR(100) | YES | NULL | |
| jurisdiction_confidence | VARCHAR(10) | YES | NULL | HIGH/MEDIUM/LOW/UNKNOWN |
| jurisdiction_source | VARCHAR(50) | YES | NULL | |
| counterparty_info | JSONB | YES | NULL | AI-extracted entity info |
| ruleset_version | VARCHAR(20) | YES | NULL | '1.0.0' |
| scoring_version | VARCHAR(20) | YES | NULL | '1.0.0' |
| model_name | VARCHAR(50) | YES | NULL | 'claude-sonnet-4-5-20250929' |
| profile_used | VARCHAR(20) | YES | NULL | 'TEST', 'SCALE', 'BROKER' |

**Total: 30 columns.**

## 3. Source Schema: contract_analysis_jobs (MOP frozen)

Transient job queue. Low row count (jobs complete quickly or fail).

| Column | Type | Null | Default | Notes |
|--------|------|------|---------|-------|
| id | UUID | NO | gen_random_uuid() | PK |
| status | TEXT | NO | 'pending' | pending/processing/completed/failed |
| document_text | TEXT | NO | — | |
| role | TEXT | NO | 'publisher' | |
| file_name | TEXT | YES | NULL | |
| document_hash | TEXT | YES | NULL | |
| playbook_preset | TEXT | YES | 'full_audit' | **PPC-specific** |
| threshold_profile | TEXT | YES | 'standard' | **PPC-specific** |
| analysis_profile | TEXT | YES | 'standard' | **PPC-specific** |
| counterparty_jurisdiction | TEXT | YES | NULL | |
| result | JSONB | YES | NULL | |
| error_message | TEXT | YES | NULL | |
| created_at | TIMESTAMPTZ | NO | NOW() | |
| updated_at | TIMESTAMPTZ | NO | NOW() | |
| completed_at | TIMESTAMPTZ | YES | NULL | |
| analysis_id | UUID | YES | NULL | FK contract_analyses(id) |

**Total: 16 columns.**

## 4. Source Schema: contract_issue_decisions (MOP frozen)

**0 rows. Dead table.** Created in initial migration but never populated by application code.

| Column | Type |
|--------|------|
| id | UUID |
| analysis_id | UUID (FK, CASCADE) |
| issue_id | INTEGER |
| status | VARCHAR(20) — 'pending', 'accepted', 'rejected', 'edited' |
| edited_text | TEXT |
| created_at | TIMESTAMPTZ |

**Decision: SKIP.** No data to migrate. Table can be dropped from MOP during unfreeze cleanup.

## 5. Source Schema: contract_issue_resolutions (MOP frozen)

VA review decision tracking. Rows exist but count is low.

| Column | Type | Notes |
|--------|------|-------|
| id | UUID | PK |
| analysis_id | UUID | FK contract_analyses(id), CASCADE |
| issue_index | INTEGER | Position in issues[] array |
| rule_id | TEXT | Which rule flagged it |
| resolution_type | TEXT | 'accepted', 'rejected', 'deferred', 'escalated', 'custom_edit' |
| resolved_by | TEXT | DEFAULT 'TLP Compliance' ← **hardcoded vertical** |
| resolved_at | TIMESTAMPTZ | |
| notes | TEXT | |
| custom_revision_text | TEXT | |

**Decision: NOT in @milo/contract-analysis target schema.** This table is NOT defined in the v0.1.0 SCHEMA.md. Two options:

1. **Defer** — Leave in MOP, migrate when v0.2.0 adds resolution tracking (recommended)
2. **Fold into metadata** — Store as `metadata.resolutions[]` on parent analysis_records row

Recommendation: **Defer.** The resolution workflow is tightly coupled to the PPC VA review process. The `resolved_by = 'TLP Compliance'` hardcode confirms vertical coupling. When @milo/contract-analysis v0.2.0 adds generic resolution tracking, this data can migrate with proper tenant-scoped `resolved_by` handling.

## 6. Column Mapping: contract_analyses → analysis_records

### Direct Mapping (25 columns)

| Source Column | Target Column | Transform |
|---------------|---------------|-----------|
| id | id | Direct |
| — | tenant_id | Constant: `'tlp'` |
| document_name | document_name | Direct |
| display_name | display_name | Direct |
| document_hash | document_hash | Direct |
| role | role | Direct (values 'publisher'/'buyer' valid in target CHECK) |
| counterparty_jurisdiction | counterparty_jurisdiction | Direct |
| jurisdiction_confidence | jurisdiction_confidence | Direct |
| jurisdiction_source | jurisdiction_source | Direct |
| raw_text | raw_text | Direct |
| summary | summary | Direct (JSONB) |
| issues | issues | Direct (JSONB) |
| — | score | NULL (not computed in MOP source) |
| contract_group_id | contract_group_id | Direct |
| version_number | version_number | Direct |
| revision_label | revision_label | Direct |
| parent_analysis_id | parent_analysis_id | Direct (self-ref preserved) |
| counterparty_info | counterparty_info | Direct (JSONB) |
| original_file_path | original_file_path | **Remap** (new bucket path, see §10) |
| original_file_size | original_file_size | Direct |
| original_file_mime_type | original_file_mime_type | Direct |
| ruleset_version | ruleset_version | Direct |
| scoring_version | scoring_version | Direct |
| model_name | model_name | Direct |
| profile_used | profile_used | Direct |
| archived_at | archived_at | Direct |
| created_at | created_at | Direct |
| updated_at | updated_at | Direct |

### Folded to metadata JSONB (5 columns)

| Source Column | Target Location | Rationale |
|---------------|-----------------|-----------|
| publisher_id | metadata.source_publisher_id | PPC entity ref — not @milo/crm counterparty_id |
| buyer_id | metadata.source_buyer_id | PPC entity ref — not @milo/crm counterparty_id |
| negotiation_stage | metadata.source_negotiation_stage | Belongs to @milo/contract-negotiation domain |
| is_finalized | metadata.source_is_finalized | Belongs to @milo/contract-negotiation domain |
| extracted_company_name | metadata.extracted_company_name | Overlaps with counterparty_info.name; preserve for audit trail |

### Why NOT map publisher_id/buyer_id to @milo/crm counterparty_id

The source `publisher_id`/`buyer_id` reference MOP's local `publishers`/`buyers` tables (frozen). The @milo/crm `crm_counterparties` table on shared Supabase already has TLP data migrated (Decision 32), but:

1. We don't have a verified ID mapping table (MOP publisher.id → crm_counterparties.id)
2. Creating that mapping requires MOP unfreeze or a separate reconciliation pass
3. The analysis data is useful independent of entity linkage

**Recommendation:** Store source IDs in metadata now. When MOP unfreezes and entity reconciliation runs, a follow-up migration can populate a future `counterparty_id` column on `analysis_records` (or link via a join table).

## 7. Column Mapping: contract_analysis_jobs → analysis_jobs

### Direct Mapping (10 columns)

| Source Column | Target Column | Transform |
|---------------|---------------|-----------|
| id | id | Direct |
| — | tenant_id | Constant: `'tlp'` |
| status | status | Direct (same CHECK values) |
| document_text | document_text | Direct |
| role | role | Direct |
| file_name | file_name | Direct |
| document_hash | document_hash | Direct |
| counterparty_jurisdiction | counterparty_jurisdiction | Direct |
| result | result | Direct (JSONB) |
| error_message | error_message | Direct |
| created_at | created_at | Direct |
| updated_at | updated_at | Direct |
| completed_at | completed_at | Direct |
| analysis_id | analysis_id | Direct (FK preserved — analysis migrates first) |

### Folded to analysis_config JSONB (3 columns)

| Source Column | Target Location | Rationale |
|---------------|-----------------|-----------|
| playbook_preset | analysis_config.playbook_preset | PPC-specific analysis preset |
| threshold_profile | analysis_config.threshold_profile | PPC-specific threshold tuning |
| analysis_profile | analysis_config.analysis_profile | PPC-specific profile selection |

### New target columns (not in source)

| Target Column | Value | Notes |
|---------------|-------|-------|
| system_prompt | NULL | MOP didn't store system prompts in jobs table |

## 8. Migration SQL: contract_analyses → analysis_records

```sql
INSERT INTO analysis_records (
  id, tenant_id, document_name, display_name, document_hash,
  role, counterparty_jurisdiction, jurisdiction_confidence,
  jurisdiction_source, raw_text, summary, issues, score,
  contract_group_id, version_number, revision_label,
  parent_analysis_id, counterparty_info,
  original_file_path, original_file_size, original_file_mime_type,
  ruleset_version, scoring_version, model_name, profile_used,
  metadata, archived_at, created_at, updated_at
)
SELECT
  id,
  'tlp',
  document_name,
  display_name,
  document_hash,
  role,
  counterparty_jurisdiction,
  jurisdiction_confidence,
  jurisdiction_source,
  raw_text,
  summary,
  issues,
  NULL,  -- score: not computed in MOP
  contract_group_id,
  version_number,
  revision_label,
  parent_analysis_id,
  counterparty_info,
  -- Remap file path: contract-originals/ → analysis-originals/ (or keep same)
  original_file_path,
  original_file_size,
  original_file_mime_type,
  ruleset_version,
  scoring_version,
  model_name,
  profile_used,
  jsonb_build_object(
    'source_publisher_id', publisher_id,
    'source_buyer_id', buyer_id,
    'source_negotiation_stage', negotiation_stage,
    'source_is_finalized', is_finalized,
    'extracted_company_name', extracted_company_name,
    'migrated_from', 'mop',
    'migrated_at', NOW()
  ),
  archived_at,
  created_at,
  updated_at
FROM mop_backup.contract_analyses
ON CONFLICT (id) DO NOTHING;
```

## 9. Migration SQL: contract_analysis_jobs → analysis_jobs

```sql
INSERT INTO analysis_jobs (
  id, tenant_id, status, document_text, role, file_name,
  document_hash, analysis_config, counterparty_jurisdiction,
  system_prompt, result, error_message, analysis_id,
  created_at, updated_at, completed_at
)
SELECT
  id,
  'tlp',
  status,
  document_text,
  role,
  file_name,
  document_hash,
  jsonb_build_object(
    'playbook_preset', playbook_preset,
    'threshold_profile', threshold_profile,
    'analysis_profile', analysis_profile,
    'migrated_from', 'mop'
  ),
  counterparty_jurisdiction,
  NULL,  -- system_prompt: not stored in MOP jobs
  result,
  error_message,
  analysis_id,
  created_at,
  updated_at,
  completed_at
FROM mop_backup.contract_analysis_jobs
ON CONFLICT (id) DO NOTHING;
```

**Note:** Only migrate `completed` and `failed` jobs. Jobs in `pending`/`processing` status at freeze time are stale (MOP is frozen, no processor running). Decision:

- `completed` → migrate (historical record, linked to analysis_id)
- `failed` → migrate (useful for debugging/audit)
- `pending`/`processing` → SKIP (stale, will never complete)

## 10. Storage Bucket Migration

### Source: contract-originals (MOP Supabase wjxtfjaixkoifdqtfmqd)

22 files per freeze manifest. Path pattern: `YYYY-MM/uuid-filename`

File types: PDF, DOCX, DOC, TXT (per bucket config: application/pdf, application/vnd.openxmlformats-officedocument.wordprocessingml.document, application/msword, text/plain)

### Target: contract-originals (shared Supabase tappyckcteqgryjniwjg)

**Two options:**

| Option | Approach | Pros | Cons |
|--------|----------|------|------|
| A: Copy files | Download from MOP → upload to shared | Clean separation, shared bucket is self-contained | 22 files × manual transfer, path update needed |
| B: Reference only | Store MOP signed URLs or keep paths as-is | Zero file movement | Depends on MOP project staying accessible, signed URLs expire |

**Recommendation: Option A (Copy files).** 22 files is trivial volume. MOP is frozen and may eventually be decommissioned. Shared Supabase should be the single source of truth.

### Migration steps:
1. Create `contract-originals` bucket on shared Supabase (same config: 50MB max, same MIME types)
2. Download all 22 files from MOP bucket via `supabase storage download`
3. Upload to shared bucket preserving path structure (`YYYY-MM/uuid-filename`)
4. Verify file checksums match
5. Update `original_file_path` in `analysis_records` if path prefix changes (likely unchanged if bucket name is preserved)

### RLS for shared bucket:
MOP used permissive (allow anonymous + authenticated). Shared Supabase should use tenant-scoped RLS:
```sql
CREATE POLICY "tenant_read" ON storage.objects
  FOR SELECT USING (
    bucket_id = 'contract-originals'
    AND (storage.foldername(name))[1] = current_setting('app.tenant_id', true)
  );
```

This requires restructuring paths from `YYYY-MM/uuid-filename` to `tlp/YYYY-MM/uuid-filename` (tenant prefix). Update `original_file_path` in `analysis_records` accordingly.

## 11. Version Chain Preservation

The `parent_analysis_id` self-reference creates version chains:

```
analysis_v1 (parent_analysis_id = NULL, contract_group_id = X)
  └── analysis_v2 (parent_analysis_id = v1.id, contract_group_id = X)
       └── analysis_v3 (parent_analysis_id = v2.id, contract_group_id = X)
```

**Migration requirement:** All members of a chain must migrate together. With only 33 rows total and all belonging to tenant 'tlp', this is guaranteed — the entire dataset migrates as one unit.

**Integrity check:** After migration, verify:
```sql
SELECT COUNT(*) FROM analysis_records
WHERE parent_analysis_id IS NOT NULL
  AND parent_analysis_id NOT IN (SELECT id FROM analysis_records);
-- Expected: 0 (no orphaned refs)
```

## 12. Rule Versioning Preservation

Source values:
- `ruleset_version`: '1.0.0' (all 33 rows)
- `scoring_version`: '1.0.0' (all 33 rows)
- `model_name`: 'claude-sonnet-4-5-20250929' (all or most rows)

These map directly to target columns. The @milo/contract-analysis engine will use its own versioning going forward, but migrated rows preserve their historical versions for audit trail.

**No transform needed.** Direct copy.

## 13. Idempotency Strategy

```sql
ON CONFLICT (id) DO NOTHING
```

Both `analysis_records` and `analysis_jobs` use UUID primary keys. The migration INSERT uses `ON CONFLICT (id) DO NOTHING`, making it safe to re-run.

Additionally, the `document_hash` unique index on `analysis_records` provides a secondary idempotency guard:
```sql
-- Target index:
idx_analysis_records_hash ON (tenant_id, document_hash)
```

If a document with the same hash already exists for tenant 'tlp', it won't create a duplicate (though the PK conflict fires first since we preserve source UUIDs).

## 14. Data Quality Concerns

### (k) Missing required fields

Source allows NULL on `document_hash`, but target also allows NULL — no issue.

Source `role` is NOT NULL with only 'publisher'/'buyer' values. Target CHECK adds 'vendor', 'client', 'partner' — existing values are valid.

**Potential issue:** `summary` column. Source default is `'[]'::jsonb` but target expects an AnalysisSummary object (not an array). Need to verify:
```sql
-- Detection query (run against backup):
SELECT id, jsonb_typeof(summary) AS summary_type
FROM contract_analyses
WHERE jsonb_typeof(summary) != 'object';
```

If any rows have `summary = '[]'` (array, not object), they're likely incomplete analyses that failed mid-processing. Decision: migrate as-is (target allows any JSONB), flag for consumer code to handle gracefully.

### Orphaned parent_analysis_id refs

```sql
SELECT id FROM contract_analyses
WHERE parent_analysis_id IS NOT NULL
  AND parent_analysis_id NOT IN (SELECT id FROM contract_analyses);
-- Expected: 0
```

### Jobs referencing non-existent analyses

```sql
SELECT id FROM contract_analysis_jobs
WHERE analysis_id IS NOT NULL
  AND analysis_id NOT IN (SELECT id FROM contract_analyses);
-- Expected: 0 (FK constraint would have prevented this)
```

## 15. Test Data Detection

With only 33 rows, some may be test analyses from development. Detection heuristics:

```sql
-- Likely test data:
SELECT id, document_name, created_at
FROM contract_analyses
WHERE document_name ILIKE '%test%'
   OR document_name ILIKE '%sample%'
   OR document_name ILIKE '%demo%'
   OR raw_text ILIKE '%lorem ipsum%'
   OR profile_used = 'TEST';
```

**Decision needed from Mark:** Migrate test data or exclude? Recommendation: migrate all 33 rows (low volume, preserves complete history). Tag test rows in metadata:
```sql
metadata = metadata || '{"is_test": true}'::jsonb
```

## 16. Execution Order

This is the **FIRST** migration in the contract pipeline ordering:

```
┌─────────────────────────────┐
│ 1. @milo/contract-analysis  │  ← THIS AUDIT
│    (analysis_records +      │
│     analysis_jobs)           │
└──────────────┬──────────────┘
               │ analysis_id FK
┌──────────────▼──────────────┐
│ 2. @milo/contract-signing   │  (audit: 2026-04-20)
│    (signing_documents)       │
└──────────────┬──────────────┘
               │ analysis_id FK
┌──────────────▼──────────────┐
│ 3. @milo/contract-negotiation│  (audit: 2026-04-20)
│    (negotiations, rounds,    │
│     links)                   │
└─────────────────────────────┘
```

**Within this migration, internal order:**
1. Create `contract-originals` bucket on shared Supabase
2. Copy 22 storage files (with tenant path prefix)
3. Migrate `analysis_records` (33 rows) — must be first (analysis_jobs.analysis_id FK)
4. Migrate `analysis_jobs` (completed/failed only)
5. Verify integrity (orphan checks, file path validation)

## 17. Pre-flight Checks

Before executing migration:

```sql
-- 1. Verify target tables exist on shared Supabase
SELECT tablename FROM pg_tables
WHERE schemaname = 'public'
  AND tablename IN ('analysis_records', 'analysis_jobs');

-- 2. Verify no existing data conflicts
SELECT COUNT(*) FROM analysis_records WHERE tenant_id = 'tlp';
-- Expected: 0 (first migration)

-- 3. Verify bucket exists
-- (via Supabase dashboard or CLI)

-- 4. Verify source row count matches manifest
SELECT COUNT(*) FROM mop_backup.contract_analyses;
-- Expected: 33
```

## 18. Dry-Run Output Shape

```json
{
  "migration": "@milo/contract-analysis",
  "source": "wjxtfjaixkoifdqtfmqd (MOP frozen)",
  "target": "tappyckcteqgryjniwjg (shared)",
  "tenant_id": "tlp",
  "tables": {
    "analysis_records": {
      "source_rows": 33,
      "migrated": 33,
      "skipped_conflict": 0,
      "columns_direct": 25,
      "columns_to_metadata": 5
    },
    "analysis_jobs": {
      "source_rows": "TBD",
      "migrated": "completed + failed only",
      "skipped_stale": "pending + processing",
      "columns_direct": 10,
      "columns_to_analysis_config": 3
    }
  },
  "storage": {
    "bucket": "contract-originals",
    "files_copied": 22,
    "path_transform": "YYYY-MM/file → tlp/YYYY-MM/file"
  },
  "excluded_tables": {
    "contract_issue_decisions": "0 rows, dead table",
    "contract_issue_resolutions": "vertical-coupled, deferred to v0.2.0"
  },
  "idempotent": true,
  "reversible": true
}
```

## 19. Rollback Plan

If migration needs reversal:

```sql
-- 1. Delete migrated analysis jobs
DELETE FROM analysis_jobs WHERE tenant_id = 'tlp';

-- 2. Delete migrated analysis records
DELETE FROM analysis_records WHERE tenant_id = 'tlp';

-- 3. Remove storage files
-- supabase storage rm contract-originals/tlp/ --recursive

-- Verification:
SELECT COUNT(*) FROM analysis_records WHERE tenant_id = 'tlp';
-- Expected: 0
```

Rollback is safe because:
- No other tables reference `analysis_records.id` yet (signing/negotiation haven't migrated)
- Tenant isolation means deletion won't affect other tenants
- Source data is preserved in frozen MOP backup

**IMPORTANT:** Once @milo/contract-signing migration runs (step 2 in ordering), rolling back analysis becomes destructive (signing_documents.analysis_id FK). Always rollback in reverse order: negotiation → signing → analysis.

## 20. Open Questions for Mark

1. **Test data handling:** Should rows with `profile_used = 'TEST'` be excluded or tagged? (33 total rows is small enough to keep all.)

2. **Storage path restructuring:** Should we add tenant prefix (`tlp/YYYY-MM/file`) for future multi-tenant RLS, or keep flat paths for now? Adding prefix is forward-compatible but requires updating all `original_file_path` values.

3. **contract_issue_resolutions timing:** When should this table's data migrate? Options:
   - Never (PPC-specific, stays in MOP archive)
   - @milo/contract-analysis v0.2.0 (when resolution tracking is designed)
   - Fold into `analysis_records.metadata.resolutions[]` now (denormalized but complete)

4. **publisher_id/buyer_id reconciliation:** Should Coder-2 build an ID mapping table (MOP publisher.id → crm_counterparties.id) before or after this migration? It's not blocking but would enable future `counterparty_id` column on `analysis_records`.

5. **Stale jobs policy:** Should `pending`/`processing` jobs at freeze time be:
   - Skipped entirely (recommended — they'll never complete)
   - Migrated with status forced to `'failed'` + error_message = 'Abandoned during MOP freeze'
   - Kept in backup only

6. **finalized_documents table:** This table exists in MOP but is NOT in either @milo/contract-analysis or @milo/contract-signing target schemas. It stores signed/executed PDFs linked to analyses. Where does it belong? Options:
   - @milo/contract-signing (it tracks signed docs)
   - @milo/contract-analysis (it references analysis_id)
   - New table in v0.2.0 of either package

---

## Appendix A: Columns NOT in Target (dropped or relocated)

| Source Column | Disposition | Rationale |
|---------------|-------------|-----------|
| publisher_id | → metadata.source_publisher_id | No @milo/crm mapping yet |
| buyer_id | → metadata.source_buyer_id | No @milo/crm mapping yet |
| negotiation_stage | → metadata.source_negotiation_stage | Belongs to @milo/contract-negotiation |
| is_finalized | → metadata.source_is_finalized | Belongs to @milo/contract-negotiation |
| extracted_company_name | → metadata.extracted_company_name | Subsumed by counterparty_info.name |

## Appendix B: Target Columns with No Source Equivalent

| Target Column | Default Value | Notes |
|---------------|---------------|-------|
| tenant_id | 'tlp' | New: multi-tenant |
| score | NULL | New: not computed in MOP's version |
| metadata | (populated from folded columns) | New: extensible bag |

## Appendix C: Comparison with Signing/Negotiation Audits

| Aspect | Analysis (this) | Signing | Negotiation |
|--------|-----------------|---------|-------------|
| Source rows | 33 | 109 | ~50 across 3 tables |
| Tables migrated | 2 of 4 | 1 of 1 | 3 of 3 |
| Tables skipped | 2 (dead + vertical) | 0 | 0 |
| PPC columns → metadata | 5 | 3 | 0 |
| Storage files | 22 | 0 | 0 |
| Ordering position | 1st (no deps) | 2nd (needs analysis) | 3rd (needs both) |
| Complexity | MEDIUM | LOW | LOW |
| Risk | LOW (small dataset, clear mapping) | LOW | LOW |
