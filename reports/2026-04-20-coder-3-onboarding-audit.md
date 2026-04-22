---
directive: "@milo/onboarding pre-extraction audit"
lane: research
coder: Coder-3
started: 2026-04-20 ~02:30 MDT
completed: 2026-04-20 ~03:00 MDT
urgency: HIGH — Coder-1 writing SCHEMA.md in parallel
---

# @milo/onboarding Pre-Extraction Audit — 2026-04-20

## 1. Executive Summary

PPCRM's onboarding system is a **12-step linear progression** controlled by boolean flags on an `onboarding_checklists` table. It is NOT a formal state machine — there's no backtracking, no branching, no event-sourced transitions. milo-ops provides a parallel orchestration layer that proxies document generation to MOP and triggers PPCRM's `startOnboarding()` endpoint.

**Extraction complexity: HIGH.** The onboarding logic is deeply coupled to:
- MOP (document generation + signing webhooks)
- TrackDrive (DID provisioning — manual operator step)
- CRM tables (publishers, buyers, campaigns)
- Teams webhooks (notifications at every step)
- pg_cron (stall detection + follow-up reminders)

**Recommended approach:** Extract the state machine + step orchestration as `@milo/onboarding`, but leave PPC-specific step implementations (creatives approval, DID provisioning, IO terms) as vertical config/plugins.

**Estimated LOC for v0.1.0:** ~1,800–2,200 (state machine + step interface + event hooks + schema)

## 2. Source Paths

| File | LOC | Role |
|------|-----|------|
| `PPCRM/supabase/functions/onboardbot/index.ts` | 5,305 | Core 12-step engine — all step logic inline |
| `PPCRM/supabase/functions/onboardbot/negotiate.ts` | 450 | Contract negotiation sub-workflow |
| `PPCRM/supabase/functions/signature-webhook/index.ts` | 412 | MOP callback handler (MSA/W9/IO signed events) |
| `PPCRM/supabase/functions/onboard-submit/index.ts` | 329 | Interactive form submission (creatives, config) |
| `milo-ops/src/lib/publisher-onboarding.ts` | ~400 | Central orchestration (3 entry paths) |
| `milo-ops/src/app/api/publisher-onboarding/route.ts` | ~150 | HTTP entrypoint |
| `milo-ops/src/components/AddProspectModal.tsx` | ~300 | Two-path entry UI (vetting vs trusted) |
| **Total relevant source** | **~7,350** | |

## 3. External Dependencies

| Dependency | Used By | How | Extractable? |
|------------|---------|-----|--------------|
| **MOP** (document signing) | onboardbot steps 2,4,9 | Generates MSA/W9/IO via bot API, receives signed webhooks | YES — @milo/contract-signing handles this |
| **TrackDrive** | onboardbot step 8 | DID provisioning (manual operator action) | NO — vertical-specific, operator-gated |
| **Teams webhooks** | onboardbot (all steps) | Notifications on progress/stalls | PARTIAL — generic notification interface extractable |
| **pg_cron** | Stall detection | Queries checklists where step stalled >3 days | YES — cron pattern is generic |
| **Supabase Storage** | onboard-submit | Creative asset uploads | NO — vertical-specific asset types |

## 4. Internal Coupling

| Coupled To | Nature | Severity |
|------------|--------|----------|
| `onboarding_checklists` table | Boolean flags for each step | HIGH — core state storage |
| `publishers` / `buyers` tables | FK lookups for entity context | MEDIUM — migrating to @milo/crm |
| `campaigns` table | Links onboarding to campaign setup | MEDIUM — PPC-specific |
| `signing_documents` table | MSA/W9/IO document status | LOW — migrating to @milo/contract-signing |
| `contract_analyses` table | Pre-signing analysis results | LOW — migrating to @milo/contract-analysis |

## 5. Horizontal Extraction Candidates

These components are **generic across verticals** (any onboarding workflow):

| Component | LOC Est. | Why Generic |
|-----------|----------|-------------|
| Step state machine (ordered steps, advance/block) | ~400 | Every vertical onboards entities through ordered gates |
| Step interface (name, guard condition, action, timeout) | ~150 | Step definition is domain-agnostic |
| Stall detection (timeout per step, escalation) | ~200 | Universal — any step can stall |
| Notification hooks (step started/completed/stalled) | ~150 | Channel-agnostic event emission |
| Checklist data model (entity_id, step states, timestamps) | ~100 | Schema is generic |
| Entry path routing (multiple triggers → same workflow) | ~150 | milo-ops already has 3 entry paths |
| Progress query API (% complete, current step, blockers) | ~200 | Dashboard/reporting universal need |

**Total horizontal:** ~1,350 LOC

## 6. Vertical Layer (PPC-specific, stays in consumer)

| Component | LOC Est. | Why Vertical |
|-----------|----------|--------------|
| Step definitions (12 PPC steps with specific guards) | ~800 | Steps are domain-specific |
| Creative approval workflow | ~300 | PPC-only asset type |
| DID provisioning integration | ~200 | TrackDrive is PPC-only |
| IO terms/pricing logic | ~250 | PPC billing model |
| Campaign linking on go-live | ~150 | PPC campaign structure |
| Teams webhook formatting | ~100 | Could be generic but message content is PPC |

**Total vertical:** ~1,800 LOC (stays in PPCRM/milo-ops)

## 7. State Machine Design (MOST IMPORTANT)

### Current Reality: NOT a State Machine

PPCRM uses a **flat boolean-flag table**:

```sql
CREATE TABLE onboarding_checklists (
  id UUID PRIMARY KEY,
  publisher_id UUID REFERENCES publishers(id),
  -- Boolean flags (one per step)
  added_to_mop BOOLEAN DEFAULT false,
  msa_sent BOOLEAN DEFAULT false,
  msa_signed BOOLEAN DEFAULT false,
  w9_sent BOOLEAN DEFAULT false,
  w9_signed BOOLEAN DEFAULT false,
  creatives_submitted BOOLEAN DEFAULT false,
  creatives_approved BOOLEAN DEFAULT false,
  did_provisioned BOOLEAN DEFAULT false,
  config_validated BOOLEAN DEFAULT false,
  io_sent BOOLEAN DEFAULT false,
  io_signed BOOLEAN DEFAULT false,
  go_live BOOLEAN DEFAULT false,
  -- Metadata
  created_at TIMESTAMPTZ,
  updated_at TIMESTAMPTZ,
  stalled_at TIMESTAMPTZ,
  notes TEXT
);
```

### Recommended Extraction: Ordered-Step Engine

```
@milo/onboarding state model:

  workflow_instance {
    id, tenant_id, entity_type, entity_id,
    workflow_definition_id,
    current_step_index,     -- integer position in ordered list
    status: 'active' | 'completed' | 'blocked' | 'cancelled',
    started_at, completed_at, blocked_at, blocked_reason
  }

  workflow_step_state {
    id, workflow_instance_id,
    step_key,               -- e.g. 'msa_sent', 'did_provisioned'
    step_index,             -- position in sequence
    status: 'pending' | 'in_progress' | 'completed' | 'skipped' | 'blocked',
    started_at, completed_at,
    timeout_hours,          -- per-step stall threshold
    metadata JSONB          -- step-specific data
  }

  workflow_definition {
    id, tenant_id, name, version,
    steps JSONB             -- ordered array of step configs
  }
```

### Why NOT a Full State Machine

A true state machine (states + transitions + guards) is overkill here because:
1. **No branching** — PPCRM steps are strictly linear (step N must complete before N+1)
2. **No backtracking** — once a step completes, it never reverts
3. **No parallel steps** — all sequential (though steps 3, 5, 10 auto-complete via webhook)
4. **No conditional paths** — every publisher goes through all 12 steps

An **ordered-step engine** with per-step status tracking is the right abstraction. It's simpler than a state machine but richer than boolean flags.

### Future Extensibility

If a future vertical needs branching (e.g., "if entity type is X, skip steps 4-6"), the `workflow_definition.steps` JSONB can support conditional skip rules without changing the core engine. But v0.1.0 should NOT implement this — YAGNI until a second vertical proves the need.

## 8. Data Model Summary

### Source Tables (PPCRM)

| Table | Rows (est.) | Migration Complexity |
|-------|-------------|---------------------|
| `onboarding_checklists` | ~200-500 | MEDIUM — boolean flags → step_state rows |
| (inline in publishers) | — | Some onboarding state lives on publisher record |

### Target Schema (recommended for @milo/onboarding)

3 tables:
- `onboarding_workflows` — workflow instance (replaces checklist row)
- `onboarding_step_states` — per-step status (replaces boolean columns)
- `onboarding_definitions` — workflow templates (new — enables multi-vertical)

### Migration Transform

Each PPCRM `onboarding_checklists` row becomes:
- 1 `onboarding_workflows` row (entity linkage + overall status)
- 12 `onboarding_step_states` rows (one per boolean flag, status derived from flag value)

This is a **fan-out migration** (1 row → 13 rows). Reversibility is straightforward since the boolean semantics are preserved in step status.

## 9. Dependencies on Other @milo/* Primitives + Ordering

```
@milo/crm (v0.1.0 shipped)
  └── entity_id references in workflow_instance

@milo/contract-signing (v0.1.0 shipped)
  └── steps 2-5 and 9-10 delegate to signing
  └── webhook callbacks advance step status

@milo/contract-analysis (v0.1.0 shipped)
  └── optional pre-step: analysis before MSA generation

@milo/contract-negotiation (v0.1.0 shipped)
  └── optional: negotiation between send and sign

@milo/blacklist (v0.1.0 shipped)
  └── milo-ops scam-check gate runs BEFORE workflow starts
  └── NOT a step — it's a pre-entry filter
```

**Extraction order:**
1. @milo/crm ✅ (shipped)
2. @milo/contract-signing ✅ (shipped)
3. @milo/contract-negotiation ✅ (shipped)
4. **@milo/onboarding** ← HERE (depends on 1-3 being available)

@milo/onboarding is correctly positioned as the LAST primitive in the contract pipeline. It orchestrates the others but doesn't need to extract their logic.

## 10. Recommended Extraction Sequence

### Phase 1: Schema + Core Engine (~1,200 LOC)
1. Define `onboarding_workflows`, `onboarding_step_states`, `onboarding_definitions` tables
2. Implement ordered-step engine: `advanceStep()`, `blockStep()`, `getProgress()`
3. Stall detection query (generic: "find steps in_progress longer than timeout_hours")
4. Event emission interface (step.started, step.completed, step.stalled, workflow.completed)

### Phase 2: Integration Layer (~600 LOC)
5. Entry path router (multiple triggers → workflow creation)
6. Webhook callback handler (external events advance steps)
7. Progress API (current step, % complete, blockers, time-in-step)
8. Tenant-scoped RLS policies

### Phase 3: Consumer Migration (Coder-2, after v0.1.0 ships)
9. PPCRM data migration (fan-out boolean flags → step_state rows)
10. PPCRM code migration (replace onboardbot's inline logic with @milo/onboarding calls)
11. milo-ops migration (replace publisher-onboarding.ts with @milo/onboarding client)

### NOT in v0.1.0
- Step branching / conditional skip
- Parallel step execution
- Step rollback / undo
- Approval chains (use step.blocked + manual advance instead)
- Notification delivery (just emit events; consumers wire their own channels)

## 11. Open Questions for Mark

1. **Should workflow definitions be DB-stored or code-defined?** PPCRM's steps are hardcoded in onboardbot. For multi-vertical, definitions could live in DB (runtime-configurable) or in consumer code (deploy-time). DB is more flexible but harder to version. Recommend: code-defined in v0.1.0, DB-stored in v0.2.0 if needed.

2. **How should the 3 auto-completing steps (MSA signed, W9 signed, IO signed) work?** Currently MOP fires a webhook → signature-webhook Edge Function → updates boolean flag. In the extracted model, should @milo/onboarding expose a webhook endpoint, or should consumers call `advanceStep()` from their own webhook handlers? Recommend: consumers call `advanceStep()` — keeps @milo/onboarding transport-agnostic.

3. **Should @milo/onboarding own the stall-detection cron, or should consumers run their own?** PPCRM uses pg_cron. If @milo/onboarding provides a `findStalledSteps()` query, consumers can call it from their own cron/scheduler. Recommend: provide the query, not the cron — scheduling is infrastructure-specific.

4. **What's the relationship between milo-ops orchestration and @milo/onboarding?** Currently milo-ops is the "brain" that decides which path to take and proxies to PPCRM. Post-extraction, does milo-ops become a thin consumer of @milo/onboarding, or does it retain orchestration authority? Recommend: milo-ops remains the orchestrator, @milo/onboarding provides the state engine.

5. **Should the pre-entry scam check (blacklist gate) be modeled as "step 0" or stay external?** Currently it's a gate in milo-ops before `startOnboarding()` is called. Making it step 0 would unify the model but couples @milo/onboarding to @milo/blacklist. Recommend: keep external — it's a pre-condition, not a step.

## 12. Estimated LOC for v0.1.0

| Component | LOC |
|-----------|-----|
| Schema (3 tables + indexes + RLS) | 200 |
| Core engine (advance, block, skip, getProgress) | 500 |
| Step state management (status transitions, timestamps) | 300 |
| Stall detection queries | 150 |
| Event emission interface | 150 |
| Entry path routing | 200 |
| Webhook callback handler (generic) | 200 |
| Progress API | 150 |
| TypeScript types + exports | 100 |
| **Total** | **1,950** |

Range: **1,800–2,200 LOC** depending on how much validation/error handling the step transitions need.

---

## Appendix A: The 12 PPCRM Onboarding Steps

| # | Step Key | Trigger | Auto? | External Dep |
|---|----------|---------|-------|--------------|
| 1 | added_to_mop | milo-ops creates MOP entity | Semi | MOP API |
| 2 | msa_sent | onboardbot sends MSA | Auto | MOP doc gen |
| 3 | msa_signed | MOP webhook callback | Auto | MOP signing |
| 4 | w9_sent | onboardbot sends W9 | Auto | MOP doc gen |
| 5 | w9_signed | MOP webhook callback | Auto | MOP signing |
| 6 | creatives_submitted | Publisher uploads via form | Manual | Supabase Storage |
| 7 | creatives_approved | Operator reviews + approves | Manual | None |
| 8 | did_provisioned | Operator provisions in TrackDrive | Manual | TrackDrive |
| 9 | config_validated | System validates all config | Auto | Internal checks |
| 10 | io_sent | onboardbot sends IO | Auto | MOP doc gen |
| 11 | io_signed | MOP webhook callback | Auto | MOP signing |
| 12 | go_live | System flips publisher active | Auto | Campaign tables |

## Appendix B: milo-ops Entry Paths

| Path | Trigger | Pre-conditions | Outcome |
|------|---------|----------------|---------|
| **Pipeline entity** | Operator clicks "Start Onboarding" on vetted prospect | Passed vetting, has CRM record | Full 12-step flow |
| **Trusted shortcut** | Operator uses AddProspectModal "trusted" path | Skips vetting, direct onboard | Full 12-step flow (no scam check) |
| **Inbound MOP webhook** | MOP receives signed doc from unknown entity | Entity exists in MOP | Creates CRM record + starts flow |

All three paths converge at `publisher-onboarding.ts` → call PPCRM `startOnboarding()`.
