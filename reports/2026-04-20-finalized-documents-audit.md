---
directive: "finalized_documents table disposition audit"
lane: ops (consolidated from Coder-3 + Coder-4 duplicate audits)
coder: Coder-3 (research), Coder-4 (consolidation)
started: 2026-04-20
completed: 2026-04-20
note: Duplicate audit due to lane routing error. Consolidated into single canonical report.
---

# finalized_documents Table Disposition Audit

## 1. Executive Summary

**Recommendation: (e) Dead table — skip migration entirely.**

`finalized_documents` has **0 rows**, **0 storage files**, and its CRUD functions are defined but **never called** by any API route, UI component, or background job. The table was created as part of the contract versioning migration (Dec 2024) but the feature was never completed — no upload UI, no API endpoints, no signed URL generation exists. The storage bucket (`finalized-contracts`) is empty.

No data migration needed. No schema extension needed. If the concept (archive of finalized PDFs) is wanted in the future, it should be designed fresh as a @milo/contract-signing v0.2.0 feature — not inherited from dead MOP code.

---

## 2. Schema Inventory

**Migration:** `20241219060000_contract_version_tracking.sql`
**Location:** `/Users/markymark/Mop3.18/MOP/supabase/migrations/`

| Column | Type | Null | Default | Notes |
|--------|------|------|---------|-------|
| id | UUID | NO | gen_random_uuid() | PK |
| analysis_id | UUID | NO | — | FK → contract_analyses(id) ON DELETE CASCADE |
| document_name | VARCHAR(255) | NO | — | Display name |
| storage_path | VARCHAR(500) | NO | — | Path in finalized-contracts bucket |
| file_size | INTEGER | YES | NULL | Bytes |
| mime_type | VARCHAR(100) | YES | NULL | application/pdf, image/png, image/jpeg |
| upload_type | VARCHAR(20) | NO | 'signed' | CHECK: 'signed', 'executed', 'amendment' |
| notes | TEXT | YES | NULL | |
| uploaded_by | VARCHAR(100) | YES | NULL | |
| uploaded_at | TIMESTAMPTZ | NO | NOW() | |
| created_at | TIMESTAMPTZ | NO | NOW() | |
| tenant_id | UUID | YES | — | FK → tenants(id) — from unapplied migration 20300101000000 |

**Indexes:**
- `idx_finalized_documents_analysis` ON (analysis_id)
- `idx_finalized_documents_type` ON (upload_type)

**RLS:** Enabled, permissive `USING(true)` policy (overly permissive — no tenant isolation). Multi-tenant draft exists in `20300101000000_multi_tenant_rls.sql` but was never applied.

**Storage bucket:** `finalized-contracts` — 50MB max, PDF/PNG/JPEG only, private. **Empty.**

**Row count: 0.** Table is empty in frozen MOP.

---

## 3. Usage Analysis

### Code Paths IN (writes)

| Function | File | Status |
|----------|------|--------|
| `saveFinalizedDocument()` | lib/contract-checker/supabase.ts:769-812 | **Exported, never called** |

The function does:
1. INSERT into finalized_documents
2. Side-effect: calls `updateNegotiationStage(analysisId, 'finalized')` if upload_type is 'signed' or 'executed'

No API route, UI component, or background job ever calls `saveFinalizedDocument()`.

### Code Paths OUT (reads)

| Function | File | Status |
|----------|------|--------|
| `getFinalizedDocuments()` | lib/contract-checker/supabase.ts:817-835 | **Exported, never called** |
| `deleteFinalizedDocument()` | lib/contract-checker/supabase.ts:840-857 | **Exported, never called** |

### Missing Integration Points

| Expected Component | Status |
|-------------------|--------|
| Upload UI (drag-and-drop or file picker) | Does not exist |
| API route (`/api/contract/finalized`) | Does not exist |
| File upload handler (bucket write) | Does not exist |
| Signed URL generator (for download) | Does not exist |
| History page integration | Mentioned in ARCHITECTURE_ANSWERS.md but not implemented |

### What actually stores signed documents

The `signing_documents` table handles the complete signing lifecycle. When a document is signed, its record is updated in-place (`signed_at`, `signature_data`, `counter_signed_at`). There is no separate "finalized" copy — the signed state IS the final state.

### Type Mismatch (abandoned design signal)

The public `FinalizedDocument` type (contract-checker.ts:38-47) uses `revision_id` and `document_url`, but the schema/internal type uses `analysis_id` and `storage_path`. This mismatch confirms the feature was designed early and never reconciled with the actual implementation.

---

## 4. Data State

| Metric | Value |
|--------|-------|
| Table rows | **0** |
| Storage bucket files | **0** |
| Functions defined | 3 (save, get, delete) |
| Functions called | **0** |
| API routes | **0** |
| UI components | **0** |

---

## 5. FK Relationships

**Outbound:**
- `analysis_id` → `contract_analyses(id)` ON DELETE CASCADE
- `tenant_id` → `tenants(id)` (from unapplied future migration)

**Inbound:**
- None. No table references finalized_documents. Completely orphaned.

---

## 6. Placement Options Evaluation

### (a) @milo/contract-signing — Extend SCHEMA.md v0.2.0

**Argument for:** The concept (archive of signed PDFs) is thematically signing-adjacent. The `upload_type` values ('signed', 'executed', 'amendment') are all post-signature states. This is the natural future home.
**Argument against:** The table was never used. The signing_documents table already stores the final state. Adding dead schema to a shipped package creates maintenance burden with zero value. Also, a proper design would FK to `signing_documents(id)` not `contract_analyses(id)`.
**Verdict:** Future home when feature is actually needed, but not now. Redesign from scratch.

### (b) @milo/contract-negotiation — Extend SCHEMA.md

**Argument for:** `analysis_id` FK and the `updateNegotiationStage('finalized')` side-effect suggest negotiation linkage.
**Argument against:** Negotiation already has a finalization flow that produces an agreement HTML (via `generateAgreement()`). This table stores file uploads, not generated documents. The side-effect was aspirational, never executed.
**Verdict:** No.

### (c) New primitive @milo/document-archive

**Argument for:** Generic document storage is a real capability.
**Argument against:** Creating a primitive from a dead table with 0 rows and 0 callers is inventing work. If document archival is needed later, it should be designed from actual requirements, not inherited dead code.
**Verdict:** No.

### (d) Vertical-layer only (Milo-for-PPC owns it)

**Argument for:** If PPC operators need a place to upload executed PDFs post-signing.
**Argument against:** The concept isn't PPC-specific. Any vertical that generates signed contracts would need this. But since it has 0 rows, there's nothing vertical-specific to keep.
**Verdict:** Maybe future, but not now.

### (e) Dead table — skip migration entirely

**Argument for:** 0 rows. 0 callers. Dead CRUD functions. Never integrated. Type mismatch indicates abandoned mid-design. The signing flow already handles document state via signing_documents. No data loss risk (there's no data). No functional loss (nothing uses it).
**Argument against:** None.
**Verdict:** Yes. Recommended.

---

## 7. Migration Impact

**Impact of skipping: Zero.**
- No rows to lose
- No active code paths disrupted
- No downstream consumers reading from it
- Signing flow unaffected (already uses signing_documents)
- The signing audit and analysis audit both correctly excluded this table

**If revisited later:** Design a proper `signing_attachments` table for @milo/contract-signing v0.2.0 with:
- `tenant_id` (multi-tenant from day one)
- `signing_document_id` FK (links to the signing record, not analysis)
- Proper storage bucket with tenant-scoped paths (`{tenant_id}/{signing_document_id}/{filename}`)
- Upload + download API
- 1:N relationship (original + signed + countersigned per signing record)

---

## 8. Open Questions for Mark

1. **Confirm skip?** Both auditors independently recommend skip. If Mark has context suggesting operators will need post-signing PDF upload soon, the feature should be designed fresh as a v0.2.0 schema amendment (Decision 38 process applies).

2. **Storage bucket disposition:** The `finalized-contracts` Supabase storage bucket exists in MOP but is empty. Should it be cleaned up during eventual MOP decommission, or left alone?

3. **When @milo/contract-signing gets file attachment support, should it:**
   - (a) Store file metadata in the existing `signing_documents` table (add storage columns)
   - (b) Create a separate `signing_attachments` table (1:N from signing_documents)
   - Recommendation: (b) — separates document state from file storage, supports multiple attachments per signing record.

---

## Appendix: Evidence Trail

```
Searches performed (combined from both auditors):
- "finalized_documents" across ~/Mop3.18/MOP/ — 3 files (migration, types, supabase.ts)
- "finalized-contracts" across ~/Mop3.18/MOP/ — 2 files (migration bucket creation, storage policies)
- "FinalizedDocument" across ~/Mop3.18/MOP/ — 2 files (types, supabase.ts)
- "saveFinalizedDocument" across ~/Mop3.18/MOP/ — 1 file (supabase.ts definition only, 0 call sites)
- Freeze manifest row counts — finalized_documents omitted (0 rows)
- Freeze manifest storage files — "finalized-contracts bucket: empty" confirmed
- Type mismatch: revision_id/document_url (public type) vs analysis_id/storage_path (schema)
```
