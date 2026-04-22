---
directive: "Pre-migration delta detection + fresh backup protocol"
lane: research
coder: Coder-3
started: 2026-04-20 ~07:30 MDT
completed: 2026-04-20 ~07:45 MDT
execution: PENDING — fire after CONTRACT_WRITES_FROZEN=true confirmed
---

# Signing + Negotiation Delta Detection Protocol — 2026-04-20

## 1. Timeline Reference

| Event | Timestamp (UTC) | Significance |
|-------|----------------|--------------|
| MOP freeze manifest taken | 2026-04-19 ~20:50 | Baseline row counts |
| MOP global unfreeze (Decision 45) | 2026-04-20 14:47:59 | Delta window OPENS |
| CONTRACT_WRITES_FROZEN activated | **PENDING** | Delta window CLOSES |

**Delta window:** Any writes between 2026-04-20 14:47:59 UTC and freeze activation.

## 2. Baseline Row Counts (Freeze Manifest 2026-04-19)

| Table | Baseline Rows | Source |
|-------|--------------|--------|
| signing_documents | 109 | mop-freeze-2026-04-19.manifest |
| contract_analyses | 33 | mop-freeze-2026-04-19.manifest |
| contract_negotiations | ~15-20 (est) | negotiation audit (2c6f7dd) |
| negotiation_rounds | ~20-30 (est) | negotiation audit (2c6f7dd) |
| negotiation_links | ~15-20 (est) | negotiation audit (2c6f7dd) |

**Note:** Negotiation table counts were not individually listed in the freeze manifest (low-row tables omitted). The "~50 across 3 tables" estimate comes from the negotiation migration audit.

## 3. Delta Detection Queries

### Prerequisites

```bash
# Get service_role key for MOP project
npx supabase projects api-keys --project-ref wjxtfjaixkoifdqtfmqd
```

All queries use the service_role key via Supabase client or `psql`. All are SELECT-only (read-only, no PII exposure — aggregates and UUIDs only).

---

### Query 1: signing_documents Row Count + Delta

```sql
-- Run AFTER CONTRACT_WRITES_FROZEN=true is confirmed active
-- Expected: 109 (no new rows possible — PPCRM frozen)

SELECT
  COUNT(*) AS total_rows,
  COUNT(*) FILTER (WHERE created_at > '2026-04-19T20:50:00Z') AS rows_created_since_manifest,
  COUNT(*) FILTER (WHERE updated_at > '2026-04-20T14:47:59Z') AS rows_updated_during_window,
  COUNT(*) FILTER (WHERE signed_at > '2026-04-20T14:47:59Z') AS signed_during_window,
  COUNT(*) FILTER (WHERE viewed_at > '2026-04-20T14:47:59Z') AS viewed_during_window,
  COUNT(*) FILTER (WHERE status = 'declined' AND updated_at > '2026-04-20T14:47:59Z') AS declined_during_window,
  MAX(updated_at) AS latest_update
FROM signing_documents;
```

**Decision tree:**
- If `rows_created_since_manifest = 0` AND `rows_updated_during_window = 0` → **ZERO DELTA**, proceed with 109-row backup
- If `rows_created_since_manifest = 0` AND `rows_updated_during_window > 0` → **STATUS DELTA**, need fresh backup
- If `rows_created_since_manifest > 0` → **UNEXPECTED** (should be impossible with PPCRM frozen — investigate)

### Query 2: signing_documents Delta Row UUIDs

```sql
-- Only run if Query 1 shows delta > 0
-- Returns UUIDs of affected rows (no PII)

SELECT id, status, updated_at,
  CASE WHEN signed_at > '2026-04-20T14:47:59Z' THEN 'signed_in_window' 
       WHEN viewed_at > '2026-04-20T14:47:59Z' THEN 'viewed_in_window'
       ELSE 'other_update' END AS delta_type
FROM signing_documents
WHERE updated_at > '2026-04-20T14:47:59Z'
ORDER BY updated_at DESC;
```

### Query 3: contract_negotiations Delta

```sql
SELECT
  COUNT(*) AS total_rows,
  COUNT(*) FILTER (WHERE created_at > '2026-04-19T20:50:00Z') AS created_since_manifest,
  COUNT(*) FILTER (WHERE updated_at > '2026-04-20T14:47:59Z') AS updated_during_window,
  COUNT(*) FILTER (WHERE status = 'finalized' AND finalized_at > '2026-04-20T14:47:59Z') AS finalized_during_window,
  COUNT(*) FILTER (WHERE status = 'cancelled' AND updated_at > '2026-04-20T14:47:59Z') AS cancelled_during_window,
  MAX(updated_at) AS latest_update
FROM contract_negotiations;
```

### Query 4: negotiation_rounds Delta

```sql
SELECT
  COUNT(*) AS total_rows,
  COUNT(*) FILTER (WHERE submitted_at > '2026-04-19T20:50:00Z') AS rounds_since_manifest,
  COUNT(*) FILTER (WHERE submitted_at > '2026-04-20T14:47:59Z') AS rounds_during_window,
  MAX(submitted_at) AS latest_round
FROM negotiation_rounds;
```

### Query 5: negotiation_links Delta

```sql
SELECT
  COUNT(*) AS total_rows,
  COUNT(*) FILTER (WHERE created_at > '2026-04-19T20:50:00Z') AS links_created_since_manifest,
  COUNT(*) FILTER (WHERE last_viewed_at > '2026-04-20T14:47:59Z') AS links_viewed_during_window,
  COUNT(*) FILTER (WHERE first_viewed_at > '2026-04-20T14:47:59Z') AS links_first_viewed_during_window,
  MAX(last_viewed_at) AS latest_view
FROM negotiation_links;
```

### Query 6: contract_analyses Delta (bonus — verify analysis migration baseline)

```sql
SELECT
  COUNT(*) AS total_rows,
  COUNT(*) FILTER (WHERE created_at > '2026-04-19T20:50:00Z') AS created_since_manifest,
  COUNT(*) FILTER (WHERE updated_at > '2026-04-20T14:47:59Z') AS updated_during_window,
  MAX(updated_at) AS latest_update
FROM contract_analyses;

-- Expected: total=33, created_since=0, updated_during=0
```

## 4. Expected vs Actual Decision Matrix

| Query Result | Meaning | Action |
|-------------|---------|--------|
| All zeros | No delta | Proceed with existing backup (109/33/~50 rows) |
| signing viewed > 0 only | View tracking fired | Fresh backup needed (viewed_at + audit_trail changed) |
| signing signed > 0 | Document was signed | Fresh backup REQUIRED (signature_data, status, etc.) |
| negotiation rounds > 0 | Counterparty submitted | Fresh backup needed for all 3 negotiation tables |
| negotiation links viewed > 0 | Review page accessed | Tolerable — view_count update is low-risk, but fresh backup preferred |
| analyses updated > 0 | Analysis modified | Fresh backup needed for contract_analyses |

## 5. Fresh Backup Procedure (If Delta Detected)

### When to use: Any delta query returns > 0

```bash
# 1. Confirm freeze is active (should get 503)
curl -s -o /dev/null -w "%{http_code}" -X POST \
  https://tlpmop.netlify.app/api/bot/contracts/analyze \
  -H "x-api-key: test"
# Expected: 503

# 2. Take fresh backup of affected tables only
# Using Supabase CLI with service_role key

# signing_documents (full table, ~109 rows)
npx supabase db dump --project-ref wjxtfjaixkoifdqtfmqd \
  --data-only --table signing_documents \
  > ~/mark\ conduct/conductor-mark/backups/signing-documents-post-freeze.sql

# contract_negotiations + negotiation_rounds + negotiation_links
npx supabase db dump --project-ref wjxtfjaixkoifdqtfmqd \
  --data-only --table contract_negotiations \
  --table negotiation_rounds \
  --table negotiation_links \
  > ~/mark\ conduct/conductor-mark/backups/negotiation-tables-post-freeze.sql

# contract_analyses (if delta detected)
npx supabase db dump --project-ref wjxtfjaixkoifdqtfmqd \
  --data-only --table contract_analyses \
  > ~/mark\ conduct/conductor-mark/backups/contract-analyses-post-freeze.sql
```

### Alternative: REST API row export (if pg_dump unavailable)

```bash
# Using supabase-js or curl with service_role key
# signing_documents — all rows, ordered by id
curl "https://wjxtfjaixkoifdqtfmqd.supabase.co/rest/v1/signing_documents?order=id" \
  -H "apikey: ${SUPABASE_ANON_KEY}" \
  -H "Authorization: Bearer ${SUPABASE_SERVICE_ROLE_KEY}" \
  -H "Accept: application/json" \
  > signing-documents-fresh.json
```

### Backup Verification

```sql
-- After fresh backup, verify row count matches live
-- (Run query and compare with backup file line count)
SELECT COUNT(*) FROM signing_documents;
-- Must match the number of INSERT statements in fresh backup
```

## 6. Updated Migration Row Counts (If Delta)

### Zero Delta Path (expected)

| Table | Rows | Source |
|-------|------|--------|
| signing_documents | 109 | Original freeze backup |
| contract_analyses | 33 | Original freeze backup |
| contract_negotiations | N | Original freeze backup |
| negotiation_rounds | N | Original freeze backup |
| negotiation_links | N | Original freeze backup |

**Use:** Existing `full-backup.sql` from 2026-04-19

### Non-Zero Delta Path

| Table | Rows | Source |
|-------|------|--------|
| signing_documents | 109 (same count, updated state) | **Fresh post-freeze backup** |
| contract_analyses | 33 (verify) | Fresh or original |
| contract_negotiations | N (verify) | Fresh if rounds/links changed |
| negotiation_rounds | N + delta | **Fresh post-freeze backup** |
| negotiation_links | N (same count, view_count updated) | **Fresh post-freeze backup** |

**Use:** Fresh per-table dumps from Step 5

### Migration SQL Impact

**NONE.** The column mapping (from audits a79fcf0 and 2c6f7dd) is identical regardless of row state. The INSERT...SELECT transform handles any valid row. The only difference is WHICH backup file is the source.

## 7. Execution Checklist

```
□ 1. Coder-1 reports: CONTRACT_WRITES_FROZEN=true active
□ 2. Verify freeze: curl POST to signing endpoint → 503
□ 3. Run Query 1 (signing_documents delta)
□ 4. Run Query 3 (contract_negotiations delta)
□ 5. Run Query 4 (negotiation_rounds delta)
□ 6. Run Query 5 (negotiation_links delta)
□ 7. Run Query 6 (contract_analyses delta)
□ 8. Decision: all zeros → proceed | any > 0 → fresh backup
□ 9. If fresh backup: run backup procedure (Step 5)
□ 10. If fresh backup: verify row counts match
□ 11. Report delta findings to Mark + Coder-1
□ 12. Clear Coder-1 to proceed with migration execution
```

## 8. Rollback (If Queries Reveal Problems)

If delta detection reveals unexpected state (e.g., rows created that shouldn't exist, or data corruption):

1. **DO NOT proceed with migration**
2. Report anomaly to Mark
3. Keep freeze active (prevents further changes)
4. Investigate: who/what wrote the unexpected data?
5. Decide: include in migration, exclude, or fix first?

## 9. Post-Migration Verification

After migration completes (all tables migrated to shared Supabase):

```sql
-- On shared Supabase (tappyckcteqgryjniwjg):
SELECT COUNT(*) FROM signing_documents WHERE tenant_id = 'tlp';
-- Must match source count exactly

SELECT COUNT(*) FROM contract_negotiations WHERE tenant_id = 'tlp';
SELECT COUNT(*) FROM negotiation_rounds WHERE tenant_id = 'tlp';
SELECT COUNT(*) FROM negotiation_links WHERE tenant_id = 'tlp';
SELECT COUNT(*) FROM analysis_records WHERE tenant_id = 'tlp';
-- All must match source counts
```

---

## Summary

| State | Action |
|-------|--------|
| Queries prepared | YES — 6 queries ready to fire |
| Execution blocked on | CONTRACT_WRITES_FROZEN=true activation (Coder-1) |
| If zero delta | Proceed immediately with existing backup |
| If non-zero delta | Take fresh per-table backups, then proceed |
| Migration SQL changes needed | NONE regardless of delta |
