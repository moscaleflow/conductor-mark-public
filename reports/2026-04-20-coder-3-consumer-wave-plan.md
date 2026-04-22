---
directive: "Post-migration consumer wave planning"
lane: research
coder: Coder-3
started: 2026-04-20 ~08:30 MDT
completed: 2026-04-20 ~09:00 MDT
supplements:
  - reports/2026-04-19-coder-3-consumer-readiness.md (de8f97e)
  - reports/2026-04-20-coder-3-mop-post-unfreeze-audit.md (1b98346)
---

# Post-Migration Consumer Wave Plan — 2026-04-20

## 1. Executive Summary

| Metric | Value |
|--------|-------|
| Total consumer work units | 19 |
| Blocked on MOP code unfreeze | 8 |
| Blocked on PPCRM code unfreeze | 4 |
| Ready to execute now | 3 |
| Blocked on Mark decisions | 4 |
| Estimated total LOC (all waves) | ~2,800-4,200 |

**Biggest blocker:** MOP code freeze. 8 of 19 consumer tasks require modifying MOP source code (changing Supabase table references from local to shared). Until MOP code unfreezes, those consumers stay on legacy local tables.

**Recommended sequence for Coder-2:**
1. milo-ops contract reads (ready now, ~200 LOC)
2. milo-ops onboarding reads (ready now, ~300 LOC)
3. mysecretary @milo/crm write-paths (blocked on v0.2 features)
4. MOP analysis/signing/negotiation reads (blocked on code unfreeze)
5. PPCRM onboarding consumer (blocked on code unfreeze)

**Strategic insight:** The contract primitives are unusual — their PRIMARY consumer is MOP itself, and MOP is code-frozen. This means the data migrations create shared-Supabase truth, but MOP continues reading from its local copy until code unfreeze. No production impact until MOP's reads are switched.

## 2. Consumer Inventory by Primitive

---

### (a) @milo/contract-analysis Consumers

| # | Consumer | Repo | Code Path | Blocker | LOC | Risk | Priority |
|---|----------|------|-----------|---------|-----|------|----------|
| A1 | Contract analyzer UI (read) | MOP | app/contract-analyzer/* | MOP code unfreeze | ~400 | MED | Medium |
| A2 | Analysis history page | MOP | app/history/* | MOP code unfreeze | ~150 | LOW | Low |
| A3 | Bot analysis endpoint (read) | MOP | app/api/bot/contracts/[entity]/* | MOP code unfreeze | ~100 | LOW | Low |
| A4 | milo-ops contract display | milo-ops | (verify existence) | None — ready | ~200 | LOW | High |

**A1 — Contract Analyzer UI:** MOP's main contract analysis page reads `contract_analyses` via Supabase client. Post-migration, should read `analysis_records` on shared Supabase. Requires:
- Change Supabase client from MOP-local to shared (or add second client)
- Update table name: `contract_analyses` → `analysis_records`
- Add `tenant_id = 'tlp'` filter to all queries
- Update column references (5 columns moved to metadata)

**A4 — milo-ops contract display:** If milo-ops reads contract analysis data (e.g., for publisher vetting), it can be migrated immediately since milo-ops code is not frozen. Verify: does milo-ops actually read contract_analyses? (From prior audits: milo-ops proxies to MOP for analysis, may not read directly.)

---

### (b) @milo/contract-signing Consumers

| # | Consumer | Repo | Code Path | Blocker | LOC | Risk | Priority |
|---|----------|------|-----------|---------|-----|------|----------|
| B1 | Signing UI (render + capture) | MOP | app/sign/[token]/* | MOP code unfreeze | ~600 | HIGH | High |
| B2 | Document status dashboard | MOP | app/documents/* | MOP code unfreeze | ~200 | LOW | Low |
| B3 | Bot document generation | MOP | app/api/bot/documents/generate | MOP code unfreeze | ~300 | MED | High |
| B4 | Signature webhook handler | MOP | app/api/sign/[token]/submit | MOP code unfreeze | ~400 | HIGH | High |
| B5 | PPCRM signature-webhook | PPCRM | supabase/functions/signature-webhook | PPCRM unfreeze | ~200 | MED | Medium |
| B6 | milo-ops signing proxy | milo-ops | src/lib/publisher-onboarding.ts | None — ready | ~150 | LOW | Medium |

**B1 — Signing UI:** The most complex consumer. Public-facing signing pages render documents and capture signatures. Two migration strategies:
- **Option 1 (minimal):** Keep MOP as the signing UI host, point its DB queries to shared Supabase → requires changing Supabase client config + table names
- **Option 2 (long-term):** Extract signing UI into a shared Next.js route that any vertical can embed → high effort, deferred

**B3/B4 — Bot generation + submit:** These are the write paths that the contract-writes freeze blocks. Once freeze activates and data migrates, these endpoints need to write to shared Supabase instead of local. This is the most critical consumer migration for signing.

**B6 — milo-ops signing proxy:** milo-ops currently calls MOP's bot API for document generation. Post-migration, it could call @milo/contract-signing directly (bypassing MOP). Ready to execute but depends on decision: does MOP remain the signing service, or does milo-ops consume the primitive directly?

---

### (c) @milo/contract-negotiation Consumers

| # | Consumer | Repo | Code Path | Blocker | LOC | Risk | Priority |
|---|----------|------|-----------|---------|-----|------|----------|
| C1 | Negotiation UI (create/manage) | MOP | app/negotiations/* | MOP code unfreeze | ~300 | MED | Low |
| C2 | Review page (public token) | MOP | app/review/[token]/* | MOP code unfreeze | ~250 | MED | Low |
| C3 | Bot negotiation endpoints | MOP | app/api/bot/negotiations/* | MOP code unfreeze | ~200 | LOW | Low |
| C4 | PPCRM negotiate.ts | PPCRM | supabase/functions/onboardbot/negotiate.ts | PPCRM unfreeze | ~150 | LOW | Low |

**Low priority across the board.** Negotiation is the least-used contract sibling. Only ~15-20 negotiations exist. Consumer migration can wait until after signing consumers are handled.

---

### (d) @milo/onboarding Consumers

| # | Consumer | Repo | Code Path | Blocker | LOC | Risk | Priority |
|---|----------|------|-----------|---------|-----|------|----------|
| D1 | Onboarding checklist UI | PPCRM | (dashboard component) | PPCRM code unfreeze | ~300 | MED | Medium |
| D2 | Onboardbot step engine | PPCRM | supabase/functions/onboardbot/index.ts | PPCRM code unfreeze | ~800 | HIGH | High |
| D3 | milo-ops pipeline display | milo-ops | src/components/pipeline/* | None — ready | ~300 | LOW | High |
| D4 | milo-ops startOnboarding | milo-ops | src/lib/publisher-onboarding.ts | None — ready | ~200 | LOW | High |

**D2 — Onboardbot:** The 5,305 LOC onboardbot is PPCRM's core onboarding engine. Migrating it to consume @milo/onboarding means replacing all `onboarding_checklists` boolean-flag reads/writes with `onboarding_runs` + `onboarding_steps` API calls. This is the highest-effort consumer migration in the entire wave (~800 LOC of changes). Blocked on PPCRM code unfreeze.

**D3/D4 — milo-ops:** Ready to execute. milo-ops can read onboarding progress from shared Supabase `onboarding_runs`/`onboarding_steps` instead of calling PPCRM's status endpoint. This provides immediate value: milo-ops gets live onboarding status without depending on PPCRM.

---

### (e) @milo/crm Remaining Consumers

| # | Consumer | Repo | Code Path | Blocker | LOC | Risk | Priority |
|---|----------|------|-----------|---------|-----|------|----------|
| E1 | mysecretary write-paths | mysecretary | 7 functions (leads, contacts) | @milo/crm v0.2 features | ~400 | MED | Medium |
| E2 | MOP read-paths (publishers/buyers) | MOP | multiple components | MOP code unfreeze | ~500 | MED | Medium |
| E3 | milo-ops full CRM writes | milo-ops | src/lib/crm-*.ts | None — ready (partial) | ~300 | LOW | Medium |

**E1 — mysecretary write-paths:** Blocked on @milo/crm v0.2 (needs create/update operations on counterparties + leads, not just read). Per TASKS.md, deferred until Coder-1 ships v0.2.

**E2 — MOP read-paths:** MOP reads publishers/buyers tables everywhere (dashboard, stats, entity pages). Migration means switching 50+ queries from local `publishers`/`buyers` to shared `crm_counterparties`. Blocked on code unfreeze.

---

## 3. Recommended Execution Order for Coder-2

### Wave 1: Ready Now (no blockers)

| Task | LOC | Value Unlocked |
|------|-----|----------------|
| D3: milo-ops onboarding reads | ~300 | Live pipeline status without PPCRM dependency |
| D4: milo-ops startOnboarding rewrite | ~200 | Direct @milo/onboarding API instead of PPCRM proxy |
| A4: milo-ops contract display (if applicable) | ~200 | Shared analysis data in ops dashboard |
| E3: milo-ops CRM write completion | ~300 | Full CRM CRUD from ops |

**Total Wave 1:** ~1,000 LOC across milo-ops only (unfrozen, deployable)

### Wave 2: After MOP Code Unfreeze

| Task | LOC | Value Unlocked |
|------|-----|----------------|
| B3: Bot document generation → shared | ~300 | PPCRM onboarding uses shared signing |
| B4: Signature webhook → shared | ~400 | Signing events write to shared Supabase |
| B1: Signing UI → shared | ~600 | Public signing pages use shared data |
| A1: Contract analyzer → shared | ~400 | Analysis UI reads from shared |
| E2: MOP CRM reads → shared | ~500 | MOP uses shared counterparty data |

**Total Wave 2:** ~2,200 LOC, all in MOP. Requires MOP code unfreeze decision.

### Wave 3: After PPCRM Code Unfreeze

| Task | LOC | Value Unlocked |
|------|-----|----------------|
| D2: Onboardbot → @milo/onboarding | ~800 | PPCRM onboarding uses shared state |
| B5: PPCRM signature-webhook | ~200 | PPCRM receives signing events from shared |
| D1: Onboarding checklist UI | ~300 | PPCRM dashboard reads shared onboarding |
| C4: PPCRM negotiate.ts | ~150 | PPCRM negotiation uses shared |

**Total Wave 3:** ~1,450 LOC, all in PPCRM. Requires PPCRM code unfreeze.

### Wave 4: After @milo/crm v0.2

| Task | LOC | Value Unlocked |
|------|-----|----------------|
| E1: mysecretary write-paths | ~400 | mysecretary full CRM integration |

---

## 4. The MOP Strategy Question

### Current State

MOP has TWO roles:
1. **TrackDrive data sink** (call_logs, ping_post) — stays forever, not a migration target
2. **Contract/signing service** (analysis, signing, negotiation) — data migrating to shared

### Post-Migration Options

| Option | MOP Effort | Outcome |
|--------|-----------|---------|
| **A: MOP reads from shared** | ~2,200 LOC (Wave 2) | MOP stays alive, reads from shared Supabase. Two DB connections. |
| **B: MOP stays on local copy** | 0 LOC | MOP continues reading stale local data. Shared Supabase is source of truth for new verticals only. Old TLP data lives in both places. |
| **C: MOP decommissions contract features** | ~500 LOC (remove) | MOP becomes TrackDrive-only. Contract UI moves to milo-ops or new vertical. |

**Recommendation:** Option B for now, Option A when MOP code unfreezes. The data migration creates shared truth for NEW consumers (milo-ops, future verticals). MOP's existing UI can continue reading local until code unfreeze enables the switch. No production impact either way since the data is frozen (no new contract work happening during freeze).

---

## 5. What Requires Mark/Morgan Decisions

| Decision Needed | Options | Impact | Urgency |
|-----------------|---------|--------|---------|
| 1. MOP code unfreeze timing | Now / After TLP vertical / Never | Gates Wave 2 (8 tasks, ~2,200 LOC) | Medium |
| 2. PPCRM code unfreeze timing | Now / After onboarding redesign | Gates Wave 3 (4 tasks, ~1,450 LOC) | Low |
| 3. MOP's long-term signing role | MOP hosts UI / Extract to shared / Decommission | Determines B1 approach | Low |
| 4. milo-ops direct vs proxy | milo-ops calls @milo/* directly / Still proxies through MOP | Determines D4/B6 design | High (Wave 1) |

### Decision 4 Detail (Most Urgent)

Currently: `milo-ops → MOP bot API → MOP's Supabase`

Post-migration options:
- **Direct:** `milo-ops → @milo/contract-signing SDK → shared Supabase`
- **Proxy:** `milo-ops → MOP bot API → shared Supabase` (MOP rewired internally)

Direct is simpler (removes MOP as middleman for new operations) but means milo-ops needs to implement document rendering, signing token generation, etc. The @milo/contract-signing package may not expose these yet.

**Recommendation:** Proxy for now (keep MOP as signing service), direct later when @milo/contract-signing has a full SDK.

---

## 6. Dependency Graph

```
@milo/crm v0.2 ──────────────────────── E1 (mysecretary writes)
                                          
MOP code unfreeze ─┬── A1 (analyzer UI)
                   ├── A2 (history)
                   ├── A3 (bot reads)
                   ├── B1 (signing UI)
                   ├── B3 (bot generate)
                   ├── B4 (sign submit)
                   ├── C1-C3 (negotiation)
                   └── E2 (MOP CRM reads)

PPCRM code unfreeze ─┬── D1 (checklist UI)
                     ├── D2 (onboardbot)
                     ├── B5 (sig webhook)
                     └── C4 (negotiate.ts)

No blocker ──────────┬── D3 (milo-ops pipeline)
                     ├── D4 (milo-ops onboarding)
                     ├── A4 (milo-ops contracts)
                     └── E3 (milo-ops CRM writes)
```

---

## 7. Risk Matrix

| Risk | Probability | Impact | Mitigation |
|------|-------------|--------|------------|
| MOP code unfreeze breaks TrackDrive | LOW | HIGH | Deploy selective code changes, keep TrackDrive paths untouched |
| PPCRM onboardbot rewrite introduces bugs | MEDIUM | HIGH | Extensive testing with 8 existing checklists as fixtures |
| Dual-DB confusion (which source of truth?) | MEDIUM | MEDIUM | Clear documentation: shared Supabase = truth, MOP local = legacy cache |
| milo-ops direct integration breaks signing flow | LOW | MEDIUM | Keep proxy path as fallback |
| @milo/crm v0.2 delays gate Wave 4 | MEDIUM | LOW | mysecretary write-paths are low urgency |

---

## 8. Open Questions for Mark

1. **MOP code unfreeze timeline:** Is there a planned date or trigger? Wave 2 (2,200 LOC, 8 tasks) is entirely gated on this. If MOP stays frozen indefinitely, those consumers never migrate and MOP reads stale local data forever.

2. **milo-ops direct consumption (Decision 4):** Should Wave 1 have milo-ops consume @milo/onboarding and @milo/contract-signing SDKs directly, or continue proxying through PPCRM/MOP? Direct is cleaner but requires the packages to have full client APIs.

3. **PPCRM onboardbot effort justification:** D2 is ~800 LOC of changes to a 5,305 LOC file. Is this worth doing on the existing PPCRM codebase, or should it wait for the TLP vertical rebuild on milo-starter? If the latter, Wave 3 is deferred entirely.

4. **MOP's contract UI future:** When MOP code unfreezes, do we invest ~2,200 LOC into migrating its contract/signing UI to shared Supabase reads? Or accept that MOP's contract UI is legacy and build fresh UI in the TLP vertical?

5. **Wave 1 authorization:** Can Coder-2 begin Wave 1 (milo-ops onboarding + CRM work, ~1,000 LOC) immediately, or should it wait for all 4 data migrations to complete first?

---

## Summary

| Wave | LOC | Blocker | Consumers |
|------|-----|---------|-----------|
| 1 (milo-ops) | ~1,000 | None | D3, D4, A4, E3 |
| 2 (MOP) | ~2,200 | MOP code unfreeze | A1-A3, B1, B3-B4, C1-C3, E2 |
| 3 (PPCRM) | ~1,450 | PPCRM code unfreeze | D1-D2, B5, C4 |
| 4 (mysecretary) | ~400 | @milo/crm v0.2 | E1 |
| **Total** | **~5,050** | | **19 tasks** |
