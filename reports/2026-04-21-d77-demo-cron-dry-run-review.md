# D77 Report: Demo Tenant Cron Dry-Run Review

**Agent:** Coder-2 | **Date:** 2026-04-21 | **Status:** Bug found

---

## Cron Configuration

| Property | Value |
|----------|-------|
| **File** | `src/app/api/cron/demo-cleanup/route.ts` |
| **Schedule** | `0 * * * *` (hourly, top of hour) |
| **vercel.json entry** | Line 44: `{"path":"/api/cron/demo-cleanup","schedule":"0 * * * *"}` |
| **Auth** | `CRON_SECRET` bearer token (Vercel-injected) |
| **Master switch** | `DEMO_CLEANUP_ENABLED` — must be `"true"` to run (currently: `true`) |
| **Dry-run flag** | `DEMO_CLEANUP_DRY_RUN` — dry-run unless set to `"false"` (currently: `"true"`) |

---

## 24-Hour Dry-Run Summary

### Prior to manual test (all hourly runs since deploy)

Every hourly invocation returned:
```json
{"status":"ok","message":"No expired demo tenants","dryRun":true}
```
**Reason:** Neither `demo-ppc` nor `milo-internal` tenant rows existed in the `tenants` table. The cron query (`is_demo = true AND demo_expires_at < NOW()`) returned zero rows every time.

### Manual test run (2026-04-21 ~21:06 UTC)

Inserted `demo-ppc` with `demo_expires_at` = 2h ago, then invoked cron:

```json
{
  "status": "ok",
  "dryRun": true,
  "tenantsProcessed": 1,
  "results": [{
    "tenantId": "demo-ppc",
    "dryRun": true,
    "tables": {
      "onboarding_steps": 0,
      "onboarding_runs": 0,
      "onboarding_flows": 0,
      "negotiation_links": 0,
      "negotiation_rounds": 0,
      "negotiation_records": 0,
      "signing_library": 0,
      "signing_documents": 0,
      "analysis_jobs": 0,
      "analysis_records": 1,
      "blacklist_entries": 0,
      "crm_activities": 0,
      "crm_contacts": 0,
      "crm_leads": 0,
      "crm_counterparties": 0,
      "feedback_signals": 0
    },
    "storageFiles": 0,
    "tenantDeleted": false
  }]
}
```

**Findings:**
- 1 `analysis_records` row found (from earlier hero test: `milo-sample-ppc-publisher-contract.pdf`, id `dc175207`)
- All 16 tenant-scoped tables scanned correctly
- Storage bucket check returned 0 files
- `tenantDeleted: false` (dry-run — no actual deletion)

### Non-demo tenant safety check (CRITICAL — PASS)

| Tenant | `is_demo` | `demo_expires_at` | Safe? |
|--------|-----------|-------------------|-------|
| `tlp` | `false` | `null` | YES — excluded by `WHERE is_demo = true` |
| `mysecretary` | `false` | `null` | YES — excluded by `WHERE is_demo = true` |
| `milo-internal` | (not yet created) | — | YES — D46 creates with `isDemo: false` |

**Zero non-demo tenants flagged. The cron will never target `tlp`, `mysecretary`, or `milo-internal`.**

Additional safety: the tenant DELETE at line 99-101 has a double guard: `.eq('id', tenantId).eq('is_demo', true)`. Even if the query somehow returned a non-demo tenant ID, the delete would be a no-op.

---

## Bug: `ensureTenant` insert missing NOT NULL columns

**Severity:** High — blocks hero flow for new visitors when `demo-ppc` doesn't exist

**Root cause:** `src/lib/demo/ensure-tenant.ts` line 23-29 inserts with only `id`, `display_name`, `is_demo`, `demo_expires_at`, `active`. The `tenants` table requires NOT NULL values for `domain`, `from_address`, `reply_to`, `physical_address`.

**Evidence:**
```
code: "23502"
message: "null value in column \"domain\" of relation \"tenants\" violates not-null constraint"
```

**Impact:** Every first-time hero analysis call hits `ensureDemoPpcTenant()` → constraint violation → 500 error. Subsequent calls also fail because the tenant never gets created.

**Fix required in `ensure-tenant.ts`:** Add placeholder values to the insert:

```typescript
// Before (broken):
await supabase.from('tenants').insert({
  id: tenantId,
  display_name: opts.displayName,
  is_demo: opts.isDemo,
  demo_expires_at: expiresAt,
  active: true,
})

// After (fixed):
await supabase.from('tenants').insert({
  id: tenantId,
  display_name: opts.displayName,
  domain: opts.domain ?? 'demo.justmilo.app',
  from_address: opts.fromAddress ?? 'Milo <milo@justmilo.app>',
  reply_to: opts.replyTo ?? 'milo@justmilo.app',
  physical_address: opts.physicalAddress ?? 'N/A',
  is_demo: opts.isDemo,
  demo_expires_at: expiresAt,
  active: true,
})
```

---

## Proposed Flag Flip Diff (DO NOT APPLY — requires Mark approval)

**File:** Vercel Environment Variables (not a code change)

```
DEMO_CLEANUP_DRY_RUN: "true" → "false"
```

**Prerequisite:** Fix the `ensure-tenant.ts` bug first. Without it, `demo-ppc` can never be created via the hero flow, so the cron has nothing to clean up.

**Effect when flipped:**
- Expired demo tenants will have all rows deleted from 16 tables
- Storage files in `contract-originals/{tenant_id}/` will be removed  
- The tenant row itself will be deleted (`WHERE id = ? AND is_demo = true`)
- Next hero visitor triggers `ensureDemoPpcTenant` → fresh tenant created

---

## Cleanup

- Test `demo-ppc` tenant row: **deleted** (reverted to pre-test state)
- `.env.production.local`: **deleted** (contained CRON_SECRET)
- The 1 `analysis_records` row (`dc175207`) remains — it's orphaned (no parent tenant) but harmless

---

## Recommendations

1. **Fix `ensure-tenant.ts` NOT NULL bug** — blocking issue for hero flow
2. **After fix ships + verify hero works end-to-end** — then flip `DEMO_CLEANUP_DRY_RUN` to `false`
3. Consider whether `analysis_records` orphaned by tenant deletion should be left (audit trail) or swept (the current cron deletes records before the tenant, so no orphans in normal flow)
