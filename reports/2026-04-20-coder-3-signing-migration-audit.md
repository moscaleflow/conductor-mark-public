---
directive: "@milo/contract-signing data migration audit"
lane: research
coder: Coder-3
started: 2026-04-20 ~03:00 MDT
completed: 2026-04-20 ~03:45 MDT
---

# @milo/contract-signing Data Migration Audit — 2026-04-20

## 1. Executive Summary

**Migration risk: LOW.** 109 rows in `signing_documents` with clean column mapping. Zero storage files need migration (finalized-contracts and partner-documents buckets were empty at freeze). The 22 files in contract-originals belong to @milo/contract-analysis, not signing.

| Metric | Value |
|--------|-------|
| Source rows (signing_documents) | 109 |
| Source rows (partner_documents) | 0 confirmed in freeze manifest |
| Storage files to migrate | 0 (both target buckets empty) |
| Columns mapped directly | 27 of 30 source columns |
| Columns dropped (PPC-specific) | 3 (entity_id, entity_type, entity_name) |
| Columns added | 3 (tenant_id, voided_at, metadata) |
| Transformation complexity | Low (mostly 1:1 + entity→metadata) |
| Active session risk | Medium (pending/viewed tokens point at old MOP URL) |

**Key decisions needed from Mark:**
1. Are any of the 109 rows test data that should be excluded?
2. Should old MOP signing URLs redirect to new deployment, or let them 404?
3. Does partner_documents have any rows? (Not in freeze manifest row counts — need verification)

## 2. Column Mapping: signing_documents → signing_documents

### Source Schema (MOP, all migrations applied)

Full column set after all 7 ALTER migrations:

| Column | Type | Added In | Not Null | Default |
|--------|------|----------|----------|---------|
| id | UUID | initial | YES | gen_random_uuid() |
| entity_id | UUID | initial | NO | — |
| entity_type | TEXT | initial | NO | CHECK: publisher, buyer |
| entity_name | TEXT | initial | YES | — |
| document_type | TEXT | initial | YES | CHECK: msa, io, w9, w8ben, w8bene, rider |
| status | TEXT | initial | YES | 'pending', CHECK: pending, viewed, signed, voided, declined |
| template_data | JSONB | initial | NO | '{}' |
| generated_pdf_url | TEXT | initial | NO | — |
| signing_token | TEXT | initial | YES | UNIQUE |
| counterparty_name | TEXT | initial | NO | — |
| counterparty_title | TEXT | initial | NO | — |
| counterparty_email | TEXT | initial | NO | — |
| counterparty_company | TEXT | initial | NO | — |
| counterparty_phone | TEXT | 20260213_expansion | NO | — |
| counterparty_address | TEXT | 20260213_expansion | NO | — |
| counterparty_city | TEXT | 20260213_expansion | NO | — |
| counterparty_state | TEXT | 20260213_expansion | NO | — |
| counterparty_zip | TEXT | 20260213_expansion | NO | — |
| counterparty_state_of_incorporation | TEXT | 20260213_expansion | NO | — |
| counterparty_entity_type | TEXT | 20260213_expansion | NO | — |
| signature_data | TEXT | initial | NO | — |
| signature_hash | TEXT | 20260213_auth_audit | NO | — |
| document_hash | TEXT | 20260213_auth_audit | NO | — |
| signature_ip | TEXT | initial | NO | — |
| signer_user_agent | TEXT | 20260213_auth_audit | NO | — |
| consent_text | TEXT | 20260213_auth_audit | NO | — |
| consent_accepted_at | TIMESTAMPTZ | 20260213_auth_audit | NO | — |
| audit_trail | JSONB | 20260213_auth_audit | NO | '[]' |
| rendered_html | TEXT | 20260213_auth_audit | NO | — |
| terms | JSONB | 20260219_terms | NO | — |
| short_code | TEXT | 20260219_short_codes | NO | UNIQUE (backfilled) |
| viewed_at | TIMESTAMPTZ | initial | NO | — |
| signed_at | TIMESTAMPTZ | initial | NO | — |
| declined_at | TIMESTAMPTZ | 20260309_decline | NO | — |
| decline_reason | TEXT | 20260309_decline | NO | — |
| webhook_sent | BOOLEAN | initial | NO | false |
| webhook_url | TEXT | initial | NO | — |
| created_at | TIMESTAMPTZ | initial | NO | NOW() |
| updated_at | TIMESTAMPTZ | initial | NO | NOW() |

### Migration Mapping

| Source Column | Target Column | Transformation | Notes |
|---------------|---------------|----------------|-------|
| id | id | Direct (UUID preserved) | PK preserved for idempotency |
| entity_id | metadata.entity_id | Move to JSONB | Preserved for reverse lookup |
| entity_type | metadata.entity_type | Move to JSONB | 'publisher' or 'buyer' |
| entity_name | metadata.entity_name | Move to JSONB | Company display name |
| document_type | document_type | Direct | No CHECK in target (engine is unconstrained) |
| status | status | Direct | Same values + 'declined' already present |
| template_data | template_data | Direct | JSONB preserved |
| generated_pdf_url | generated_pdf_url | Direct | NULL for most rows (HTML-based signing) |
| signing_token | signing_token | Direct | UNIQUE NOT NULL |
| counterparty_* (11 cols) | counterparty_* (11 cols) | Direct | All 1:1 mapping |
| signature_data | signature_data | Direct | Base64 |
| signature_hash | signature_hash | Direct | SHA-256 |
| document_hash | document_hash | Direct | SHA-256 |
| signature_ip | signature_ip | Direct | |
| signer_user_agent | signer_user_agent | Direct | |
| consent_text | consent_text | Direct | |
| consent_accepted_at | consent_accepted_at | Direct | |
| audit_trail | audit_trail | Direct | JSONB array preserved as-is |
| rendered_html | rendered_html | Direct | |
| terms | terms | Direct | |
| short_code | short_code | Direct | Already backfilled (LEFT(id::text, 8)) |
| viewed_at | viewed_at | Direct | |
| signed_at | signed_at | Direct | |
| declined_at | declined_at | Direct | |
| decline_reason | decline_reason | Direct | |
| webhook_sent | webhook_sent | Direct | |
| webhook_url | webhook_url | Direct | |
| created_at | created_at | Direct | |
| updated_at | updated_at | Direct | |
| — | **tenant_id** | **Set 'tlp'** | All rows belong to TLP tenant |
| — | **voided_at** | **Derive from audit_trail** | If status='voided', extract timestamp from audit_trail event 'document_voided' |
| — | **metadata** | **Construct JSONB** | `{ entity_id, entity_type, entity_name }` |

### Transformation Logic (pseudocode)

```sql
INSERT INTO shared.signing_documents (
  id, tenant_id, document_type, status, template_data, rendered_html,
  terms, generated_pdf_url, signing_token, short_code,
  counterparty_name, counterparty_title, counterparty_email,
  counterparty_company, counterparty_phone, counterparty_address,
  counterparty_city, counterparty_state, counterparty_zip,
  counterparty_state_of_incorporation, counterparty_entity_type,
  signature_data, signature_hash, document_hash, signature_ip,
  signer_user_agent, consent_text, consent_accepted_at,
  audit_trail, viewed_at, signed_at, declined_at, voided_at,
  decline_reason, webhook_sent, webhook_url, metadata,
  created_at, updated_at
)
SELECT
  id,
  'tlp' AS tenant_id,
  document_type,
  status,
  template_data,
  rendered_html,
  terms,
  generated_pdf_url,
  signing_token,
  short_code,
  counterparty_name, counterparty_title, counterparty_email,
  counterparty_company, counterparty_phone, counterparty_address,
  counterparty_city, counterparty_state, counterparty_zip,
  counterparty_state_of_incorporation, counterparty_entity_type,
  signature_data, signature_hash, document_hash, signature_ip,
  signer_user_agent, consent_text, consent_accepted_at,
  audit_trail,
  viewed_at, signed_at, declined_at,
  -- Derive voided_at from audit_trail
  CASE WHEN status = 'voided'
    THEN (audit_trail->>(jsonb_array_length(audit_trail) - 1))::jsonb->>'timestamp'
    ELSE NULL
  END AS voided_at,
  decline_reason, webhook_sent, webhook_url,
  -- Construct metadata JSONB
  jsonb_build_object(
    'entity_id', entity_id,
    'entity_type', entity_type,
    'entity_name', entity_name,
    'source', 'mop_migration',
    'migrated_at', NOW()
  ) AS metadata,
  created_at, updated_at
FROM mop.signing_documents;
```

## 3. Column Mapping: partner_documents → signing_library

| Source Column | Target Column | Transformation | Notes |
|---------------|---------------|----------------|-------|
| id | id | Direct | UUID preserved |
| publisher_id | metadata.entity_id + metadata.entity_type='publisher' | Move to JSONB | |
| buyer_id | metadata.entity_id + metadata.entity_type='buyer' | Move to JSONB | |
| document_type | document_type | Lowercase normalize | MOP uses uppercase ('MSA', 'IO'); engine is case-insensitive |
| document_name | document_name | Direct | |
| storage_path | storage_path | Direct | Bucket path preserved |
| file_size | file_size | Direct | |
| mime_type | mime_type | Direct | |
| contract_analysis_id | analysis_id | Rename only | Shortened column name |
| uploaded_by | uploaded_by | Direct | |
| created_at | created_at | Direct | |
| updated_at | updated_at | Direct | |
| archived_at | archived_at | Direct | |
| — | **tenant_id** | **Set 'tlp'** | |
| — | **metadata** | **Construct JSONB** | `{ entity_id, entity_type, source: 'mop_migration' }` |

**Row count concern:** partner_documents is NOT listed in the freeze manifest row counts. This means either:
- (a) It has 0 rows (possible — partner-documents storage bucket was empty)
- (b) It was omitted from the manifest count

**Verification needed:** Query `SELECT COUNT(*) FROM partner_documents` on frozen MOP, or check the full-backup.sql COPY statement for this table.

## 4. Excluded Columns (Not Migrating)

| Column | Table | Reason |
|--------|-------|--------|
| entity_id | signing_documents | PPC-specific FK → moved to metadata |
| entity_type | signing_documents | PPC-specific CHECK → moved to metadata |
| entity_name | signing_documents | PPC display name → moved to metadata |
| publisher_id | partner_documents | PPC-specific FK → moved to metadata |
| buyer_id | partner_documents | PPC-specific FK → moved to metadata |

These are NOT deleted — they're preserved in the `metadata` JSONB column for reverse lookup and potential @milo/crm counterparty resolution later.

## 5. Data Quality Concerns

### 5.1 Likely Quality Issues (can't verify without query, but predictable from schema)

| Issue | Risk | Detection Query | Mitigation |
|-------|------|-----------------|------------|
| entity_name NOT NULL but entity_id nullable | Low | `WHERE entity_id IS NULL` | entity_name always present; entity_id may be null for early documents before matching |
| audit_trail may be empty '[]' | Low | `WHERE audit_trail = '[]'` | Early documents created before audit trail feature (migration added DEFAULT '[]') |
| counterparty_* fields all NULL for unsigned docs | Expected | `WHERE status = 'pending' AND counterparty_name IS NULL` | Normal — these fields are captured at signing time |
| short_code backfill collision risk | Very Low | `WHERE short_code IN (SELECT short_code FROM signing_documents GROUP BY short_code HAVING COUNT(*) > 1)` | Backfill used LEFT(id::text, 8) which is UUID prefix — collision astronomically unlikely for 109 rows |
| document_type may include values not in original CHECK | None | All 6 types (msa, io, w9, w8ben, w8bene, rider) added via migrations | Target has no CHECK — all values valid |
| rendered_html NULL for early documents | Expected | `WHERE rendered_html IS NULL` | Early docs pre-date auth_audit migration. Certificate generation may fail for these — handle gracefully |
| voided_at derivation may fail | Low | `WHERE status = 'voided' AND NOT audit_trail @> '[{"event": "document_voided"}]'` | If a doc was voided before audit_trail existed, no event to derive from. Fallback: use updated_at |

### 5.2 Test Data Detection

From freeze manifest storage listing, there are test files:
- `contract-originals/__test__/phase4b-test-1770352849069.txt`
- `contract-originals/2026-02/9e1378d1-test-contract.txt`
- `contract-originals/2026-03/4979debc-test-contract.txt`
- `contract-originals/2026-03/b79d8198-TLP_Buyer_MSA_redlined_TestRedline.txt`

These are in the contract-originals bucket (belongs to @milo/contract-analysis, not signing). But corresponding test signing_documents rows likely exist. Detection:

```sql
-- Find test rows
SELECT id, entity_name, document_type, status, created_at
FROM signing_documents
WHERE entity_name ILIKE '%test%'
   OR template_data::text ILIKE '%test%'
   OR entity_name ILIKE '%ContractTestCo%';
```

**Recommendation:** Run detection query, then decide whether to migrate test rows (with a `metadata.is_test = true` flag) or exclude them.

## 6. Storage Bucket Migration Plan

| Bucket | Files at Freeze | Belongs To | Migration Action |
|--------|-----------------|------------|------------------|
| contract-originals | 22 | @milo/contract-analysis | **No action** — not part of signing migration |
| finalized-contracts | 0 | @milo/contract-signing | **No action** — empty bucket, create in shared Supabase |
| partner-documents | 0 | @milo/contract-signing (signing_library) | **No action** — empty bucket, create in shared Supabase |
| partner-creatives | 1 (test file) | MOP vertical | **No action** — not signing-related |

**Net storage migration: Zero files to copy.**

Create buckets in shared Supabase (tappyckcteqgryjniwjg):
```sql
INSERT INTO storage.buckets (id, name, public, file_size_limit, allowed_mime_types)
VALUES
  ('finalized-contracts', 'finalized-contracts', false, 26214400, ARRAY['application/pdf']),
  ('partner-documents', 'partner-documents', false, 26214400, ARRAY['application/pdf'])
ON CONFLICT (id) DO NOTHING;
```

The `generated_pdf_url` column in signing_documents likely contains NULL for all 109 rows (HTML-based signing, no PDF generation). If any rows DO have a URL, it would reference MOP's Supabase storage — those URLs would break after migration unless the files are copied. **Verification needed.**

## 7. Audit Trail Migration Strategy

**Decision: Migrate as-is.** The audit_trail JSONB column is an append-only event log. Its format already matches the @milo/contract-signing `AuditEvent[]` interface:

```typescript
// MOP format (from submit route)
{ event: 'document_signed', timestamp: '2026-03-15T10:30:00.000Z', ip: '...', user_agent: '...' }

// @milo/contract-signing target format
{ event: 'document_signed', timestamp: '2026-03-15T10:30:00.000Z', ip: '...', user_agent: '...' }
```

Same shape. No transformation needed.

**Add migration event:** Append a `migration_completed` event to each row's audit trail during migration:

```sql
UPDATE signing_documents
SET audit_trail = audit_trail || jsonb_build_array(
  jsonb_build_object(
    'event', 'migration_completed',
    'timestamp', NOW()::text,
    'details', jsonb_build_object(
      'source', 'mop_wjxtfjaixkoifdqtfmqd',
      'target', 'shared_tappyckcteqgryjniwjg',
      'migration_version', '1.0.0'
    )
  )
);
```

This preserves the full history and marks the migration point in the audit trail.

## 8. Counterparty Reference Resolution

### Current State

MOP's `entity_id` column references `publishers.id` or `buyers.id`. These are MOP-local UUIDs in the frozen MOP Supabase (wjxtfjaixkoifdqtfmqd).

### Future State

When @milo/crm completes migration, publishers/buyers become `crm_counterparties` rows in shared Supabase (tappyckcteqgryjniwjg). Each counterparty gets a new UUID in the shared database.

### Resolution Strategy

**Phase 1 (now):** Store MOP entity_id in metadata. No resolution needed.

```json
{
  "entity_id": "abc-123-...",
  "entity_type": "publisher",
  "entity_name": "Acme Media",
  "source": "mop_migration"
}
```

**Phase 2 (after @milo/crm migration):** Run a reconciliation pass:

```sql
UPDATE signing_documents sd
SET metadata = sd.metadata || jsonb_build_object(
  'counterparty_id', cc.id
)
FROM crm_counterparties cc
WHERE cc.metadata->>'legacy_mop_id' = sd.metadata->>'entity_id'
  AND sd.tenant_id = 'tlp';
```

This assumes the @milo/crm migration stores the original MOP entity_id in its own metadata (same pattern Coder-2 used for the PPCRM crm-reader adapter).

**No blocking dependency.** Signing migration can proceed independently of CRM migration. The entity linkage resolves later via a reconciliation pass.

## 9. Active Signing Sessions Risk

### The Problem

Some of the 109 rows may have `status = 'pending'` or `status = 'viewed'`. These represent documents that:
1. Were sent to a counterparty for signing
2. Have an active signing_token
3. The signer has a URL: `https://tlpmop.netlify.app/sign/[token]`

### During Freeze

MOP is frozen (`MOP_WRITES_FROZEN=true`). A signer clicking the link can:
- **VIEW** the document (GET requests pass through freeze middleware) ✓
- **SIGN** the document (POST blocked by freeze middleware) ✗

So active sessions are in a "viewable but not signable" state.

### After Migration

If the signing system moves to a new deployment (e.g., `signing.scaleflow.co/sign/[token]`):
- Old `tlpmop.netlify.app/sign/[token]` URLs would either 404 or hit the frozen POST blocker
- Signers with bookmarked/emailed links lose access

### Mitigation Options

| Option | Effort | Trade-off |
|--------|--------|-----------|
| **A: Redirect middleware on MOP** | Low (1 hour) | Add a `/sign/[token]` redirect route to MOP that 301s to new URL. MOP stays frozen but redirects signing traffic. |
| **B: Keep MOP signing routes live** | Zero | Don't migrate active sessions. Let them complete on MOP (unfreeze signing POST routes only). Migrate only terminal-state rows. |
| **C: Notify counterparties** | Medium | Send new signing links from new system. Void old tokens. |
| **D: Accept link breakage** | Zero | If pending docs are old/stale (months), counterparties likely aren't going to sign anyway. Archive them. |

**Recommendation:** Option B for any truly active sessions (check `status IN ('pending', 'viewed') AND created_at > NOW() - INTERVAL '30 days'`). Option D for stale pending docs (older than 30 days = expired per engine's default TTL). Migrate all terminal-state rows (signed, voided, declined) immediately.

### Pre-Flight Query

```sql
-- Active session inventory
SELECT status, COUNT(*),
  MIN(created_at) AS oldest,
  MAX(created_at) AS newest
FROM signing_documents
WHERE status IN ('pending', 'viewed')
GROUP BY status;

-- Recently active (potential live sessions)
SELECT id, entity_name, document_type, status, created_at
FROM signing_documents
WHERE status IN ('pending', 'viewed')
  AND created_at > NOW() - INTERVAL '30 days';
```

## 10. HMAC Format — Migration Impact

### Historical Context

MOP's submit route uses `signBody()` (correct: `HMAC(timestamp.body)` with Unix epoch). MOP's decline route hand-rolls HMAC (broken: `HMAC(body)` with ISO timestamp). This is the bug identified in the signing audit.

### Migration Impact: None

- **Already-fired webhooks** (webhook_sent = true): Historical. Format cannot be retroactively changed. PPCRM consumers already received these webhooks (correctly for signs, incorrectly for declines).
- **Future events** on migrated rows: If a migrated document gets voided post-migration, the engine fires the webhook with the correct unified format.
- **No re-signing needed.** Webhook signatures are not stored in the database — they're computed at fire time. Migrated data doesn't carry webhook signatures.

**The HMAC bug is fixed going forward by the engine's unified `signPayload()` method. No data migration action needed.**

## 11. Idempotency Strategy

Same pattern as prior data migrations (Coder-2 PPCRM → @milo/crm):

### Pre-Flight Check

```sql
-- Count already-migrated rows in target
SELECT COUNT(*) AS already_migrated
FROM shared.signing_documents
WHERE tenant_id = 'tlp'
  AND metadata->>'source' = 'mop_migration';
```

### Upsert Guard

```sql
INSERT INTO shared.signing_documents (...)
SELECT ... FROM mop.signing_documents
ON CONFLICT (id) DO NOTHING;
-- OR: DO UPDATE SET updated_at = EXCLUDED.updated_at (for re-runs)
```

### Post-Migration Verification

```sql
-- Verify row counts match
SELECT
  (SELECT COUNT(*) FROM mop.signing_documents) AS source_count,
  (SELECT COUNT(*) FROM shared.signing_documents WHERE tenant_id = 'tlp' AND metadata->>'source' = 'mop_migration') AS target_count;

-- Verify no data loss (spot check critical fields)
SELECT s.id, s.signing_token, s.status
FROM mop.signing_documents s
LEFT JOIN shared.signing_documents t ON t.id = s.id
WHERE t.id IS NULL;
```

### Rollback

```sql
-- Clean rollback (remove all migrated rows)
DELETE FROM shared.signing_documents
WHERE tenant_id = 'tlp'
  AND metadata->>'source' = 'mop_migration';
```

## 12. Proposed --dry-run Output Shape

```json
{
  "migration": "@milo/contract-signing",
  "version": "1.0.0",
  "source": {
    "project": "wjxtfjaixkoifdqtfmqd",
    "table": "signing_documents",
    "row_count": 109
  },
  "target": {
    "project": "tappyckcteqgryjniwjg",
    "table": "signing_documents",
    "tenant_id": "tlp"
  },
  "plan": {
    "rows_to_migrate": 105,
    "rows_excluded_test": 4,
    "rows_excluded_active": 0,
    "rows_already_migrated": 0,
    "storage_files_to_copy": 0,
    "transformations": [
      { "column": "entity_id → metadata.entity_id", "affected": 109 },
      { "column": "entity_type → metadata.entity_type", "affected": 109 },
      { "column": "entity_name → metadata.entity_name", "affected": 109 },
      { "column": "voided_at (derived)", "affected": "N (rows with status=voided)" },
      { "column": "tenant_id (set 'tlp')", "affected": 109 }
    ],
    "quality_warnings": [
      { "issue": "audit_trail empty", "count": "N", "severity": "low" },
      { "issue": "rendered_html null", "count": "N", "severity": "low" },
      { "issue": "generated_pdf_url references MOP storage", "count": "N", "severity": "medium" }
    ]
  },
  "sample_row": {
    "id": "abc-123...",
    "tenant_id": "tlp",
    "document_type": "msa",
    "status": "signed",
    "signing_token": "def-456...",
    "short_code": "abc12345",
    "counterparty_name": "Jane Doe",
    "metadata": {
      "entity_id": "old-mop-uuid...",
      "entity_type": "publisher",
      "entity_name": "Acme Media",
      "source": "mop_migration",
      "migrated_at": "2026-04-20T..."
    },
    "audit_trail": [
      { "event": "document_created", "timestamp": "..." },
      { "event": "document_signed", "timestamp": "..." },
      { "event": "migration_completed", "timestamp": "...", "details": { "source": "mop_wjxtfjaixkoifdqtfmqd" } }
    ]
  }
}
```

## 13. Tenant Assignment

**All 109 rows → tenant_id = 'tlp'.**

MOP is a single-tenant application serving The Lead Penguin. There's no scenario where a MOP signing_document belongs to a different tenant. The multi-tenant migration (`20300101000000_multi_tenant_rls.sql`) exists but was never applied — all data is TLP's.

**Archive candidates:** Test rows (entity_name containing "test", "TestCo", "ContractTestCo") should be flagged. Options:
1. Migrate with `metadata.is_test = true` (preserves for reference)
2. Exclude from migration entirely
3. Migrate to a separate `tenant_id = 'tlp_test'` (cleanest separation)

**Recommendation:** Option 1 (migrate with flag). Test rows are harmless in the target DB and useful for verifying the migration pipeline works correctly.

## 14. Migration Execution Order

```
1. Create target tables in shared Supabase (signing_documents, signing_library)
2. Create storage buckets (finalized-contracts, partner-documents)
3. Run pre-flight checks (count source, count already-migrated)
4. --dry-run: produce plan JSON without writing
5. Migrate terminal-state rows (status IN: signed, voided, declined)
6. Migrate pending/viewed rows (per Mark's decision on active sessions)
7. Append migration_completed event to all audit trails
8. Post-migration verification (row count, spot check, signing_token uniqueness)
9. If partner_documents has rows: run signing_library migration
10. Update MOP freeze to redirect /sign/[token] to new URL (if needed)
```

## 15. Open Questions for Mark

1. **partner_documents row count?** Not in freeze manifest. If empty (0 rows), signing_library migration is a no-op. If populated, need the same column mapping treatment. Can verify with: `SELECT COUNT(*) FROM partner_documents` on frozen MOP.

2. **Test data disposition?** Migrate test rows with `metadata.is_test = true`, or exclude entirely? The freeze manifest shows at least 4 test files in storage, suggesting test signing_documents rows exist.

3. **Active session handling?** How many pending/viewed docs exist, and what's Mark's preference? Option B (keep MOP signing routes live for active sessions) or Option D (accept link breakage for stale pending docs)?

4. **generated_pdf_url verification?** If any of the 109 rows have a non-null `generated_pdf_url` pointing to MOP's Supabase storage, those files need copying. Likely all NULL (HTML-based signing), but needs verification.

5. **Timing vs @milo/crm migration?** Signing migration can run independently (entity linkage resolves later via reconciliation pass). But should it wait for @milo/crm so counterparty_id is immediately available in metadata, or proceed now with just the legacy entity_id?

6. **New deployment URL?** What will the signing URL be for TLP going forward? `signing.scaleflow.co`? `milo-ops...`? This determines whether old MOP URLs need redirect middleware.

7. **Webhook URL update?** The `webhook_url` column on migrated rows still points at PPCRM's webhook endpoint. After migration, should these be updated to point at a new consumer, or left as historical references?
