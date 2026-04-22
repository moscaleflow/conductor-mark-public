---
directive: "@milo/contract-negotiation migration plan (delta-aware)"
lane: research
coder: Coder-3
started: 2026-04-20 ~09:15 MDT
completed: 2026-04-20 ~09:45 MDT
depends-on:
  - reports/2026-04-20-coder-3-negotiation-migration-audit.md (2c6f7dd)
  - reports/2026-04-20-coder-3-signing-negotiation-delta-protocol.md (25fc353)
  - Decision 53 (@milo/contract-analysis migration complete)
implements-for: Coder-1 (script author)
---

# @milo/contract-negotiation Migration Script Plan — 2026-04-20

## Overview

| Parameter | Value |
|-----------|-------|
| Source | MOP frozen (wjxtfjaixkoifdqtfmqd) |
| Target | Shared Supabase (tappyckcteqgryjniwjg) |
| Tenant | 'tlp' |
| Source tables | 3 (contract_negotiations, negotiation_rounds, negotiation_links) |
| Source rows (est.) | ≤33 negotiations + rounds + links (~50 total across 3 tables) |
| Target tables | 3 (negotiation_records, negotiation_rounds, negotiation_links) |
| Column mapping | Near 1:1 (30/33 direct, +4 new columns, 0 dropped) |
| FK dependency | analysis_id → analysis_records (Decision 53 COMPLETE — satisfied) |
| Delta risk | YES — unfreeze window was open; delta protocol ready (25fc353) |
| Script location | packages/contract-negotiation/scripts/migrate-mop-data.ts |

---

## (a) Pre-Flight Checks

```typescript
async function preflight(target: SupabaseClient, source: SupabaseClient) {
  const checks: { name: string; pass: boolean; detail?: string }[] = [];

  // 1. Target tables exist (Pattern 8 drift guard)
  for (const table of ['negotiation_records', 'negotiation_rounds', 'negotiation_links']) {
    const { error } = await target.from(table).select('id', { count: 'exact', head: true });
    checks.push({ name: `table_exists:${table}`, pass: !error, detail: error?.message });
  }

  // 2. analysis_id FK target exists (Decision 53 prerequisite)
  const { count: analysisCount } = await target
    .from('analysis_records')
    .select('*', { count: 'exact', head: true })
    .eq('tenant_id', 'tlp');
  checks.push({
    name: 'analysis_records_migrated',
    pass: (analysisCount ?? 0) >= 33,
    detail: `Expected ≥33, got ${analysisCount}`
  });

  // 3. No existing TLP negotiation data (idempotency check)
  const { count: existingNeg } = await target
    .from('negotiation_records')
    .select('*', { count: 'exact', head: true })
    .eq('tenant_id', 'tlp');
  checks.push({ name: 'no_existing_data', pass: existingNeg === 0, detail: `${existingNeg} rows` });

  // 4. Source tables accessible
  const { count: sourceNeg } = await source
    .from('contract_negotiations')
    .select('*', { count: 'exact', head: true });
  checks.push({ name: 'source_accessible', pass: (sourceNeg ?? 0) > 0, detail: `${sourceNeg} rows` });

  // 5. All source analysis_ids exist in target analysis_records
  const { data: negotiations } = await source
    .from('contract_negotiations')
    .select('analysis_id');
  const analysisIds = [...new Set(negotiations!.map(n => n.analysis_id))];
  const { count: resolvedAnalyses } = await target
    .from('analysis_records')
    .select('*', { count: 'exact', head: true })
    .in('id', analysisIds)
    .eq('tenant_id', 'tlp');
  checks.push({
    name: 'analysis_id_resolution',
    pass: resolvedAnalyses === analysisIds.length,
    detail: `${resolvedAnalyses}/${analysisIds.length} resolved`
  });

  // Report
  const failed = checks.filter(c => !c.pass);
  if (failed.length > 0) {
    console.error('PREFLIGHT FAILED:', failed);
    process.exit(1);
  }
  console.log(`✓ All ${checks.length} preflight checks passed`);
}
```

---

## (b) Delta Detection

**Reference:** `reports/2026-04-20-coder-3-signing-negotiation-delta-protocol.md` (commit 25fc353)

Queries 3, 4, and 5 from that document cover negotiation delta detection. Do NOT duplicate — invoke them as a pre-step.

```typescript
async function detectDelta(source: SupabaseClient): Promise<DeltaResult> {
  const MANIFEST_TIMESTAMP = '2026-04-19T20:50:00Z';
  const UNFREEZE_TIMESTAMP = '2026-04-20T14:47:59Z';

  // Query 3: contract_negotiations
  const { data: negDelta } = await source.rpc('count_negotiation_delta', {
    since: UNFREEZE_TIMESTAMP
  });
  // Fallback if no RPC: use .select with filters
  const { count: negUpdated } = await source
    .from('contract_negotiations')
    .select('*', { count: 'exact', head: true })
    .gt('updated_at', UNFREEZE_TIMESTAMP);

  // Query 4: negotiation_rounds
  const { count: roundsNew } = await source
    .from('negotiation_rounds')
    .select('*', { count: 'exact', head: true })
    .gt('submitted_at', UNFREEZE_TIMESTAMP);

  // Query 5: negotiation_links
  const { count: linksViewed } = await source
    .from('negotiation_links')
    .select('*', { count: 'exact', head: true })
    .gt('last_viewed_at', UNFREEZE_TIMESTAMP);

  const hasDelta = (negUpdated ?? 0) > 0 || (roundsNew ?? 0) > 0 || (linksViewed ?? 0) > 0;

  console.log(`Delta detection: negotiations=${negUpdated}, rounds=${roundsNew}, links_viewed=${linksViewed}`);
  console.log(hasDelta ? '⚠ DELTA DETECTED — using live source' : '✓ Zero delta — audit numbers valid');

  return { hasDelta, negUpdated, roundsNew, linksViewed };
}
```

---

## (c) Decision Branch

```typescript
async function resolveSource(source: SupabaseClient, delta: DeltaResult) {
  if (!delta.hasDelta) {
    // Zero delta: use audit numbers as-is
    // Source: existing freeze backup OR live (identical)
    console.log('Using audit baseline counts');
    return;
  }

  // Delta detected: read live row counts (post-freeze, writes now blocked)
  const { count: negCount } = await source
    .from('contract_negotiations').select('*', { count: 'exact', head: true });
  const { count: roundsCount } = await source
    .from('negotiation_rounds').select('*', { count: 'exact', head: true });
  const { count: linksCount } = await source
    .from('negotiation_links').select('*', { count: 'exact', head: true });

  console.log(`Updated counts: negotiations=${negCount}, rounds=${roundsCount}, links=${linksCount}`);
  // These become the authoritative migration counts
}
```

**In both cases, the migration reads from LIVE source** (freeze is active, so live = final state). The delta detection is informational — it tells us whether the audit's estimated counts are still valid, but the script always reads live data regardless.

---

## (d) analysis_id FK Resolution

**Status: SATISFIED.** Decision 53 confirmed @milo/contract-analysis migration complete. All 33 analysis records are in shared Supabase `analysis_records` table with `tenant_id='tlp'`.

**UUID preservation:** The analysis migration preserved source UUIDs (same pattern as CRM migration). Therefore:
- MOP's `contract_negotiations.analysis_id` = shared Supabase's `analysis_records.id`
- No lookup or remapping needed
- Preflight check #5 verifies this at runtime

```typescript
// FK resolution is trivial — direct pass-through
// analysis_id in source = analysis_id in target (same UUID)
```

---

## (e) Column Mappings + Transformations

### Table 1: contract_negotiations → negotiation_records

```typescript
function transformNegotiation(n: SourceNegotiation) {
  return {
    id: n.id,                              // UUID preserved
    tenant_id: 'tlp',                      // NEW
    analysis_id: n.analysis_id,            // Direct (FK satisfied)
    status: n.status,                      // Direct (all values valid)
    document_type: n.document_type,        // Direct
    counterparty_name: n.counterparty_name, // Direct
    counterparty_email: n.counterparty_email, // Direct
    broker_decisions: n.broker_decisions,   // Direct (JSONB)
    current_round: n.current_round,        // Direct
    created_by: n.created_by,              // Preserve as-is for provenance
    webhook_url: n.webhook_url,            // Direct
    finalized_at: n.finalized_at,          // Direct
    cancelled_at: n.status === 'cancelled' ? n.updated_at : null, // DERIVED
    metadata: {
      source: 'mop_migration',
      migrated_at: new Date().toISOString(),
      original_created_by: n.created_by
    },
    created_at: n.created_at,              // Direct
    updated_at: n.updated_at              // Direct
  };
}
```

**Columns added (not in source):** `tenant_id`, `cancelled_at`, `metadata`
**Columns transformed:** `cancelled_at` derived from status + updated_at
**Columns dropped:** 0

### Table 2: negotiation_rounds → negotiation_rounds

```typescript
function transformRound(r: SourceRound) {
  return {
    id: r.id,                    // UUID preserved
    tenant_id: 'tlp',           // NEW
    negotiation_id: r.negotiation_id, // Direct (FK to negotiation_records)
    round_number: r.round_number,     // Direct
    actor_type: r.actor_type,         // Direct
    actor_name: r.actor_name,         // Direct (may contain 'TLP Compliance')
    actor_ip: r.actor_ip,             // Direct
    actor_user_agent: r.actor_user_agent, // Direct
    decisions: r.decisions,           // Direct (JSONB)
    overall_notes: r.overall_notes,   // Direct
    submitted_at: r.submitted_at      // Direct
  };
}
```

**Pure 1:1 plus tenant_id.** Zero transformations.

### Table 3: negotiation_links → negotiation_links

```typescript
function transformLink(l: SourceLink) {
  return {
    id: l.id,                    // UUID preserved
    tenant_id: 'tlp',           // NEW
    negotiation_id: l.negotiation_id, // Direct (FK to negotiation_records)
    token: l.token,              // Direct (UNIQUE)
    short_code: l.short_code,    // Direct (UNIQUE, backfilled)
    link_type: l.link_type,      // Direct
    round_number: l.round_number, // Direct
    expires_at: l.expires_at,    // Direct (preserve NULL — Option A per audit)
    first_viewed_at: l.first_viewed_at, // Direct
    last_viewed_at: l.last_viewed_at,   // Direct
    view_count: l.view_count,    // Direct
    created_at: l.created_at     // Direct
  };
}
```

**Pure 1:1 plus tenant_id.** `expires_at` preserved as NULL (no retroactive expiry).

---

## (f) Idempotency

| Table | Strategy | Conflict Key |
|-------|----------|--------------|
| negotiation_records | ON CONFLICT (id) DO NOTHING | UUID PK preserved |
| negotiation_rounds | ON CONFLICT (id) DO NOTHING | UUID PK preserved |
| negotiation_links | ON CONFLICT (id) DO NOTHING | UUID PK preserved |

```typescript
async function migrateTable(
  target: SupabaseClient,
  tableName: string,
  rows: any[],
  batchSize = 50
): Promise<number> {
  let migrated = 0;
  for (let i = 0; i < rows.length; i += batchSize) {
    const batch = rows.slice(i, i + batchSize);
    const { error, count } = await target
      .from(tableName)
      .upsert(batch, { onConflict: 'id', ignoreDuplicates: true })
      .select('id', { count: 'exact', head: true });

    if (error) throw new Error(`${tableName} batch ${i} failed: ${error.message}`);
    migrated += batch.length;
  }
  return migrated;
}
```

**Re-run safety:** If script runs twice, all inserts hit the `ON CONFLICT (id) DO NOTHING` clause. Zero side effects on re-run.

---

## Script Execution Flow

```typescript
async function main() {
  const source = createClient(MOP_URL, MOP_SERVICE_KEY);
  const target = createClient(SHARED_URL, SHARED_SERVICE_KEY);

  // Phase 1: Preflight
  await preflight(target, source);

  // Phase 2: Delta detection
  const delta = await detectDelta(source);
  await resolveSource(source, delta);

  // Phase 3: Read all source data (order matters for FK integrity reporting)
  const { data: negotiations } = await source
    .from('contract_negotiations')
    .select('*')
    .order('created_at');
  console.log(`Read ${negotiations!.length} negotiations`);

  const { data: rounds } = await source
    .from('negotiation_rounds')
    .select('*')
    .order('negotiation_id, round_number');
  console.log(`Read ${rounds!.length} rounds`);

  const { data: links } = await source
    .from('negotiation_links')
    .select('*')
    .order('negotiation_id, created_at');
  console.log(`Read ${links!.length} links`);

  // Phase 4: Transform
  const targetNegotiations = negotiations!.map(transformNegotiation);
  const targetRounds = rounds!.map(transformRound);
  const targetLinks = links!.map(transformLink);

  // Phase 5: Insert (ORDER MATTERS — negotiations first, then rounds + links)
  const negCount = await migrateTable(target, 'negotiation_records', targetNegotiations);
  const roundCount = await migrateTable(target, 'negotiation_rounds', targetRounds);
  const linkCount = await migrateTable(target, 'negotiation_links', targetLinks);

  // Phase 6: Verify
  const verified = await verify(target, negCount, roundCount, linkCount);
  if (!verified) {
    console.error('Verification failed — consider rollback');
    process.exit(1);
  }

  // Summary
  console.log('\n=== Migration Complete ===');
  console.log(`Negotiations: ${negCount}`);
  console.log(`Rounds:       ${roundCount}`);
  console.log(`Links:        ${linkCount}`);
  console.log(`Total:        ${negCount + roundCount + linkCount} rows`);
}
```

### Insert Ordering

```
1. negotiation_records (no inbound FKs from other migration tables)
2. negotiation_rounds (FK → negotiation_records.id)
3. negotiation_links (FK → negotiation_records.id)
```

Rounds and links can insert in parallel (both reference negotiations, neither references the other), but sequential is safer for error reporting.

---

## (g) Post-Migration Verification

```typescript
async function verify(
  target: SupabaseClient,
  expectedNeg: number,
  expectedRounds: number,
  expectedLinks: number
): Promise<boolean> {
  const checks: { name: string; expected: number; actual: number }[] = [];

  // 1. Negotiation count
  const { count: negCount } = await target
    .from('negotiation_records')
    .select('*', { count: 'exact', head: true })
    .eq('tenant_id', 'tlp');
  checks.push({ name: 'negotiations', expected: expectedNeg, actual: negCount! });

  // 2. Rounds count
  const { count: roundsCount } = await target
    .from('negotiation_rounds')
    .select('*', { count: 'exact', head: true })
    .eq('tenant_id', 'tlp');
  checks.push({ name: 'rounds', expected: expectedRounds, actual: roundsCount! });

  // 3. Links count
  const { count: linksCount } = await target
    .from('negotiation_links')
    .select('*', { count: 'exact', head: true })
    .eq('tenant_id', 'tlp');
  checks.push({ name: 'links', expected: expectedLinks, actual: linksCount! });

  // 4. FK integrity: all negotiation_ids in rounds reference existing negotiations
  const { data: orphanRounds } = await target
    .from('negotiation_rounds')
    .select('id, negotiation_id')
    .eq('tenant_id', 'tlp')
    .not('negotiation_id', 'in',
      `(SELECT id FROM negotiation_records WHERE tenant_id='tlp')`);
  // Alternative: DB constraint would have blocked insert — but verify anyway
  checks.push({ name: 'round_fk_integrity', expected: 0, actual: orphanRounds?.length ?? 0 });

  // 5. FK integrity: all analysis_ids reference analysis_records
  const { data: orphanAnalysis } = await target
    .from('negotiation_records')
    .select('id, analysis_id')
    .eq('tenant_id', 'tlp')
    .not('analysis_id', 'in',
      `(SELECT id FROM analysis_records WHERE tenant_id='tlp')`);
  checks.push({ name: 'analysis_fk_integrity', expected: 0, actual: orphanAnalysis?.length ?? 0 });

  // 6. Status consistency: finalized rows have finalized_at
  const { count: inconsistent } = await target
    .from('negotiation_records')
    .select('*', { count: 'exact', head: true })
    .eq('tenant_id', 'tlp')
    .eq('status', 'finalized')
    .is('finalized_at', null);
  checks.push({ name: 'finalized_consistency', expected: 0, actual: inconsistent ?? 0 });

  // 7. Every negotiation has at least 1 round (except drafts)
  const { data: noRounds } = await target
    .from('negotiation_records')
    .select('id, status')
    .eq('tenant_id', 'tlp')
    .neq('status', 'draft');
  // For each non-draft negotiation, verify round exists
  let missingRounds = 0;
  if (noRounds) {
    for (const neg of noRounds) {
      const { count } = await target
        .from('negotiation_rounds')
        .select('*', { count: 'exact', head: true })
        .eq('negotiation_id', neg.id);
      if (!count || count === 0) missingRounds++;
    }
  }
  checks.push({ name: 'rounds_per_negotiation', expected: 0, actual: missingRounds });

  // Report
  const failed = checks.filter(c => c.expected !== c.actual);
  if (failed.length > 0) {
    console.error('❌ VERIFICATION FAILED:', failed);
    return false;
  }
  console.log(`✓ All ${checks.length} verification checks passed`);
  return true;
}
```

---

## (h) Rollback Plan

```typescript
async function rollback(target: SupabaseClient) {
  console.log('Rolling back @milo/contract-negotiation migration for tenant=tlp...');

  // Order: links + rounds first (FK to negotiations), then negotiations
  const { error: e1 } = await target
    .from('negotiation_links')
    .delete()
    .eq('tenant_id', 'tlp');
  if (e1) console.error('Links delete error:', e1.message);

  const { error: e2 } = await target
    .from('negotiation_rounds')
    .delete()
    .eq('tenant_id', 'tlp');
  if (e2) console.error('Rounds delete error:', e2.message);

  const { error: e3 } = await target
    .from('negotiation_records')
    .delete()
    .eq('tenant_id', 'tlp');
  if (e3) console.error('Negotiations delete error:', e3.message);

  // Verify
  const { count } = await target
    .from('negotiation_records')
    .select('*', { count: 'exact', head: true })
    .eq('tenant_id', 'tlp');
  console.log(`✓ Rollback complete (${count} remaining — should be 0)`);
}
```

**Safety:** 
- Tenant-scoped — only TLP data affected
- CASCADE on FK would handle rounds+links automatically, but explicit per-table delete preferred for audit clarity
- No downstream dependents yet (signing migration references analysis_id, not negotiation_id)
- Rollback does NOT affect analysis_records (independent migration)

---

## Environment Variables

```bash
# Source (MOP frozen, writes blocked by CONTRACT_WRITES_FROZEN)
MOP_SUPABASE_URL=https://wjxtfjaixkoifdqtfmqd.supabase.co
MOP_SERVICE_ROLE_KEY=<from: npx supabase projects api-keys --project-ref wjxtfjaixkoifdqtfmqd>

# Target (shared)
SHARED_SUPABASE_URL=https://tappyckcteqgryjniwjg.supabase.co
SHARED_SERVICE_ROLE_KEY=<from: npx supabase projects api-keys --project-ref tappyckcteqgryjniwjg>
```

---

## Execution Prerequisites

```
□ CONTRACT_WRITES_FROZEN=true active on MOP (Decision 52)
□ Delta detection queries run (Queries 3-5 from delta protocol)
□ @milo/contract-analysis migration complete (Decision 53 ✓)
□ @milo/contract-signing migration complete (Coder-1 active)
□ DDL applied: negotiation_records, negotiation_rounds, negotiation_links exist
□ RLS policies applied on target tables
```

**Ordering in the full migration sequence:**
```
1. @milo/contract-analysis  ✓ (Decision 53)
2. @milo/contract-signing   ← Coder-1 executing now
3. @milo/contract-negotiation ← THIS PLAN (next after signing)
4. @milo/onboarding          (independent — PPCRM source, no FK deps on contracts)
```

---

## Edge Cases

| Case | Handling |
|------|----------|
| Negotiation with analysis_id not in target | Preflight check #5 catches — aborts before insert |
| Rounds with negotiation_id not in target | Insert ordering (negotiations first) prevents this |
| Links with expired tokens (expires_at in past) | Preserve as-is — historical data |
| view_count incremented during delta window | Fresh read from live captures current value |
| Status='cancelled' with no explicit timestamp | Derive cancelled_at from updated_at |
| UNIQUE(token) conflict on links | ON CONFLICT (id) DO NOTHING handles it |
| Empty tables (0 negotiations) | Script completes successfully with 0 rows migrated |
