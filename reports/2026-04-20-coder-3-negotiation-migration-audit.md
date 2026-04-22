---
directive: "@milo/contract-negotiation data migration audit"
lane: research
coder: Coder-3
started: 2026-04-20 ~04:00 MDT
completed: 2026-04-20 ~04:30 MDT
---

# @milo/contract-negotiation Data Migration Audit — 2026-04-20

## 1. Executive Summary

**Migration risk: LOW.** Three tables (contract_negotiations, negotiation_rounds, negotiation_links) with near-1:1 column mapping. No PPC-specific entity columns to dismantle (unlike signing). The primary complexity is the `analysis_id` FK ordering dependency — `@milo/contract-analysis` migration MUST complete first.

| Metric | Value |
|--------|-------|
| Source tables | 3 (contract_negotiations, negotiation_rounds, negotiation_links) |
| Source rows (est.) | ≤33 negotiations (upper bound = contract_analyses count), likely fewer |
| Columns mapped directly | 30 of 33 source columns across all 3 tables |
| Columns added | 4 (tenant_id x3, cancelled_at x1, metadata x1) |
| Columns dropped | 0 |
| Transformation complexity | Very Low (tenant_id assignment + metadata construction) |
| FK dependency | analysis_id → analysis_records (must migrate analysis first) |
| Active link risk | Medium (review tokens point at old MOP URL) |
| rider_templates | NOT migrating (PPC-specific, stays in vertical) |

**Row count note:** The freeze manifest doesn't list negotiation tables explicitly (only the top ~18 tables by size). Upper bound is 33 (one negotiation per analysis). Verification query needed before migration.

## 2. Column Mapping: contract_negotiations → negotiation_records

### Source Schema (MOP)

| Column | Type | Not Null | Default | Notes |
|--------|------|----------|---------|-------|
| id | UUID | YES | gen_random_uuid() | PK |
| analysis_id | UUID | YES | — | FK → contract_analyses(id) ON DELETE CASCADE |
| status | TEXT | YES | 'draft' | CHECK: draft, broker_review, counterparty_review, finalized, cancelled |
| document_type | TEXT | YES | — | CHECK: msa, io, rider, amendment |
| counterparty_name | TEXT | YES | — | |
| counterparty_email | TEXT | NO | — | |
| broker_decisions | JSONB | NO | '[]' | |
| current_round | INTEGER | NO | 0 | |
| created_by | TEXT | NO | 'mop' | |
| webhook_url | TEXT | NO | — | |
| created_at | TIMESTAMPTZ | NO | NOW() | |
| updated_at | TIMESTAMPTZ | NO | NOW() | |
| finalized_at | TIMESTAMPTZ | NO | — | |

### Mapping Table

| Source Column | Target Column | Transformation | Notes |
|---------------|---------------|----------------|-------|
| id | id | Direct | UUID preserved (idempotency key) |
| analysis_id | analysis_id | **FK retarget** | Was → contract_analyses(id), now → analysis_records(id). Same UUID since analysis migration preserves IDs. |
| status | status | Direct | All MOP values valid in engine |
| document_type | document_type | Direct | Engine removes CHECK (unconstrained TEXT) |
| counterparty_name | counterparty_name | Direct | |
| counterparty_email | counterparty_email | Direct | |
| broker_decisions | broker_decisions | Direct | JSONB array preserved |
| current_round | current_round | Direct | |
| created_by | created_by | **Map 'mop' → 'system'** | Or preserve as-is for provenance |
| webhook_url | webhook_url | Direct | |
| created_at | created_at | Direct | |
| updated_at | updated_at | Direct | |
| finalized_at | finalized_at | Direct | |
| — | **tenant_id** | **Set 'tlp'** | All rows TLP |
| — | **cancelled_at** | **Derive** | If status='cancelled', use updated_at as proxy |
| — | **metadata** | **Construct** | `{ source: 'mop_migration', migrated_at: NOW() }` |

### Transformation SQL

```sql
INSERT INTO shared.negotiation_records (
  id, tenant_id, analysis_id, status, document_type,
  counterparty_name, counterparty_email, broker_decisions,
  current_round, created_by, webhook_url,
  finalized_at, cancelled_at, metadata,
  created_at, updated_at
)
SELECT
  id,
  'tlp' AS tenant_id,
  analysis_id,  -- same UUID, now references analysis_records
  status,
  document_type,
  counterparty_name,
  counterparty_email,
  broker_decisions,
  current_round,
  CASE WHEN created_by = 'mop' THEN 'system' ELSE created_by END,
  webhook_url,
  finalized_at,
  CASE WHEN status = 'cancelled' THEN updated_at ELSE NULL END AS cancelled_at,
  jsonb_build_object(
    'source', 'mop_migration',
    'migrated_at', NOW(),
    'original_created_by', created_by
  ) AS metadata,
  created_at,
  updated_at
FROM mop.contract_negotiations;
```

## 3. Column Mapping: negotiation_rounds → negotiation_rounds

### Source Schema (MOP)

| Column | Type | Not Null | Default | Notes |
|--------|------|----------|---------|-------|
| id | UUID | YES | gen_random_uuid() | PK |
| negotiation_id | UUID | YES | — | FK → contract_negotiations(id) ON DELETE CASCADE |
| round_number | INTEGER | YES | — | UNIQUE(negotiation_id, round_number) |
| actor_type | TEXT | YES | — | CHECK: broker, counterparty |
| actor_name | TEXT | NO | — | |
| actor_ip | TEXT | NO | — | |
| actor_user_agent | TEXT | NO | — | |
| decisions | JSONB | YES | '[]' | |
| overall_notes | TEXT | NO | — | |
| submitted_at | TIMESTAMPTZ | NO | NOW() | |

### Mapping Table

| Source Column | Target Column | Transformation | Notes |
|---------------|---------------|----------------|-------|
| id | id | Direct | UUID preserved |
| negotiation_id | negotiation_id | Direct | Same UUID, FK retargets to negotiation_records |
| round_number | round_number | Direct | |
| actor_type | actor_type | Direct | Same CHECK values |
| actor_name | actor_name | Direct | May contain "TLP Compliance" — historical data preserved |
| actor_ip | actor_ip | Direct | |
| actor_user_agent | actor_user_agent | Direct | |
| decisions | decisions | Direct | JSONB array preserved |
| overall_notes | overall_notes | Direct | |
| submitted_at | submitted_at | Direct | |
| — | **tenant_id** | **Set 'tlp'** | |

**Zero transformations except tenant_id.** Pure 1:1 plus the new column.

```sql
INSERT INTO shared.negotiation_rounds (
  id, tenant_id, negotiation_id, round_number, actor_type,
  actor_name, actor_ip, actor_user_agent, decisions,
  overall_notes, submitted_at
)
SELECT
  id, 'tlp', negotiation_id, round_number, actor_type,
  actor_name, actor_ip, actor_user_agent, decisions,
  overall_notes, submitted_at
FROM mop.negotiation_rounds;
```

## 4. Column Mapping: negotiation_links → negotiation_links

### Source Schema (MOP, post-20260227 migration)

| Column | Type | Not Null | Default | Notes |
|--------|------|----------|---------|-------|
| id | UUID | YES | gen_random_uuid() | PK |
| negotiation_id | UUID | YES | — | FK → contract_negotiations(id) ON DELETE CASCADE |
| token | TEXT | YES | — | UNIQUE |
| short_code | TEXT | NO | — | UNIQUE (added 20260227, backfilled) |
| link_type | TEXT | YES | — | CHECK: review, final_approval |
| round_number | INTEGER | YES | 1 | |
| expires_at | TIMESTAMPTZ | NO | — | Always NULL in MOP (links never expire) |
| first_viewed_at | TIMESTAMPTZ | NO | — | |
| last_viewed_at | TIMESTAMPTZ | NO | — | |
| view_count | INTEGER | NO | 0 | |
| created_at | TIMESTAMPTZ | NO | NOW() | |

### Mapping Table

| Source Column | Target Column | Transformation | Notes |
|---------------|---------------|----------------|-------|
| id | id | Direct | UUID preserved |
| negotiation_id | negotiation_id | Direct | Same UUID |
| token | token | Direct | UNIQUE NOT NULL |
| short_code | short_code | Direct | Already backfilled from LEFT(id::text, 8) |
| link_type | link_type | Direct | Same CHECK values |
| round_number | round_number | Direct | |
| expires_at | expires_at | **Set 7-day default for active links** | See discussion below |
| first_viewed_at | first_viewed_at | Direct | |
| last_viewed_at | last_viewed_at | Direct | |
| view_count | view_count | Direct | |
| created_at | created_at | Direct | |
| — | **tenant_id** | **Set 'tlp'** | |

### expires_at Strategy

MOP stores `expires_at = NULL` on all links (links never expire — bug #4). Two options:

**Option A: Preserve NULL (safest).** Migrated links keep NULL. Only NEW links get 7-day default. Avoids accidentally expiring active links during migration.

**Option B: Backfill for stale links.** Set `expires_at = created_at + INTERVAL '7 days'` for links whose negotiation is in a terminal state (finalized/cancelled). Leave active links (broker_review/counterparty_review) with NULL.

**Recommendation: Option A.** Don't retroactively expire links. The engine's 7-day default applies to newly generated links only.

## 5. State Machine Migration Plan

### MOP's Status Distribution (Predicted)

MOP's create route inserts directly as `broker_review` (skipping `draft`). So existing rows should have statuses in:

| Status | Predicted Rows | Valid in Engine? | Notes |
|--------|----------------|-----------------|-------|
| draft | 0 | Yes | Create skips this in MOP, but engine allows it |
| broker_review | Some | Yes | Active negotiations awaiting broker action |
| counterparty_review | Some | Yes | Sent to counterparty, awaiting response |
| finalized | Some | Yes | Terminal — all parties agreed |
| cancelled | Some | Yes | Terminal — abandoned |

**All MOP statuses are valid in the engine.** No status transformation needed. The engine's state machine simply enforces going forward that transitions follow the allowed paths.

### Ambiguous State Risk

MOP routes implement ad-hoc guards instead of using the state machine. This means rows could theoretically be in states that the state machine wouldn't allow reaching. For example:
- A row in `counterparty_review` with `current_round = 0` (no round ever sent — shouldn't be possible)
- A `finalized` row with no `finalized_at` timestamp

**Detection queries:**

```sql
-- Counterparty review but no rounds sent
SELECT * FROM contract_negotiations
WHERE status = 'counterparty_review'
AND current_round = 0;

-- Finalized but no finalized_at
SELECT * FROM contract_negotiations
WHERE status = 'finalized'
AND finalized_at IS NULL;

-- Cancelled but has active links
SELECT cn.id, nl.token
FROM contract_negotiations cn
JOIN negotiation_links nl ON nl.negotiation_id = cn.id
WHERE cn.status = 'cancelled'
AND nl.first_viewed_at IS NULL;
```

**Mitigation:** Run detection queries before migration. Fix any anomalous rows in MOP's dump before inserting into shared Supabase. If rows are anomalous but non-critical, migrate with a `metadata.state_anomaly = true` flag.

## 6. analysis_id FK Resolution — Ordering

### The Dependency Chain

```
@milo/contract-analysis migration (FIRST)
  └── contract_analyses → analysis_records (preserves UUIDs)
       └── analysis_id FK in negotiation_records → analysis_records(id)

@milo/contract-negotiation migration (SECOND)
  └── contract_negotiations → negotiation_records
  └── negotiation_rounds → negotiation_rounds  
  └── negotiation_links → negotiation_links
```

### Why This Ordering Matters

The target `negotiation_records.analysis_id` has a hard FK constraint:
```sql
analysis_id UUID NOT NULL REFERENCES analysis_records(id)
```

If `analysis_records` doesn't contain the referenced row, the INSERT fails with a foreign key violation.

### Ordering Guarantee

1. **@milo/contract-analysis migration** runs first, inserting all 33 `contract_analyses` rows into `analysis_records` with preserved UUIDs.
2. **@milo/contract-negotiation migration** runs second, inserting negotiation rows whose `analysis_id` now resolves against `analysis_records`.

### Pre-Flight Check

```sql
-- Verify all analysis_ids in negotiations exist in target
SELECT cn.analysis_id
FROM mop.contract_negotiations cn
LEFT JOIN shared.analysis_records ar ON ar.id = cn.analysis_id
WHERE ar.id IS NULL;
-- Must return 0 rows before negotiation migration proceeds
```

### What If Analysis Migration Hasn't Run?

**Hard block.** The negotiation migration script MUST check for this and abort with a clear error:

```typescript
const missingAnalyses = await checkMissingAnalyses(sourceIds);
if (missingAnalyses.length > 0) {
  throw new Error(
    `Cannot migrate negotiations: ${missingAnalyses.length} analysis_ids not found in analysis_records. Run @milo/contract-analysis migration first.`
  );
}
```

## 7. Active Links Risk

### The Problem

`negotiation_links` contain tokens for URLs like `https://tlpmop.netlify.app/review/[token]`. If a counterparty has a bookmarked review link, it points at frozen MOP.

### Current Behavior During Freeze

- **GET /api/review/[token]** — READ request, passes through freeze middleware. Counterparty can VIEW the negotiation.
- **POST /api/review/[token]/submit** — WRITE request, blocked by `MOP_WRITES_FROZEN=true`. Counterparty CANNOT submit their round.

### Status-Based Strategy

| Negotiation Status | Link Risk | Action |
|--------------------|-----------|----|
| finalized | None | Terminal. Links are informational only. Migrate freely. |
| cancelled | None | Terminal. Links are dead. Migrate freely. |
| broker_review | Low | Counterparty can't access (link is for broker). No external URLs active. |
| counterparty_review | **Medium** | Active review link exists. Counterparty may have URL. |

### Mitigation (same pattern as signing)

For negotiations in `counterparty_review`:
- **Option A: Redirect middleware.** Add `/review/[token]` redirect on MOP → new deployment.
- **Option B: Keep MOP review routes live.** Unfreeze GET + POST for review routes only.
- **Option C: Re-send link.** Generate new link from engine, email counterparty.
- **Option D: Accept breakage.** If negotiations are stale (created > 30 days ago), counterparty likely isn't responding.

**Recommendation:** Check how many negotiations are in `counterparty_review`. If 0-2, option C (manual re-send). If more, option A (redirect).

## 8. Round Decisions Preservation

### Decision JSONB Structure (MOP)

```json
[
  { "issue_id": 1, "status": "accepted" },
  { "issue_id": 2, "status": "rejected", "notes": "Not applicable to our agreement" },
  { "issue_id": 3, "status": "edited", "edited_text": "Revised clause text here" },
  { "issue_id": 4, "status": "counter", "edited_text": "Counter-proposal", "notes": "We suggest..." }
]
```

### Target Decision Interface

```typescript
interface NegotiationDecision {
  issue_id: number
  status: 'accepted' | 'rejected' | 'edited' | 'counter' | 'pending'
  edited_text?: string
  notes?: string
}
```

**Same shape.** MOP's decision JSONB matches the engine's `NegotiationDecision` interface exactly. The field names, value enums, and optional fields are all identical.

**No transformation needed.** Both `broker_decisions` (on negotiation_records) and `decisions` (on negotiation_rounds) migrate as-is.

### actor_name Preservation

MOP hardcodes "TLP Compliance" as the broker actor_name. Migrated rows retain this value as historical data. Going forward, the engine uses `config.defaultActorName` (e.g., "Compliance Team" — TLP can set whatever they want via vertical config).

## 9. Webhook Impact

### MOP Behavior

MOP's `negotiation-webhooks.ts` fires webhooks on:
- `negotiation.created`
- `negotiation.round_submitted`
- `negotiation.finalized`
- `negotiation.cancelled`

URL resolution: `negotiation.webhook_url || process.env.PPCRM_WEBHOOK_URL`

Fire-and-forget (no retry — bug #7).

### Migration Impact

- **Historical webhooks:** Already fired. No action needed.
- **webhook_url column:** Migrated as-is. Contains the URL that was configured when the negotiation was created (likely `PPCRM_WEBHOOK_URL` value or NULL).
- **Future events on migrated rows:** If a migrated negotiation gets finalized post-migration, the engine fires the webhook to the stored `webhook_url` using its retry mechanism (3 attempts, HMAC-signed). If the PPCRM webhook consumer is offline/decommissioned, the retries will fail gracefully (4xx = no retry).
- **No re-firing needed.** Historical webhooks don't need re-delivery.

### Potential Issue

If `webhook_url` references an environment-specific URL (e.g., `https://ppcrm-supabase.co/functions/v1/some-handler`), that endpoint may not exist post-PPCRM-freeze. For terminal-state negotiations (finalized/cancelled), this is irrelevant — no future events will fire. For active negotiations (broker_review/counterparty_review), the webhook URL should be updated to the new consumer endpoint.

```sql
-- Update webhook URLs for active negotiations (if new consumer exists)
UPDATE negotiation_records
SET webhook_url = 'https://new-consumer.example.com/webhooks/negotiation'
WHERE status IN ('broker_review', 'counterparty_review')
  AND webhook_url IS NOT NULL
  AND tenant_id = 'tlp';
```

## 10. rider_templates — NOT Migrating

| Column | Type | Notes |
|--------|------|-------|
| id | UUID | PK |
| name | TEXT | e.g., "Standard Compliance Rider" |
| description | TEXT | |
| category | TEXT | default 'general' |
| clauses | JSONB | Array of clause objects |
| created_by | TEXT | default 'TLP Compliance' |
| created_at | TIMESTAMPTZ | |
| updated_at | TIMESTAMPTZ | |
| usage_count | INTEGER | |
| is_archived | BOOLEAN | |

**Why not migrating:**
1. Not part of @milo/contract-negotiation's schema (SCHEMA.md doesn't include it)
2. PPC-specific content (rider clauses for TLP compliance agreements)
3. "TLP Compliance" branding hardcoded in `created_by` default
4. The engine's template system is injected via config, not stored in DB

**Where it goes:** Stays in PPC vertical layer. If TLP needs rider templates post-migration, they store them in their own vertical's config or a vertical-owned table.

## 11. Idempotency Guard

### Pre-Flight Checks

```sql
-- 1. Verify analysis_records populated (FK dependency)
SELECT COUNT(*) AS analysis_count FROM shared.analysis_records WHERE tenant_id = 'tlp';
-- Must be >= count of distinct analysis_ids in source negotiations

-- 2. Check already-migrated rows
SELECT COUNT(*) AS already_migrated
FROM shared.negotiation_records
WHERE tenant_id = 'tlp'
  AND metadata->>'source' = 'mop_migration';

-- 3. Source row counts
SELECT
  (SELECT COUNT(*) FROM mop.contract_negotiations) AS negotiations,
  (SELECT COUNT(*) FROM mop.negotiation_rounds) AS rounds,
  (SELECT COUNT(*) FROM mop.negotiation_links) AS links;
```

### Upsert Pattern

```sql
-- negotiation_records
INSERT INTO shared.negotiation_records (...)
SELECT ... FROM mop.contract_negotiations
ON CONFLICT (id) DO NOTHING;

-- negotiation_rounds (must run after negotiation_records)
INSERT INTO shared.negotiation_rounds (...)
SELECT ... FROM mop.negotiation_rounds
ON CONFLICT (id) DO NOTHING;

-- negotiation_links (must run after negotiation_records)
INSERT INTO shared.negotiation_links (...)
SELECT ... FROM mop.negotiation_links
ON CONFLICT (id) DO NOTHING;
```

### Post-Migration Verification

```sql
SELECT
  'negotiation_records' AS table_name,
  (SELECT COUNT(*) FROM mop.contract_negotiations) AS source,
  (SELECT COUNT(*) FROM shared.negotiation_records WHERE tenant_id = 'tlp' AND metadata->>'source' = 'mop_migration') AS target
UNION ALL
SELECT
  'negotiation_rounds',
  (SELECT COUNT(*) FROM mop.negotiation_rounds),
  (SELECT COUNT(*) FROM shared.negotiation_rounds WHERE tenant_id = 'tlp')
UNION ALL
SELECT
  'negotiation_links',
  (SELECT COUNT(*) FROM mop.negotiation_links),
  (SELECT COUNT(*) FROM shared.negotiation_links WHERE tenant_id = 'tlp');
```

### Rollback

```sql
DELETE FROM shared.negotiation_links WHERE tenant_id = 'tlp';
DELETE FROM shared.negotiation_rounds WHERE tenant_id = 'tlp';
DELETE FROM shared.negotiation_records WHERE tenant_id = 'tlp' AND metadata->>'source' = 'mop_migration';
```

Order matters: links and rounds first (FK CASCADE from negotiation_records would handle it, but explicit is clearer).

## 12. Proposed --dry-run Output

```json
{
  "migration": "@milo/contract-negotiation",
  "version": "1.0.0",
  "dependency_check": {
    "analysis_records_present": true,
    "analysis_ids_resolved": 33,
    "analysis_ids_missing": 0
  },
  "source": {
    "project": "wjxtfjaixkoifdqtfmqd",
    "tables": {
      "contract_negotiations": { "rows": "N" },
      "negotiation_rounds": { "rows": "N" },
      "negotiation_links": { "rows": "N" }
    }
  },
  "target": {
    "project": "tappyckcteqgryjniwjg",
    "tables": {
      "negotiation_records": { "already_migrated": 0 },
      "negotiation_rounds": { "already_migrated": 0 },
      "negotiation_links": { "already_migrated": 0 }
    },
    "tenant_id": "tlp"
  },
  "plan": {
    "negotiation_records": {
      "rows_to_migrate": "N",
      "transformations": [
        { "column": "tenant_id = 'tlp'", "affected": "all" },
        { "column": "created_by 'mop' → 'system'", "affected": "rows with created_by='mop'" },
        { "column": "cancelled_at (derived from updated_at)", "affected": "rows with status='cancelled'" },
        { "column": "metadata (constructed)", "affected": "all" }
      ],
      "status_distribution": {
        "broker_review": "?",
        "counterparty_review": "?",
        "finalized": "?",
        "cancelled": "?"
      }
    },
    "negotiation_rounds": {
      "rows_to_migrate": "N",
      "transformations": [
        { "column": "tenant_id = 'tlp'", "affected": "all" }
      ]
    },
    "negotiation_links": {
      "rows_to_migrate": "N",
      "transformations": [
        { "column": "tenant_id = 'tlp'", "affected": "all" }
      ],
      "active_links": "? (counterparty_review negotiations)"
    }
  },
  "warnings": [],
  "blockers": []
}
```

## 13. Migration Execution Order

```
Pre-requisite: @milo/contract-analysis migration complete (analysis_records populated)

1. Run pre-flight checks (FK resolution, already-migrated count)
2. Run anomaly detection queries (state integrity)
3. --dry-run: produce plan JSON
4. Migrate negotiation_records (all statuses)
5. Migrate negotiation_rounds (depends on negotiation_records FK)
6. Migrate negotiation_links (depends on negotiation_records FK)
7. Post-migration verification (row counts, FK integrity)
8. Decision: redirect old MOP /review/[token] URLs or accept breakage
```

## 14. Full Migration Dependency Graph

```
@milo/contract-analysis migration
  │
  └──> analysis_records populated (33 rows, UUIDs preserved)
         │
         ├──> @milo/contract-signing migration (signing_documents.analysis_id?)
         │      └── No — signing_documents has no analysis_id FK
         │
         └──> @milo/contract-negotiation migration (THIS)
                └── negotiation_records.analysis_id REFERENCES analysis_records(id)

@milo/contract-signing migration
  │
  └──> signing_documents populated (109 rows)
         │
         └──> Short code namespace: signing resolver queries both
              signing_documents AND negotiation_links
              └── Negotiation migration should run AFTER signing
                  (so resolver can see both tables)
```

**Final ordering:**
1. @milo/contract-analysis migration
2. @milo/contract-signing migration
3. @milo/contract-negotiation migration (this one last)

## 15. Open Questions for Mark

1. **Row counts?** The freeze manifest doesn't list negotiation tables. Before migration, need: `SELECT COUNT(*) FROM contract_negotiations`, `...negotiation_rounds`, `...negotiation_links`. Can verify from the full-backup.sql dump (grep for COPY statements).

2. **created_by mapping:** Should `created_by = 'mop'` be preserved as-is (for provenance), or mapped to `'system'` (per engine default)? I've proposed mapping + preserving original in `metadata.original_created_by`, but Mark may prefer keeping `'mop'` as historical data.

3. **Active counterparty_review negotiations:** How many exist? If any counterparties are actively in review (sent a link, awaiting response), need a decision on redirect vs re-send vs accept breakage. A single `SELECT COUNT(*) FROM contract_negotiations WHERE status = 'counterparty_review'` answers this.

4. **Webhook URL disposition:** Active negotiations' `webhook_url` may point at PPCRM endpoints that no longer accept connections. Should these be nulled out, updated to a new consumer, or left as-is (engine retries will fail gracefully)?

5. **rider_templates future:** Does TLP need these templates post-migration? If yes, where should they live? Options: vertical's own Supabase table, a JSON config file, or embedded in the vertical's `vertical.config.ts`.

6. **Short code namespace collision:** Both signing_documents and negotiation_links use `LEFT(id::text, 8)` for short_code backfill. Since IDs are UUIDs, first-8-chars are unique per table but could theoretically collide across tables. The engine's resolver queries both tables — should we check for cross-table collisions before migration?

7. **ON DELETE CASCADE → what happens?** MOP's FK is `REFERENCES contract_analyses(id) ON DELETE CASCADE`. Engine's FK is `REFERENCES analysis_records(id)` (no CASCADE specified in SCHEMA.md). If an analysis_record is deleted, should negotiations cascade-delete or orphan? This is a design decision for Coder-1 — flagging here for awareness.
