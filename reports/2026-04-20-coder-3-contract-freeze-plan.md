---
directive: "Scope selective contract-writes freeze on MOP"
lane: research
coder: Coder-3
started: 2026-04-20 ~05:15 MDT
completed: 2026-04-20 ~05:45 MDT
---

# MOP Selective Contract-Writes Freeze Plan — 2026-04-20

## 1. Executive Summary

**26 write-path routes identified** across contract analysis, signing, negotiation, and publisher/buyer domains. All are currently live (global unfreeze per Decision 45).

**Recommended mechanism: Option C — middleware pattern matching.** Extend existing `middleware.ts` with a second env var `CONTRACT_WRITES_FROZEN=true` that 503s only contract/signing/negotiation paths. This reuses the proven freeze pattern, requires ~15 lines of code change, and is deploy-toggled via Netlify env vars.

**Financial cron re-enable:** Safe to uncomment `sync-late-conversions` and `sync-unconversions` in `netlify.toml`. Both only UPDATE `call_logs` (not a migration target). Zero dependency conflicts.

## 2. Write-Path Inventory: Contract Analysis

**Tables affected:** `contract_analyses`, `contract_analysis_jobs`

| # | Route | Method | Table(s) Written | Operation | Auth | Freq |
|---|-------|--------|------------------|-----------|------|------|
| 1 | `/api/contract/analyze-async` | POST | contract_analysis_jobs | INSERT | Public (anon) | On-demand |
| 2 | `/api/contract/analyze-process` | POST | contract_analysis_jobs, contract_analyses | UPDATE, INSERT | Public (anon) | Background |
| 3 | `/api/contract/analyze-status/[jobId]` | GET | contract_analysis_jobs, contract_analyses | UPDATE, INSERT | Public (anon) | Polling |
| 4 | `/api/bot/contracts/analyze` | POST | contract_analyses | INSERT+UPDATE | x-api-key | On-demand |
| 5 | `/api/bot/contracts/compare` | POST | contract_analyses | INSERT+UPDATE | x-api-key | On-demand |

**Note:** Route #3 is a GET with write side-effect (saves analysis result on first status poll after completion). The freeze must block this GET or handle it specially.

**Pattern for middleware match:**
```
/api/contract/analyze-async
/api/contract/analyze-process
/api/contract/analyze-status/*
/api/bot/contracts/analyze
/api/bot/contracts/compare
```

## 3. Write-Path Inventory: Signing

**Tables affected:** `signing_documents`, `publishers` (backfill), `buyers` (backfill)

| # | Route | Method | Table(s) Written | Operation | Auth | Freq |
|---|-------|--------|------------------|-----------|------|------|
| 6 | `/api/sign/[token]/submit` | POST | signing_documents, publishers, buyers | UPDATE | Public (token) | ~1-2/week |
| 7 | `/api/sign/[token]/decline` | POST | signing_documents | UPDATE | Public (token) | Rare |
| 8 | `/api/sign/[token]/capture-view` | POST | signing_documents | UPDATE | Public (token) | Per-view |
| 9 | `/api/bot/documents/generate` | POST | signing_documents | INSERT | x-api-key | On-demand |
| 10 | `/api/bot/documents/[id]/void` | POST | signing_documents | UPDATE | x-api-key | Rare |
| 11 | `/api/bot/webhooks/refire` | POST | signing_documents | UPDATE (audit_trail) | x-api-key | Rare |

**Critical note on #8 (capture-view):** This updates `viewed_at` and `status → 'viewed'` on first page load. If a signing token is active when freeze deploys, the signer can't even VIEW the document without hitting 503. See §8 for mitigation.

**Pattern for middleware match:**
```
/api/sign/*/submit
/api/sign/*/decline
/api/sign/*/capture-view
/api/bot/documents/generate
/api/bot/documents/*/void
/api/bot/webhooks/refire
```

## 4. Write-Path Inventory: Negotiation

**Tables affected:** `contract_negotiations`, `negotiation_rounds`, `negotiation_links`

| # | Route | Method | Table(s) Written | Operation | Auth | Freq |
|---|-------|--------|------------------|-----------|------|------|
| 12 | `/api/negotiations/create` | POST | contract_negotiations, negotiation_rounds | INSERT | Internal | On-demand |
| 13 | `/api/negotiations/[id]/send-round` | POST | all 3 tables | INSERT+UPDATE | Internal | Per-round |
| 14 | `/api/negotiations/[id]/generate-link` | POST | contract_negotiations, negotiation_links | UPDATE+INSERT | Internal | Per-negotiation |
| 15 | `/api/negotiations/[id]/finalize` | POST | contract_negotiations | UPDATE | Internal | Once |
| 16 | `/api/negotiations/[id]/cancel` | POST | contract_negotiations | UPDATE | Internal | On-demand |
| 17 | `/api/review/[token]/submit` | POST | all 3 tables | INSERT+UPDATE | Public (token) | Per-round |
| 18 | `/api/review/[token]` | GET | negotiation_links | UPDATE (view tracking) | Public (token) | Per-view |
| 19 | `/api/bot/negotiations/create` | POST | contract_negotiations, negotiation_rounds | INSERT | x-api-key | On-demand |
| 20 | `/api/bot/negotiations/[id]/generate-link` | POST | contract_negotiations, negotiation_links | INSERT+UPDATE | x-api-key | On-demand |
| 21 | `/api/bot/negotiations/[id]/finalize` | POST | contract_negotiations | UPDATE | x-api-key | On-demand |

**Critical note on #18 (review/[token] GET):** Updates `view_count` and `first_viewed_at` on negotiation_links table. Same issue as capture-view — freezing this blocks counterparty from even viewing the review page.

**Pattern for middleware match:**
```
/api/negotiations/*
/api/review/*/submit
/api/review/*  (GET write side-effect — special handling needed)
/api/bot/negotiations/*
```

## 5. Write-Path Inventory: Publisher/Buyer (Future @milo/crm)

**Tables affected:** `publishers`, `buyers`, `campaign_publishers`, `campaign_applications`

| # | Route | Method | Table(s) Written | Operation | Auth | Freq |
|---|-------|--------|------------------|-----------|------|------|
| 22 | `/api/bot/partners` | POST | publishers or buyers | INSERT | x-api-key | On-demand |
| 23 | `/api/bot/partners/create` | POST | publishers or buyers | INSERT | x-api-key | On-demand |
| 24 | `/api/bot/partners/[id]/archive` | POST | publishers or buyers | UPDATE | x-api-key | Rare |
| 25 | `/api/bot/campaigns/[id]/publishers` | POST | campaign_publishers | INSERT | x-api-key | On-demand |
| 26 | `/api/public/campaigns/verify-publisher` | POST | campaign_publishers, campaign_applications | INSERT/UPDATE | Public | Low |

**Note:** These are for future @milo/crm migration. Not urgent to freeze now — that migration is blocked on MOP code unfreeze anyway (TASKS.md). Include in the mechanism design for when it's needed.

## 6. Recommended Freeze Mechanism: Option C

### Why Option C (middleware pattern matching)

| Option | Pros | Cons | Verdict |
|--------|------|------|---------|
| A: New env var in middleware + route patterns | Reuses proven pattern, single deploy toggle | Slightly complex regex | **RECOMMENDED** |
| B: Per-route guard imports | Fine-grained control | 26 files to modify, easy to miss one | No |
| C: Same as A (Option C = middleware) | — | — | Same as A |

Options A and C are the same concept. The middleware approach wins because:
1. **Single point of control** — one env var toggles all contract writes
2. **Proven pattern** — `MOP_WRITES_FROZEN` already works this way
3. **Minimal code change** — ~15 lines added to existing middleware.ts
4. **Deploy-toggled** — Netlify env var change + redeploy, no code commit needed to activate
5. **No route-file modifications** — frozen codebase stays frozen

### Implementation

```typescript
// middleware.ts — additions (insert after existing MOP_WRITES_FROZEN block)

const CONTRACT_FREEZE_PATHS = [
  // Contract analysis
  '/api/contract/analyze-async',
  '/api/contract/analyze-process',
  '/api/contract/analyze-status',
  '/api/bot/contracts/analyze',
  '/api/bot/contracts/compare',
  // Signing
  '/api/sign/',
  '/api/bot/documents/generate',
  '/api/bot/documents/',     // covers [id]/void
  '/api/bot/webhooks/refire',
  // Negotiation
  '/api/negotiations/',
  '/api/review/',
  '/api/bot/negotiations/',
  // Publisher/Buyer (future, enable when @milo/crm migration fires)
  // '/api/bot/partners',
  // '/api/bot/campaigns/',
  // '/api/public/campaigns/',
];

if (process.env.CONTRACT_WRITES_FROZEN === 'true') {
  const path = request.nextUrl.pathname;
  const isContractPath = CONTRACT_FREEZE_PATHS.some(p => path.startsWith(p));
  const isWriteMethod = ['POST', 'PATCH', 'PUT', 'DELETE'].includes(request.method);
  
  // Special: analyze-status is GET but writes (side-effect)
  const isGetWithSideEffect = path.startsWith('/api/contract/analyze-status')
    || path.match(/^\/api\/review\/[^/]+$/);  // review/[token] GET
  
  if (isContractPath && (isWriteMethod || isGetWithSideEffect)) {
    return NextResponse.json(
      {
        error: 'Contract writes suspended for data migration',
        frozen_at: '2026-04-20T...',
        contact: 'Mark — contract migration in progress',
        read_endpoints_available: true,
      },
      { status: 503, headers: { 'Retry-After': '3600' } }
    );
  }
}
```

### What Stays UNFROZEN

| Path | Why |
|------|-----|
| `/api/trackdrive/webhook` | TrackDrive call ingest — NOT a migration target |
| `/api/trackdrive/ping-post` | Ping/post logs — NOT a migration target |
| `/api/cron/sync-late-conversions` | Updates call_logs only — NOT a migration target |
| `/api/cron/sync-unconversions` | Updates call_logs only — NOT a migration target |
| `/api/invoices/*` | Invoice tables — NOT currently a migration target |
| `/api/outreach/*` | Outreach queue — NOT a migration target |
| `/api/bot/blacklist` | Blacklist — already migrated, MOP writes are legacy |
| `/api/offer/*` | Campaign offers — NOT a migration target |
| All GET routes (except noted) | Read paths always pass |

## 7. Financial Cron Re-Enable Plan

### What to Re-Enable

| Cron | Schedule (UTC) | Schedule (Denver) | Table Written | Operation |
|------|---------------|-------------------|---------------|-----------|
| sync-late-conversions | `0 15 * * *` | 9:00 AM daily | call_logs | UPDATE (revenue, payout, disposition) |
| sync-unconversions | `30 15 * * *` | 9:30 AM daily | call_logs | UPDATE (buyer_converted=false, revenue=0) |

### Where to Change

**File:** `/Users/markymark/MOP/netlify.toml`

```toml
# Current (frozen):
[functions."sync-late-conversions"]
  # schedule = "0 15 * * *"

[functions."sync-unconversions"]
  # schedule = "30 15 * * *"

# Target (re-enabled):
[functions."sync-late-conversions"]
  schedule = "0 15 * * *"

[functions."sync-unconversions"]
  schedule = "30 15 * * *"
```

### Dependencies Check

| Dependency | Status | Risk |
|------------|--------|------|
| TrackDrive API (`fetchCalls()`) | Active (TrackDrive is live) | NONE |
| `supabaseAdmin` client | Active (MOP Supabase accessible) | NONE |
| `call_logs` table | Active, receiving webhook writes | NONE |
| `APP_TIMEZONE` constant | Unchanged | NONE |
| CRON_SECRET env var | Set in Netlify (verified at freeze time) | NONE |

### Risk Assessment

**Risk: LOW.** These crons:
- Only UPDATE existing `call_logs` rows (never INSERT)
- Target mutable fields (revenue, payout, disposition) that are expected to change
- Don't interact with any migration-target tables
- Don't conflict with TrackDrive webhook writes (they update different fields)
- Have been tested in production for months before freeze
- Use batch sizes of 20 (won't overwhelm DB)

**One consideration:** 30+ days of un-synced late conversions will queue up. First run will process the full backlog (30 days × ~N calls/day). The batch processing (20 at a time) and timeout handling (newest-first for resilience) should handle this, but first-run may timeout and need 2-3 invocations to catch up.

### Deploy Sequence for Cron Re-Enable

1. Uncomment the 2 schedule lines in `netlify.toml`
2. Commit to MOP's deployed branch
3. Netlify auto-deploys
4. First scheduled run: sync-late-conversions at 9:00 AM Denver next day
5. Monitor: Check Netlify function logs for first successful run
6. Verify: `SELECT COUNT(*) FROM call_logs WHERE updated_at > '2026-04-20'` — should show updated rows

## 8. Active Signing Sessions: Pre-Freeze Check

Before deploying `CONTRACT_WRITES_FROZEN=true`, verify no signing tokens are in active use:

```sql
-- Active signing sessions (tokens that have been sent but not yet signed/declined)
SELECT id, entity_name, document_type, status, created_at,
       viewed_at, signing_token
FROM signing_documents
WHERE status IN ('pending', 'viewed')
ORDER BY created_at DESC;
```

**If active tokens exist:**
- Option 1: Wait for them to complete (sign/decline) before freezing
- Option 2: Freeze anyway — signers get a 503 with "contact Mark" message
- Option 3: Exempt `/api/sign/[token]/submit` and `/api/sign/[token]/capture-view` from the freeze until those specific tokens resolve, then tighten

**Recommendation:** Check the query. If 0 rows in `pending`/`viewed` status, freeze immediately. If active tokens exist, wait for resolution (typically <48 hours for a signing token) or communicate to counterparties that signing is temporarily paused.

## 9. Deploy Sequence (Full Plan)

```
Step 1: Re-enable financial crons (safe, independent)
  └─ Uncomment 2 lines in netlify.toml → deploy
  └─ Wait 1 day to confirm runs succeed

Step 2: Check active signing sessions (query above)
  └─ If 0 active → proceed immediately
  └─ If active → wait for resolution or accept 503 risk

Step 3: Deploy CONTRACT_WRITES_FROZEN code change
  └─ PR to MOP with middleware.ts addition (~15 lines)
  └─ Set CONTRACT_WRITES_FROZEN=false initially (inactive)
  └─ Deploy to production

Step 4: Activate freeze (pre-migration)
  └─ Set CONTRACT_WRITES_FROZEN=true in Netlify env vars
  └─ Redeploy (env change triggers rebuild)
  └─ Verify: POST /api/bot/contracts/analyze → 503
  └─ Verify: POST /api/trackdrive/webhook → 200 (still flowing)

Step 5: Execute contract data migrations (Coder-2)
  └─ analysis_records (33 rows + 22 files)
  └─ signing_documents (109 rows)
  └─ contract_negotiations + rounds + links

Step 6: Post-migration decision
  └─ Option A: Keep freeze permanent (route contract work through milo-ops)
  └─ Option B: Redirect MOP endpoints to write to shared Supabase
  └─ Option C: Remove freeze, accept dual-write (not recommended)
```

## 10. Rollback Plan

### If freeze breaks something unexpected

```
1. Set CONTRACT_WRITES_FROZEN=false in Netlify env vars
2. Trigger redeploy
3. All contract write paths immediately restored
4. Time to rollback: ~2 minutes (env change + deploy)
```

### If financial crons cause issues

```
1. Re-comment the 2 schedule lines in netlify.toml
2. Commit and deploy
3. Crons stop on next schedule window
4. Time to rollback: ~3 minutes (commit + deploy)
```

### If middleware code has a bug

```
1. Revert the middleware.ts commit
2. Deploy previous version
3. Or: just set CONTRACT_WRITES_FROZEN=false (deactivates without code revert)
```

## 11. Open Questions for Mark

1. **Active signing tokens:** Should we query MOP's signing_documents for active tokens before freezing, or accept the risk? If TrackDrive emergency was worth an unfreeze, a signing token might be worth waiting 48 hours.

2. **Review page GET side-effect:** Freezing `/api/review/[token]` GET means counterparties can't even VIEW a negotiation review page. Should we exempt view-tracking writes from the freeze? They only update `view_count`/`first_viewed_at` on `negotiation_links` — low migration risk.

3. **Publisher/buyer paths:** Include in CONTRACT_WRITES_FROZEN now (commented out in implementation) or wait for @milo/crm migration to be imminent? Currently those paths are `x-api-key` only (PPCRM bot), and PPCRM is frozen anyway.

4. **Cron re-enable approval:** Can Coder-1 (or whoever has MOP deploy access) uncomment the 2 netlify.toml lines and deploy? This is the only code change that doesn't require CONTRACT_WRITES_FROZEN infrastructure.

5. **Who deploys the middleware change?** This is a code modification on MOP (frozen codebase). It's a minimal, surgical change (~15 lines) but breaks the "no code ships" covenant of the code freeze. Is this acceptable because it's freeze infrastructure, not feature code?

---

## Appendix A: Route Pattern Summary

All 26 routes organized by freeze group:

```
CONTRACT_WRITES_FROZEN scope:
├── Contract Analysis (5 routes)
│   ├── POST /api/contract/analyze-async
│   ├── POST /api/contract/analyze-process
│   ├── GET  /api/contract/analyze-status/[jobId]  ← side-effect write
│   ├── POST /api/bot/contracts/analyze
│   └── POST /api/bot/contracts/compare
├── Signing (6 routes)
│   ├── POST /api/sign/[token]/submit
│   ├── POST /api/sign/[token]/decline
│   ├── POST /api/sign/[token]/capture-view
│   ├── POST /api/bot/documents/generate
│   ├── POST /api/bot/documents/[id]/void
│   └── POST /api/bot/webhooks/refire
├── Negotiation (10 routes)
│   ├── POST /api/negotiations/create
│   ├── POST /api/negotiations/[id]/send-round
│   ├── POST /api/negotiations/[id]/generate-link
│   ├── POST /api/negotiations/[id]/finalize
│   ├── POST /api/negotiations/[id]/cancel
│   ├── POST /api/review/[token]/submit
│   ├── GET  /api/review/[token]  ← side-effect write
│   ├── POST /api/bot/negotiations/create
│   ├── POST /api/bot/negotiations/[id]/generate-link
│   └── POST /api/bot/negotiations/[id]/finalize
└── Publisher/Buyer — FUTURE (5 routes, commented out)
    ├── POST /api/bot/partners
    ├── POST /api/bot/partners/create
    ├── POST /api/bot/partners/[id]/archive
    ├── POST /api/bot/campaigns/[id]/publishers
    └── POST /api/public/campaigns/verify-publisher

ALWAYS UNFROZEN (TrackDrive + operational):
├── POST /api/trackdrive/webhook        → call_logs
├── POST /api/trackdrive/ping-post      → ping_post_logs
├── POST /api/cron/sync-late-conversions → call_logs (UPDATE only)
├── POST /api/cron/sync-unconversions   → call_logs (UPDATE only)
├── *    /api/invoices/*                → invoices
├── *    /api/outreach/*                → outreach_queue
└── GET  (all read paths)               → no writes
```

## Appendix B: Middleware.ts Current State

**File:** `/Users/markymark/MOP/middleware.ts`
**Current behavior:**
- Matches: `/api/:path*`
- When `MOP_WRITES_FROZEN=true`: blocks ALL POST/PATCH/PUT/DELETE
- Exempt: `/api/bot/health`, `/api/health`, `/api/admin/unfreeze`
- Currently: `MOP_WRITES_FROZEN=false` (global unfreeze per Decision 45)

The selective freeze adds a SECOND check (after the global check passes) that narrows to contract paths only.
