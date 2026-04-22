# Coder-2 Report: Wave 1 milo-ops Consumer Migrations (Directive #20)

**Date**: 2026-04-20
**Lane**: Coder-2 (Migration)
**Task**: Execute Wave 1 consumer tasks D3, D4, A4, E3 from Coder-3's wave plan
**Status**: D3/D4 complete, A4 no-op, E3 blocked

---

## Commits

| SHA | Description |
|-----|-------------|
| `bb58ccf` | feat: add @milo/onboarding v0.1.0 as workspace dep + client wrapper |
| `755a234` | refactor(D4): replace PPCRM startOnboarding with @milo/onboarding startRun |
| `fe24717` | refactor(D3): onboarding reads now try @milo/onboarding before PPCRM |

## Task Results

### D4: milo-ops startOnboarding rewrite — COMPLETE

**Files changed:** `src/lib/publisher-onboarding.ts`
**LOC:** ~30 (15 insertions, 15 deletions)

Replaced PPCRM `startOnboarding()` proxy call with `@milo/onboarding` `startRun()` via new `onboarding-client.ts` wrapper. New onboarding runs now write directly to shared Supabase (`onboarding_runs` + `onboarding_steps`) instead of calling PPCRM's `onboardbot/start` edge function.

Key changes:
- Import switched from `ppcrm-client.startOnboarding` to `onboarding-client.startPublisherOnboarding`
- `ppcrmPartnerId` references replaced with `onboardingRunId`
- Pipeline entity ID passed as `counterpartyId` (UUID preserved per Decision 32)
- TLP publisher 12-step flow auto-defined on first use (mirrors PPCRM checklist)

### D3: milo-ops onboarding reads — COMPLETE

**Files changed:** `src/app/api/ppcrm/route.ts`, `src/app/api/milo/route.ts`, `src/lib/milo-tools.ts`
**LOC:** ~44 insertions, ~26 deletions

Three onboarding read paths updated with dual-source strategy:

| Read path | Before | After |
|-----------|--------|-------|
| PPCRM proxy route (`/api/ppcrm?action=onboarding_status`) | PPCRM only | @milo/onboarding → PPCRM fallback |
| Milo chat intent handler (`ONBOARDING_STATUS`) | MOP → PPCRM fallback | @milo/onboarding → MOP → PPCRM fallback |
| AI tool (`getPPCRMStatusTool`) | PPCRM only | @milo/onboarding → PPCRM fallback |

The adapter layer (`onboarding-client.ts:mapRunStateToCompat`) transforms `@milo/onboarding` RunState into PPCRM-compatible shape, so the EntityDetailPanel UI renders identically regardless of data source. Response includes `source: 'shared'|'ppcrm'|'mop'` for debugging.

### A4: milo-ops contract display — NO-OP

milo-ops does NOT read from `contract_analyses` or `analysis_records` tables. Contract analysis is handled locally:
- `analyze-contract.ts` tool calls Anthropic directly
- Results stored in `contract_documents.analyzed_clauses`
- `mop-client.ts` has optional MOP history reads (not table-level)

No consumer migration needed. `@milo/contract-analysis` shared tables are consumed by MOP (Wave 2, blocked on code unfreeze).

### E3: milo-ops CRM write completion — BLOCKED

`@milo/crm` is NOT imported in milo-ops. All writes go to TLP operational tables:

| Table | Writes found | @milo/crm equivalent |
|-------|-------------|---------------------|
| `contacts` | 14 write sites | `crm_contacts` (different schema) |
| `prospect_pipeline` | 16+ write sites | None in @milo/crm |
| `campaign_routes` | 1 write site | None in @milo/crm |
| `agreed_terms` | 3 write sites | None in @milo/crm |
| `crm_counterparties` | 0 writes | @milo/crm table |
| `crm_leads` | 0 writes | @milo/crm table |
| `crm_contacts` | 0 writes | @milo/crm table |

**Blocker:** @milo/crm v0.1.x only has read operations. Write paths (create/update counterparties, contacts, leads) require @milo/crm v0.2 which is not yet built. The `contacts` → `crm_contacts` migration also requires schema reconciliation — milo-ops `contacts` has fields (`entity_type`, `research_status`, `personal_notes`) that don't exist in `crm_contacts`.

---

## Infrastructure: @milo/onboarding client wrapper

**File:** `src/lib/onboarding-client.ts` (~165 LOC)

| Feature | Detail |
|---------|--------|
| Singleton pattern | Same as `blacklist-client.ts` — cached `OnboardingClient` instance |
| Env vars | `MILO_ENGINE_SUPABASE_URL`, `MILO_ENGINE_SUPABASE_SERVICE_KEY` |
| Tenant | `tlp` |
| Flow auto-define | `ensureTlpPublisherFlow()` — defines 12-step flow on first use if absent |
| Adapter | `mapRunStateToCompat()` — RunState → PPCRM-compatible shape for UI |
| Copy script | `copy-milo-packages.mjs` updated with onboarding entry |
| transpilePackages | `next.config.ts` includes `@milo/onboarding` |

### TLP Publisher Flow Steps (auto-defined)

| # | Key | Label | Integration |
|---|-----|-------|-------------|
| 1 | add_to_mop | Add to MOP | — |
| 2 | msa_sent | MSA sent | contract_signing (msa) |
| 3 | msa_signed | MSA signed | — |
| 4 | w9_sent | W-9 sent | contract_signing (w9) |
| 5 | w9_signed | W-9 signed | — |
| 6 | creatives_submitted | Creatives submitted | file_upload |
| 7 | creatives_approved | Creatives approved | — |
| 8 | did_provisioned | DID provisioned | — |
| 9 | config_validated | Config validated | custom |
| 10 | io_sent | IO sent | contract_signing (io) |
| 11 | io_signed | IO signed | — |
| 12 | go_live | Go live | — |

---

## Build Verification

`npm run build` passes clean after all 3 commits.

## Wave 1 Summary

| Task | Status | LOC | Commits |
|------|--------|-----|---------|
| D3 (onboarding reads) | Complete | ~70 net | fe24717 |
| D4 (startOnboarding) | Complete | ~30 net | 755a234 |
| A4 (contract display) | No-op | 0 | — |
| E3 (CRM writes) | Blocked on @milo/crm v0.2 | 0 | — |
| Infrastructure | Complete | ~165 | bb58ccf |
| **Total** | | **~265** | **3 commits** |

Wave plan estimated ~1,000 LOC for Wave 1. Actual: ~265 LOC. The gap is because:
- A4 was a no-op (milo-ops doesn't consume contract_analyses)
- E3 is blocked (requires @milo/crm v0.2 write APIs + schema reconciliation)
- D3 adapter pattern is more efficient than full UI rewrite (~70 LOC vs estimated ~300)

---

*Coder-2 — Migration lane*
