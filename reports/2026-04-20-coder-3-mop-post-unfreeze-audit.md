---
directive: "MOP post-unfreeze strategy audit"
lane: research
coder: Coder-3
started: 2026-04-20 ~04:30 MDT
completed: 2026-04-20 ~05:00 MDT
---

# MOP Post-Unfreeze Strategy Audit — 2026-04-20

## 1. Executive Summary

**Critical finding: The unfreeze is GLOBAL, not selective.**

Decision 45 set `MOP_WRITES_FROZEN=false` to restore TrackDrive webhook processing, but the freeze mechanism is an all-or-nothing middleware gate. **ALL write endpoints are now live**, including:
- Contract analysis creation (writes to `contract_analyses` — migration target)
- Signing document updates (writes to `signing_documents` — migration target)
- Negotiation workflows (writes to `contract_negotiations` — migration target)
- Publisher/buyer updates (writes to `publishers`/`buyers` — migration target)

**This creates a split-brain risk.** If Coder-2 executes contract data migrations while MOP is actively writing to those same tables, the shared Supabase will immediately become stale.

**Recommended strategy:** Accept the current state for TrackDrive continuity, but **fence migration-target tables before executing any data migration.** Two options: (1) selective route-level guards on contract/signing endpoints, or (2) execute migrations and immediately redirect MOP writes to shared Supabase (dual-write or proxy).

## 2. What's Actually Writing Post-Unfreeze

### Confirmed Active (external triggers, flowing NOW)

| Write Path | Table(s) | Trigger | Volume |
|------------|----------|---------|--------|
| `/api/trackdrive/webhook` | call_logs, route_daily_stats | TrackDrive callback | High — every call completion |
| `/api/trackdrive/ping-post` | ping_post_logs | TrackDrive callback | High — every buyer response |
| `/api/sign/[token]/submit` | signing_documents, publishers, buyers | User signs document | Low — ~1-2/week at current volume |
| `/api/sign/[token]/decline` | signing_documents | User declines | Rare |
| `/api/public/campaigns/accept` | campaign_publishers | Publisher self-serve | Low |
| `/api/public/campaigns/apply` | campaign_publishers | Publisher self-serve | Low |

### Available But Unlikely Without Operator Action

| Write Path | Table(s) | Trigger | Notes |
|------------|----------|---------|-------|
| `/api/bot/documents/generate` | signing_documents | PPCRM onboardbot | Only fires if PPCRM triggers onboarding |
| `/api/bot/contracts/analyze` | contract_analyses, contract_analysis_jobs | Bot API call | Only if analysis is triggered |
| `/api/bot/partners/create` | publishers, buyers | Bot API call | Only if PPCRM creates entities |
| `/api/negotiations/create` | contract_negotiations | Manual or bot | Only if negotiation initiated |
| `/api/invoices/*` | invoices, invoice_audit_log | Manual operator | Operator must explicitly act |
| `/api/bot/blacklist` | blacklist tables | Bot API call | Only if PPCRM adds entries |

### Confirmed DISABLED (crons commented out in netlify.toml)

| Cron | Table(s) | Schedule | Status |
|------|----------|----------|--------|
| scheduled-sync | call_logs | */15 min | DISABLED |
| sync-ping-post | ping_post_logs | */15 min | DISABLED |
| refresh-ping-agg | agg tables | */15 min | DISABLED |
| check-overdue | invoices | Daily 8am | DISABLED |
| generate-invoices | invoices | Monday 6am | DISABLED |
| sync-late-conversions | call_logs | Daily 9am | DISABLED |
| sync-unconversions | call_logs | Daily 9:30am | DISABLED |
| reconciliation | (webhooks only) | Daily 7am | DISABLED |

## 3. Data Accumulating Since Unfreeze

### Freeze Manifest Baseline (2026-04-19 ~14:50 MDT)

| Table | Rows at Freeze | Growing Post-Unfreeze? | Migration Target? |
|-------|---------------|----------------------|-------------------|
| ping_post_logs | 9,279,038 | **YES** (TrackDrive webhook) | No |
| call_logs | 30,239 | **YES** (TrackDrive webhook) | No |
| buyer_responses | 1,629,063 | Likely (ping/post) | No |
| signing_documents | 109 | **POSSIBLY** (if anyone signs) | **YES** → @milo/contract-signing |
| contract_analyses | 33 | **POSSIBLY** (if bot triggers) | **YES** → @milo/contract-analysis |
| publishers | 310 | **POSSIBLY** (signing backfill) | **YES** → @milo/crm |
| buyers | 574 | **POSSIBLY** (signing backfill) | **YES** → @milo/crm |
| campaigns | 61 | Unlikely | No |
| invoices | 53 | No (crons disabled) | No |
| contract_negotiations | (in negotiations table) | **POSSIBLY** | **YES** → @milo/contract-negotiation |

### Key Insight: High-Volume vs Migration-Target Split

The high-volume writes (ping_post_logs, call_logs, buyer_responses) are NOT migration targets. They're MOP's core product data and stay in MOP.

The migration targets (contract_analyses, signing_documents, publishers/buyers) receive LOW volume writes — maybe 1-5 new rows per week in the current operational tempo. This is good news: the split-brain window is narrow and low-velocity.

## 4. Split-Brain Risk Analysis

### Scenario: Coder-2 Executes Contract-Analysis Migration

1. Migration copies 33 `contract_analyses` rows → shared Supabase `analysis_records`
2. Next day, bot triggers `/api/bot/contracts/analyze` → creates analysis #34 in MOP only
3. Shared Supabase now has 33 rows, MOP has 34 — **split-brain**

### Risk Rating by Table

| Table | Write Frequency | Migration Planned | Split-Brain Risk |
|-------|----------------|-------------------|-----------------|
| contract_analyses | ~1/week | YES (audit complete) | **MEDIUM** |
| signing_documents | ~1-2/week | YES (audit complete) | **MEDIUM** |
| contract_negotiations | ~1/week | YES (audit complete) | **MEDIUM** |
| publishers/buyers | ~1/month (backfill) | YES (blocked on code unfreeze) | **LOW** |
| call_logs | ~500+/day | No | N/A |
| ping_post_logs | ~1000+/day | No | N/A |

### Mitigation Options

| Option | Complexity | Risk Eliminated | Recommended? |
|--------|-----------|-----------------|--------------|
| A: Route-level guards on contract endpoints | LOW | Full | **YES** |
| B: Dual-write (MOP writes to both DBs) | HIGH | Full | No — requires code change on frozen codebase |
| C: Accept drift, reconcile post-migration | MEDIUM | Partial | Acceptable given low volume |
| D: Re-freeze MOP entirely during migration window | LOW | Full | No — breaks TrackDrive |

**Recommended: Option A + C hybrid.** Add route-level guards (or narrow freeze) on contract/signing/negotiation endpoints specifically, keeping TrackDrive flowing. If a few rows sneak through before guards deploy, reconcile with a delta migration (low volume makes this trivial).

## 5. MOP's Strategic Future

### Option Analysis

| Future | Probability | Implications |
|--------|-------------|--------------|
| **Stay alive as TLP's TrackDrive sink** | HIGH (near-term) | MOP's core value is call routing + ping/post. This is vertical-specific, not extractable. MOP continues indefinitely for this. |
| **Migrate TrackDrive handler to microservice** | LOW (no need) | TrackDrive webhooks are tightly coupled to MOP's call_logs + route aggregation. Extracting just the webhook gains nothing. |
| **Fold into Milo-for-PPC vertical** | MEDIUM (long-term) | When milo-starter produces the TLP vertical, MOP's TrackDrive/ping-post data becomes the vertical's operational core. MOP effectively IS the TLP vertical's backend. |
| **Decommission after extraction** | LOW | Can't decommission while TrackDrive is live. Decommission only if TLP vertical replaces MOP entirely. |

### Recommended Long-Term Path

```
NOW ─────────────────────────────────────────────── FUTURE
 │                                                    │
 │  MOP (unfrozen, code-frozen)                      │
 │  ├─ TrackDrive webhook → call_logs (stays)        │
 │  ├─ Ping/post webhook → ping_post_logs (stays)   │
 │  ├─ Signing endpoints (FENCE before migration) ──→ redirect to shared Supabase
 │  ├─ Contract analysis (FENCE before migration) ──→ redirect to shared Supabase
 │  └─ Crons (DISABLED, re-enable selectively)       │
 │                                                    │
 │  Eventually: MOP becomes TLP vertical's            │
 │  call-routing backend. Contract/signing/CRM        │
 │  fully migrated to @milo/* primitives.             │
```

## 6. Risk: Frozen Code with Flowing Writes

### What Breaks If Dependencies Update?

| Dependency | Risk | Timeline |
|------------|------|----------|
| Supabase JS client | LOW — pinned in package.json | Years |
| Next.js 14 | LOW — Netlify adapter is stable | 12-18 months |
| Netlify Functions runtime | MEDIUM — Node.js version EOL | ~18 months (Node 18 EOL Apr 2025, already past) |
| Supabase project (wjxtfjaixkoifdqtfmqd) | LOW — Supabase doesn't force-upgrade | Years |
| TrackDrive API contract | MEDIUM — if they change webhook format | Unknown |
| SSL certificates | AUTO — Netlify handles | N/A |

### Realistic Lifespan

**12-18 months without intervention.** The main risk is Node.js runtime deprecation on Netlify and TrackDrive API changes. Neither is imminent.

### Tech Debt Accumulation

With crons disabled:
- **Late conversion sync stopped** → CPA revenue numbers in call_logs become progressively stale (4-30 day lag not being reconciled)
- **Unconversion sync stopped** → Reversed calls not reflected
- **Invoice generation stopped** → AR/AP invoices not auto-generated (manual only)
- **Overdue detection stopped** → No automatic overdue flagging
- **Ping/post aggregation stopped** → Dashboard agg tables stale

**Impact:** Financial reporting from MOP's dashboard is degrading daily. The raw data (call_logs) is still correct (webhook writes it), but derived data (aggregations, conversions, invoices) is stale.

## 7. Migration Readiness: Contract Tables

### Current State

| Table | Rows at Freeze | Migration Audit | Can Migrate Now? |
|-------|---------------|-----------------|------------------|
| contract_analyses | 33 | Complete (8e4d146) | **YES — with fence** |
| signing_documents | 109 | Complete (a79fcf0) | **YES — with fence** |
| contract_negotiations + rounds + links | ~50 | Complete (2c6f7dd) | **YES — with fence** |

### Pre-Migration Checklist

Before Coder-2 executes any contract data migration:

1. **Deploy route-level fence** on MOP:
   ```typescript
   // In each contract write endpoint:
   if (process.env.CONTRACT_WRITES_FROZEN === 'true') {
     return NextResponse.json(
       { error: 'Contract writes temporarily suspended for migration' },
       { status: 503 }
     );
   }
   ```
   Affected routes:
   - `/api/bot/contracts/analyze` (contract_analyses)
   - `/api/bot/documents/generate` (signing_documents)
   - `/api/sign/[token]/submit` (signing_documents)
   - `/api/sign/[token]/decline` (signing_documents)
   - `/api/negotiations/create` (contract_negotiations)
   - `/api/negotiations/[id]/finalize` (contract_negotiations)

2. **Verify no in-flight transactions:** Check for pending signing tokens (signing_documents where status='pending' and viewed_at IS NOT NULL). These represent active signing sessions.

3. **Execute migration** (analysis → signing → negotiation, per ordering)

4. **Post-migration:** Either redirect MOP contract endpoints to write to shared Supabase, or keep fence permanent and route new contract work through milo-ops directly.

### Alternative: Accept-and-Reconcile

Given the low write volume (~1-5 contract rows/week), an alternative is:
1. Execute migration now (no fence)
2. After migration, run a delta query: "find rows in MOP with created_at > migration_timestamp"
3. Migrate delta rows

This is simpler but less clean. Works because volume is trivially low.

## 8. Should Writes Be Redirected NOW?

**No.** Redirecting writes requires code changes on MOP (frozen codebase), which contradicts the code freeze. The correct sequence is:

1. Execute data migration (one-time copy from MOP → shared)
2. Deploy write fence on contract endpoints (env var toggle, minimal code change)
3. Route new contract/signing work through milo-ops (already the orchestrator)
4. Leave TrackDrive webhook untouched (not a migration target)

The write fence is the only code change needed on MOP, and it's a single env var check — minimal blast radius on a frozen codebase.

## 9. Disabled Crons: Re-Enable Strategy

### Should Any Crons Be Re-Enabled?

| Cron | Impact of Being Disabled | Re-Enable? |
|------|-------------------------|------------|
| scheduled-sync | Redundant with webhook (webhook is primary ingest) | NO — webhook handles it |
| sync-ping-post | Redundant with ping-post webhook | NO — webhook handles it |
| refresh-ping-agg | Dashboard agg tables stale | **MAYBE** — if operators use MOP dashboard |
| sync-late-conversions | CPA revenue stale (4-30 day lag) | **YES** — financial accuracy |
| sync-unconversions | Reversed calls not reflected | **YES** — financial accuracy |
| check-overdue | No overdue flagging | LOW — manual workaround exists |
| generate-invoices | No auto-invoicing | LOW — manual workaround exists |
| reconciliation | No anomaly detection | LOW — informational only |

**Recommendation:** Re-enable `sync-late-conversions` and `sync-unconversions` for financial accuracy. These update existing `call_logs` rows (not migration targets) and don't interact with contract tables. Uncomment their schedules in `netlify.toml` and redeploy.

## 10. Open Questions for Mark

1. **Route-level fence vs accept-and-reconcile:** For contract data migration, do you want a clean fence (deploy `CONTRACT_WRITES_FROZEN=true` env var on MOP before migration) or is accept-and-reconcile acceptable given ~1-5 rows/week volume?

2. **Cron re-enablement:** Should we re-enable `sync-late-conversions` and `sync-unconversions` now? They're causing CPA revenue drift in call_logs. Requires uncommenting 2 lines in `netlify.toml` and redeploying MOP (no code change, just config).

3. **PPCRM→MOP interaction post-unfreeze:** PPCRM is still frozen but if it unfreezes, its onboardbot calls MOP's `/api/bot/documents/generate` to create signing documents. Should the signing fence block PPCRM too, or should PPCRM's onboarding flow remain operational?

4. **MOP dashboard usage:** Are operators still using MOP's dashboard for call routing analytics? If yes, `refresh-ping-agg` should be re-enabled. If they've moved to milo-ops dashboards, MOP's agg tables can stay stale.

5. **Long-term MOP ownership:** When the TLP vertical launches on milo-starter, does MOP's TrackDrive webhook handler move to the new vertical's codebase, or does MOP stay running as a dedicated call-routing service alongside the vertical?

---

## Summary Decision Matrix

| Question | Answer | Action |
|----------|--------|--------|
| (a) What writes post-unfreeze? | ALL endpoints (global unfreeze), primarily TrackDrive webhooks + signing | See §2 |
| (b) Data accumulating? | High-volume in call_logs/ping_post (not migration targets); low-volume in contract tables (migration targets) | Monitor, fence before migration |
| (c) MOP's future? | Stay alive as TLP call-routing backend; contract/signing migrates away | No decommission planned |
| (d) Risk? | 12-18 month lifespan, stale financial data from disabled crons, low dependency risk | Re-enable 2 crons |
| (e) Migration readiness? | Ready — deploy route-level fence on contract endpoints first | Fence → migrate → redirect |
