---
directive: "Consumer readiness audit — @milo/ai-client, @milo/crm, @milo/blacklist"
lane: research
coder: Coder-3
started: 2026-04-19 ~23:00 MDT
completed: 2026-04-19 ~23:45 MDT
---

# Consumer Readiness Audit — 2026-04-19

## 1. Executive Summary

Three `@milo/*` primitives are shipped or in-progress: `@milo/ai-client` (shipped), `@milo/crm` (building), `@milo/blacklist` (shipped). This audit maps every potential consumer repo, quantifies migration work, identifies blockers, and recommends a Coder-2 migration sequence.

**Key findings:**

- **@milo/ai-client** is the easiest win — 2 repos already migrated, milo-ops nearly done. MOP is small (~36 LOC). PPCRM is **permanently blocked** by Deno runtime incompatibility.
- **@milo/crm** is the heaviest lift — MOP alone has ~161 files touching CRM data (~10,000+ LOC). PPCRM has an 80% complete adapter (`crm-reader.ts`) but is frozen.
- **@milo/blacklist** has a clean extraction path — milo-outreach is the ideal first consumer (greenfield, no legacy tables). milo-ops has the richest implementation (29 files, multi-signal screening).
- **Deno blocker** affects PPCRM and pepperbooks — both run Supabase Edge Functions on Deno runtime and cannot import Node-based `@milo/*` packages directly.
- **Two repos not found on disk:** va-time-tracker, ap-platform-mvp. Gap in audit coverage.

## 2. Per-Primitive Consumer Tables

### 2.1 @milo/ai-client

| Consumer | Status | Files | LOC | Blockers | Migration LOC | Priority |
|----------|--------|-------|-----|----------|---------------|----------|
| mysecretary | **Migrated** | 0 remaining | 0 | None — fully on @milo/ai-client | 0 | Done |
| milo-outreach | **Migrated** | 0 remaining | 0 | None — fully on @milo/ai-client | 0 | Done |
| milo-ops | 95% done | ~2 files | ~10 | Minor: a few direct Anthropic SDK calls remain | ~10 | P1 (trivial) |
| MOP | Not started | 9 files | ~36 | MOP frozen (MOP_WRITES_FROZEN=true). Calls are simple `Anthropic()` instantiations | ~36 | P2 (post-unfreeze) |
| PPCRM | **Blocked** | 6+ files | ~50 | Deno runtime — cannot import Node @milo/ai-client. Would need Deno-compatible wrapper or HTTP proxy | N/A | Blocked |
| contract-checker | N/A | — | — | Uses OpenAI, not Anthropic. Not a consumer. | 0 | None |
| pepperbooks | **Blocked** | — | — | Deno runtime (same as PPCRM) | N/A | Blocked |

**Migration effort: ~46 LOC across 2 actionable repos (milo-ops, MOP).**

### 2.2 @milo/crm

| Consumer | Status | Files | LOC | Blockers | Migration LOC | Priority |
|----------|--------|-------|-----|----------|---------------|----------|
| MOP | Not started | 161 files | ~10,000+ | Heaviest target. Full CRUD on `publishers`, `buyers`, `contacts`. Split entity model (pub/buyer) vs @milo/crm unified `crm_counterparties`. Every publisher/buyer query needs rewriting. MOP frozen. | ~3,000-5,000 (adapter layer) | P1 (largest consumer, highest ROI) |
| PPCRM | 80% migrated | ~20 files | ~800 | `crm-reader.ts` adapter reads from `crm_counterparties`. Write path still hits legacy tables. PPCRM frozen (PPCRM_WRITES_FROZEN=true). 11 pg_cron jobs disabled. | ~200 (write path) | P2 (finish adapter) |
| milo-ops | Indirect | ~15 files | ~500 | Accesses CRM data via API clients to MOP/PPCRM, not direct DB. When MOP migrates, milo-ops gets it free via API. | ~100 (update API response types) | P3 (follows MOP) |
| milo-outreach | Zero | 0 | 0 | No CRM integration yet. Future consumer when outreach campaigns need counterparty data. | 0 | Future |
| mysecretary | Zero | 0 | 0 | No CRM data usage. | 0 | None |
| contract-checker | Zero | 0 | 0 | Standalone analyzer, no CRM. | 0 | None |

**Migration effort: ~3,300-5,300 LOC across 3 actionable repos. MOP dominates.**

### 2.3 @milo/blacklist

| Consumer | Status | Files | LOC | Blockers | Migration LOC | Priority |
|----------|--------|-------|-----|----------|---------------|----------|
| milo-outreach | Ideal first | 0 (greenfield) | 0 | None — no legacy blacklist code. Can import @milo/blacklist directly for pre-send screening. | ~50 (integration) | **P0 (best first consumer)** |
| milo-ops | Richest impl | 29 files | ~1,500 | Multi-signal screening engine (domain, email, phone, company name). Already implements fuzzy matching. Needs migration to @milo/blacklist schema (`blacklist_entries` table) from current custom tables. | ~800 (schema migration + adapter) | P1 |
| MOP | Heavy legacy | 50+ files | ~2,350 | Split tables: `master_blacklist_publishers` + `master_blacklist_buyers`. Needs unification into `blacklist_entries`. MOP frozen. | ~1,200 (table unification + query rewrite) | P2 (post-unfreeze) |
| PPCRM | Frozen/coupled | 11 files | ~400 | Reads from MOP blacklist tables via direct Supabase queries. Deno runtime blocks @milo/blacklist import. PPCRM frozen. | N/A (Deno blocked + frozen) | Blocked |
| mysecretary | Zero | 0 | 0 | No blacklist usage. | 0 | None |
| contract-checker | Zero | 0 | 0 | No blacklist usage. | 0 | None |

**Migration effort: ~2,050 LOC across 3 actionable repos. milo-outreach is the no-risk pilot.**

## 3. Cross-Cutting Blockers

### 3.1 Deno Runtime Incompatibility

PPCRM and pepperbooks run Supabase Edge Functions on Deno. All `@milo/*` packages are Node/TypeScript. Options:

| Option | Effort | Trade-off |
|--------|--------|-----------|
| Deno-compatible build of @milo/* | High (~2 weeks) | Dual-target build adds maintenance burden |
| HTTP proxy service | Medium (~1 week) | Adds latency + infra. PPCRM calls proxy instead of importing directly |
| Migrate PPCRM off Deno | Very high | Rewrites 26 Edge Functions to Next.js API routes |
| Accept gap | Zero | PPCRM stays on direct Supabase queries, doesn't consume @milo/* |

**Recommendation:** Accept gap for now. PPCRM is frozen and approaching end-of-life as MOP absorbs its functionality. Not worth solving Deno compatibility for a sunset system.

### 3.2 MOP Freeze

MOP is frozen (`MOP_WRITES_FROZEN=true`, branch `claude/lead-penguin-mop-mvp-011CUoVx9tU8haWeq8nMGZ4p`). All migration work on MOP is post-unfreeze. This affects:
- @milo/ai-client: 9 files, ~36 LOC
- @milo/crm: 161 files, ~10,000+ LOC (the big one)
- @milo/blacklist: 50+ files, ~2,350 LOC

### 3.3 Split Entity Model (publishers/buyers vs crm_counterparties)

MOP's entire data model assumes separate `publishers` and `buyers` tables. @milo/crm uses unified `crm_counterparties` with an `entity_type` discriminator. Every MOP query that joins on `publisher_id` or `buyer_id` needs rewriting. This is the single largest migration cost.

### 3.4 Missing Repos

`va-time-tracker` and `ap-platform-mvp` were not found on disk. These may have blacklist or CRM usage that isn't captured in this audit.

## 4. Recommended Coder-2 Migration Sequence

Based on effort, risk, dependency ordering, and value delivered:

### Phase 1: Quick Wins (est. 1-2 days)

| Step | Primitive | Consumer | LOC | Rationale |
|------|-----------|----------|-----|-----------|
| 1a | @milo/ai-client | milo-ops | ~10 | Finish the last 5% — trivial, proves pipeline |
| 1b | @milo/blacklist | milo-outreach | ~50 | Greenfield integration, zero legacy risk, validates @milo/blacklist API surface |

### Phase 2: PPCRM Write Path (est. 2-3 days)

| Step | Primitive | Consumer | LOC | Rationale |
|------|-----------|----------|-----|-----------|
| 2 | @milo/crm | PPCRM | ~200 | Finish the crm-reader.ts adapter's write path. PPCRM already 80% migrated. Requires unfreezing write path or working on frozen branch. |

### Phase 3: milo-ops Blacklist (est. 3-5 days)

| Step | Primitive | Consumer | LOC | Rationale |
|------|-----------|----------|-----|-----------|
| 3 | @milo/blacklist | milo-ops | ~800 | Schema migration from custom tables to `blacklist_entries`. milo-ops has the richest multi-signal screening — validates @milo/blacklist handles real-world complexity. |

### Phase 4: MOP Migrations (est. 2-3 weeks, post-unfreeze)

| Step | Primitive | Consumer | LOC | Rationale |
|------|-----------|----------|-----|-----------|
| 4a | @milo/ai-client | MOP | ~36 | Trivial, do first to warm up on MOP codebase |
| 4b | @milo/blacklist | MOP | ~1,200 | Unify split blacklist tables, rewrite queries. Medium complexity. |
| 4c | @milo/crm | MOP | ~3,000-5,000 | The big one. Unify publishers/buyers into crm_counterparties. Touch 161 files. Do last because it's highest risk and benefits from lessons learned in 4a/4b. |

### Deferred (not recommended now)

| Item | Reason |
|------|--------|
| PPCRM @milo/ai-client | Deno blocked, PPCRM sunsetting |
| PPCRM @milo/blacklist | Deno blocked, PPCRM sunsetting |
| pepperbooks anything | Deno blocked |
| contract-checker | Uses OpenAI, not an @milo consumer |

## 5. Migration Effort Summary

| Primitive | Total Actionable LOC | Repos | Biggest Target |
|-----------|---------------------|-------|----------------|
| @milo/ai-client | ~46 | 2 (milo-ops, MOP) | MOP (36 LOC) |
| @milo/crm | ~3,300-5,300 | 3 (MOP, PPCRM, milo-ops) | MOP (~10,000+ LOC surface, ~3-5K migration) |
| @milo/blacklist | ~2,050 | 3 (milo-outreach, milo-ops, MOP) | MOP (~2,350 LOC surface, ~1,200 migration) |
| **Total** | **~5,400-7,400** | | |

## 6. Open Questions for Mark

1. **PPCRM end-of-life timeline?** If PPCRM sunsets within 3 months, Deno compatibility work is wasted. If it lives longer, the gap grows.

2. **MOP unfreeze date?** Phase 4 (the bulk of migration work) is blocked until MOP unfreezes. Should Coder-2 queue other work or wait?

3. **va-time-tracker and ap-platform-mvp repos** — are these active? Should they be scanned for @milo/* consumer potential?

4. **milo-ops blacklist migration scope** — milo-ops has a sophisticated multi-signal screening engine (domain + email + phone + company name matching). Should @milo/blacklist's API surface be expanded to match milo-ops's capabilities before migration, or should milo-ops keep its extra logic as a consumer-side extension?

5. **PPCRM crm-reader.ts write path** — PPCRM is frozen with `PPCRM_WRITES_FROZEN=true`. Should Coder-2 work on the write-path migration on the frozen branch (ready to deploy post-unfreeze), or wait for explicit unfreeze?

6. **contract-checker: OpenAI -> Anthropic?** contract-checker uses OpenAI for its analyzer. Is there intent to migrate it to Anthropic/@milo/ai-client, or does it stay on OpenAI?
