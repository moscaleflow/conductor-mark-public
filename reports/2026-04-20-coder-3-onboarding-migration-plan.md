---
directive: "@milo/onboarding migration script design"
lane: research
coder: Coder-3
started: 2026-04-20 ~08:00 MDT
completed: 2026-04-20 ~08:20 MDT
depends-on:
  - reports/2026-04-20-coder-3-onboarding-migration-audit.md (dcada44)
  - Counterparty UUID preservation confirmed (813bf45)
implements-for: Coder-1 (script author)
---

# @milo/onboarding Migration Script Plan — 2026-04-20

## Overview

| Parameter | Value |
|-----------|-------|
| Source | PPCRM frozen (jdzqkaxmnqbboqefjolf) |
| Target | Shared Supabase (tappyckcteqgryjniwjg) |
| Tenant | 'tlp' |
| Source rows | 8 (onboarding_checklists) |
| Target rows created | 129 (1 flow + 8 runs + 120 steps) |
| Delta risk | NONE (PPCRM fully frozen, no write window) |
| Counterparty resolution | DIRECT (partner_id = counterparty_id, UUIDs preserved) |
| Script language | TypeScript (match packages/crm/scripts/migrate-ppcrm-data.ts pattern) |
| Script location | packages/onboarding/scripts/migrate-ppcrm-data.ts |

---

## (a) Pre-Flight Checks

Run before any writes. All must pass or script aborts.

```typescript
async function preflight(target: SupabaseClient, source: SupabaseClient) {
  const checks: PreflightCheck[] = [];

  // 1. Target tables exist
  const { data: tables } = await target.rpc('pg_tables_check', {
    table_names: ['onboarding_flows', 'onboarding_runs', 'onboarding_steps']
  });
  // Or: SELECT tablename FROM pg_tables WHERE schemaname='public' AND tablename IN (...)
  checks.push({ name: 'target_tables_exist', pass: tables?.length === 3 });

  // 2. Source row count matches audit
  const { count } = await source
    .from('onboarding_checklists')
    .select('*', { count: 'exact', head: true });
  checks.push({ name: 'source_row_count', pass: count === 8, actual: count });

  // 3. All 8 partner_ids resolve in crm_counterparties
  const { data: checklists } = await source
    .from('onboarding_checklists')
    .select('partner_id');
  const partnerIds = checklists!.map(c => c.partner_id);
  
  const { count: resolvedCount } = await target
    .from('crm_counterparties')
    .select('*', { count: 'exact', head: true })
    .in('id', partnerIds)
    .eq('tenant_id', 'tlp');
  checks.push({ 
    name: 'counterparty_resolution', 
    pass: resolvedCount === partnerIds.length,
    actual: resolvedCount,
    expected: partnerIds.length
  });

  // 4. No existing TLP onboarding data (idempotency guard)
  const { count: existingRuns } = await target
    .from('onboarding_runs')
    .select('*', { count: 'exact', head: true })
    .eq('tenant_id', 'tlp');
  checks.push({ name: 'no_existing_data', pass: existingRuns === 0, actual: existingRuns });

  // 5. No existing TLP flow definition
  const { count: existingFlows } = await target
    .from('onboarding_flows')
    .select('*', { count: 'exact', head: true })
    .eq('tenant_id', 'tlp')
    .eq('name', 'ppc-publisher-onboarding');
  checks.push({ name: 'no_existing_flow', pass: existingFlows === 0, actual: existingFlows });

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

## (b) Flow Definition Seed

### The 15-Step PPC Flow

```typescript
const PPC_FLOW_STEPS: FlowStepDefinition[] = [
  {
    key: 'add_to_mop',
    label: 'Add to MOP',
    order: 1,
    required: true,
    auto_advance: true,
    timeout_days: 3,
    depends_on: [],
    integration: { type: 'webhook', url: 'mop', event: 'partner.created' },
    metadata: {}
  },
  {
    key: 'offer_accepted',
    label: 'Campaign Offer Accepted',
    order: 2,
    required: false,
    auto_advance: true,
    timeout_days: null,
    depends_on: ['add_to_mop'],
    integration: { type: 'webhook', url: 'mop', event: 'campaign.offer_accepted' },
    metadata: {}
  },
  {
    key: 'msa_sent',
    label: 'MSA Sent',
    order: 3,
    required: true,
    auto_advance: true,
    timeout_days: 3,
    depends_on: ['add_to_mop'],
    integration: { type: 'contract_signing', documentType: 'msa' },
    metadata: {}
  },
  {
    key: 'msa_signed',
    label: 'MSA Signed',
    order: 4,
    required: true,
    auto_advance: true,
    timeout_days: 7,
    depends_on: ['msa_sent'],
    integration: { type: 'contract_signing', documentType: 'msa' },
    metadata: {}
  },
  {
    key: 'w9_sent',
    label: 'W-9/W-8 Sent',
    order: 5,
    required: true,
    auto_advance: true,
    timeout_days: 3,
    depends_on: ['msa_signed'],
    integration: { type: 'contract_signing', documentType: 'w9' },
    metadata: { tax_doc_routing: true }
  },
  {
    key: 'w9_signed',
    label: 'W-9/W-8 Signed',
    order: 6,
    required: true,
    auto_advance: true,
    timeout_days: 7,
    depends_on: ['w9_sent'],
    integration: { type: 'contract_signing', documentType: 'w9' },
    metadata: {}
  },
  {
    key: 'creatives_submitted',
    label: 'Creatives Submitted',
    order: 7,
    required: true,
    auto_advance: false,
    timeout_days: 14,
    depends_on: ['w9_signed'],
    integration: { type: 'file_upload', bucket: 'partner-creatives' },
    metadata: {}
  },
  {
    key: 'creatives_approved',
    label: 'Creatives Approved',
    order: 8,
    required: true,
    auto_advance: false,
    timeout_days: 3,
    depends_on: ['creatives_submitted'],
    integration: { type: 'custom', handler: 'creatives_review' },
    metadata: {}
  },
  {
    key: 'rider_sent',
    label: 'Rider Sent',
    order: 9,
    required: false,
    auto_advance: true,
    timeout_days: 3,
    depends_on: ['creatives_approved'],
    integration: { type: 'contract_signing', documentType: 'rider' },
    metadata: {}
  },
  {
    key: 'rider_signed',
    label: 'Rider Signed',
    order: 10,
    required: false,
    auto_advance: true,
    timeout_days: 7,
    depends_on: ['rider_sent'],
    integration: { type: 'contract_signing', documentType: 'rider' },
    metadata: {}
  },
  {
    key: 'did_provisioned',
    label: 'DID Provisioned',
    order: 11,
    required: true,
    auto_advance: false,
    timeout_days: 7,
    depends_on: ['creatives_approved'],
    integration: { type: 'custom', handler: 'trackdrive_did' },
    metadata: {}
  },
  {
    key: 'config_validated',
    label: 'Config Validated',
    order: 12,
    required: true,
    auto_advance: true,
    timeout_days: 3,
    depends_on: ['did_provisioned'],
    integration: { type: 'custom', handler: 'config_validation' },
    metadata: {}
  },
  {
    key: 'io_sent',
    label: 'IO Sent',
    order: 13,
    required: true,
    auto_advance: true,
    timeout_days: 3,
    depends_on: ['config_validated'],
    integration: { type: 'contract_signing', documentType: 'io' },
    metadata: {}
  },
  {
    key: 'io_signed',
    label: 'IO Signed',
    order: 14,
    required: true,
    auto_advance: true,
    timeout_days: 7,
    depends_on: ['io_sent'],
    integration: { type: 'contract_signing', documentType: 'io' },
    metadata: {}
  },
  {
    key: 'go_live',
    label: 'Go Live',
    order: 15,
    required: true,
    auto_advance: true,
    timeout_days: 1,
    depends_on: ['io_signed'],
    integration: { type: 'crm_update', fields: ['status'] },
    metadata: {}
  }
];
```

### Seed Operation

```typescript
async function seedFlow(target: SupabaseClient): Promise<string> {
  const { data, error } = await target
    .from('onboarding_flows')
    .upsert({
      tenant_id: 'tlp',
      name: 'ppc-publisher-onboarding',
      description: 'TLP PPC publisher onboarding — 15-step linear flow',
      steps: PPC_FLOW_STEPS,
      is_active: true,
      version: 1,
      metadata: { migrated_from: 'ppcrm', migrated_at: new Date().toISOString() }
    }, { onConflict: 'tenant_id,name' })
    .select('id')
    .single();

  if (error) throw new Error(`Flow seed failed: ${error.message}`);
  console.log(`✓ Flow seeded: ${data.id}`);
  return data.id;
}
```

---

## (c) Runs Fan-Out

### Transform Function

```typescript
interface ChecklistRow {
  id: string;
  partner_id: string;
  partner_name: string | null;
  deal_id: string | null;
  tax_doc_type: string | null;
  days_to_onboard: number | null;
  creatives_reviewed_by: string | null;
  did_setup_by: string | null;
  config_validation_status: string | null;
  completed: boolean | null;
  archived_at: string | null;
  created_at: string;
  updated_at: string | null;
  // 15 boolean flags
  step_add_to_mop: boolean;
  step_offer_accepted: boolean;
  step_msa_sent: boolean;
  step_msa_signed: boolean;
  step_w9_sent: boolean;
  step_w9_signed: boolean;
  step_creatives_submitted: boolean;
  step_creatives_approved: boolean;
  step_rider_sent: boolean;
  step_rider_signed: boolean;
  step_did_provisioned: boolean;
  step_config_validated: boolean;
  step_io_sent: boolean;
  step_io_signed: boolean;
  step_go_live: boolean;
  // Timestamp columns
  go_live_at: string | null;
  added_to_mop_at: string | null;
  // ... etc
}

function transformToRun(c: ChecklistRow, flowId: string, flowSteps: FlowStepDefinition[]) {
  const status = c.archived_at ? 'cancelled'
    : c.step_go_live ? 'completed'
    : 'active';

  return {
    // id: generated by DB (gen_random_uuid)
    tenant_id: 'tlp',
    flow_id: flowId,
    flow_snapshot: flowSteps,  // deep copy frozen at migration time
    counterparty_id: c.partner_id,  // UUID preservation — direct mapping
    status,
    started_by: 'system',
    entry_path: 'pipeline',
    completed_at: c.step_go_live ? c.go_live_at : null,
    cancelled_at: c.archived_at,
    cancel_reason: c.archived_at ? 'archived' : null,
    metadata: {
      source_checklist_id: c.id,
      source_partner_name: c.partner_name,
      source_deal_id: c.deal_id,
      tax_doc_type: c.tax_doc_type,
      days_to_onboard: c.days_to_onboard,
      creatives_reviewed_by: c.creatives_reviewed_by,
      did_setup_by: c.did_setup_by,
      config_validation_status: c.config_validation_status,
      migrated_from: 'ppcrm',
      migrated_at: new Date().toISOString()
    },
    created_at: c.created_at,
    updated_at: c.updated_at || c.created_at
  };
}
```

### Insert Runs

```typescript
async function migrateRuns(
  target: SupabaseClient,
  checklists: ChecklistRow[],
  flowId: string,
  flowSteps: FlowStepDefinition[]
): Promise<Map<string, string>> {
  // Map: source checklist_id → new run_id
  const checklistToRun = new Map<string, string>();

  const runs = checklists.map(c => transformToRun(c, flowId, flowSteps));

  const { data, error } = await target
    .from('onboarding_runs')
    .insert(runs)
    .select('id, metadata');

  if (error) throw new Error(`Runs insert failed: ${error.message}`);

  for (const run of data!) {
    checklistToRun.set(run.metadata.source_checklist_id, run.id);
  }

  console.log(`✓ ${data!.length} runs migrated`);
  return checklistToRun;
}
```

---

## (d) Steps Fan-Out

### Step Flag → Column Mapping

```typescript
const STEP_COLUMNS: { key: string; order: number; flag: keyof ChecklistRow; timestamp: keyof ChecklistRow | null; required: boolean }[] = [
  { key: 'add_to_mop',          order: 1,  flag: 'step_add_to_mop',          timestamp: 'added_to_mop_at',     required: true },
  { key: 'offer_accepted',      order: 2,  flag: 'step_offer_accepted',      timestamp: 'offer_accepted_at',   required: false },
  { key: 'msa_sent',            order: 3,  flag: 'step_msa_sent',            timestamp: 'msa_sent_at',         required: true },
  { key: 'msa_signed',          order: 4,  flag: 'step_msa_signed',          timestamp: 'msa_signed_at',       required: true },
  { key: 'w9_sent',             order: 5,  flag: 'step_w9_sent',             timestamp: 'w9_sent_at',          required: true },
  { key: 'w9_signed',           order: 6,  flag: 'step_w9_signed',           timestamp: 'w9_signed_at',        required: true },
  { key: 'creatives_submitted', order: 7,  flag: 'step_creatives_submitted', timestamp: null,                  required: true },
  { key: 'creatives_approved',  order: 8,  flag: 'step_creatives_approved',  timestamp: 'creatives_approved_at', required: true },
  { key: 'rider_sent',          order: 9,  flag: 'step_rider_sent',          timestamp: null,                  required: false },
  { key: 'rider_signed',        order: 10, flag: 'step_rider_signed',        timestamp: null,                  required: false },
  { key: 'did_provisioned',     order: 11, flag: 'step_did_provisioned',     timestamp: 'did_provisioned_at',  required: true },
  { key: 'config_validated',    order: 12, flag: 'step_config_validated',    timestamp: 'config_validated_at', required: true },
  { key: 'io_sent',             order: 13, flag: 'step_io_sent',             timestamp: 'io_sent_at',          required: true },
  { key: 'io_signed',           order: 14, flag: 'step_io_signed',           timestamp: 'io_signed_at',        required: true },
  { key: 'go_live',             order: 15, flag: 'step_go_live',             timestamp: 'go_live_at',          required: true },
];
```

### Status Derivation Logic

```typescript
function deriveStepStatus(
  checklist: ChecklistRow,
  runStatus: string
): { key: string; order: number; status: string; started_at: string | null; completed_at: string | null }[] {
  const steps: ReturnType<typeof deriveStepStatus> = [];
  let lastCompletedOrder = 0;
  let foundActive = false;

  for (const col of STEP_COLUMNS) {
    const flagValue = checklist[col.flag] as boolean;
    const timestampValue = col.timestamp ? (checklist[col.timestamp] as string | null) : null;

    if (flagValue) {
      // Step completed
      steps.push({
        key: col.key,
        order: col.order,
        status: 'completed',
        started_at: timestampValue || checklist.updated_at,
        completed_at: timestampValue || checklist.updated_at
      });
      lastCompletedOrder = col.order;
    } else if (!foundActive && runStatus === 'active') {
      // First incomplete step on an active run = active step
      // But only for required steps (optional steps that weren't done get skipped)
      if (col.required || col.order === lastCompletedOrder + 1) {
        steps.push({
          key: col.key,
          order: col.order,
          status: 'active',
          started_at: checklist.updated_at,
          completed_at: null
        });
        foundActive = true;
      } else {
        // Non-required step that was bypassed
        steps.push({
          key: col.key,
          order: col.order,
          status: 'skipped',
          started_at: null,
          completed_at: null
        });
      }
    } else if (!flagValue && !col.required && lastCompletedOrder >= col.order) {
      // Non-required step where later steps are completed → skipped
      steps.push({
        key: col.key,
        order: col.order,
        status: 'skipped',
        started_at: null,
        completed_at: null
      });
    } else {
      // Future step, not yet reached
      steps.push({
        key: col.key,
        order: col.order,
        status: 'pending',
        started_at: null,
        completed_at: null
      });
    }
  }

  return steps;
}
```

### Insert Steps

```typescript
async function migrateSteps(
  target: SupabaseClient,
  checklists: ChecklistRow[],
  checklistToRun: Map<string, string>
): Promise<number> {
  const allSteps: any[] = [];

  for (const checklist of checklists) {
    const runId = checklistToRun.get(checklist.id)!;
    const runStatus = checklist.archived_at ? 'cancelled'
      : checklist.step_go_live ? 'completed'
      : 'active';

    const derivedSteps = deriveStepStatus(checklist, runStatus);

    for (const step of derivedSteps) {
      allSteps.push({
        tenant_id: 'tlp',
        run_id: runId,
        step_key: step.key,
        step_order: step.order,
        status: step.status,
        started_at: step.started_at,
        completed_at: step.completed_at,
        attempt_count: step.status === 'completed' ? 1 : 0,
        metadata: {},
        created_at: checklist.created_at
      });
    }
  }

  // 120 rows — single batch is fine
  const { error } = await target
    .from('onboarding_steps')
    .insert(allSteps);

  if (error) throw new Error(`Steps insert failed: ${error.message}`);
  console.log(`✓ ${allSteps.length} steps created`);
  return allSteps.length;
}
```

---

## (e) Idempotency

| Table | Natural Key | Conflict Strategy |
|-------|-------------|-------------------|
| onboarding_flows | UNIQUE(tenant_id, name) | `onConflict: 'tenant_id,name'` → DO NOTHING |
| onboarding_runs | metadata.source_checklist_id | Pre-check: if any runs with `metadata->>'source_checklist_id'` exist, abort |
| onboarding_steps | UNIQUE(run_id, step_key) | Insert will fail on duplicate → caught by pre-check |

### Re-Run Safety

```typescript
async function idempotencyCheck(target: SupabaseClient): Promise<boolean> {
  const { count } = await target
    .from('onboarding_runs')
    .select('*', { count: 'exact', head: true })
    .eq('tenant_id', 'tlp')
    .not('metadata->source_checklist_id', 'is', null);

  if (count && count > 0) {
    console.warn(`⚠ Migration already ran (${count} runs found). Skipping.`);
    return false;  // already migrated
  }
  return true;  // safe to proceed
}
```

---

## (f) Post-Migration Verification

```typescript
async function verify(target: SupabaseClient) {
  const checks: { name: string; expected: number; actual: number }[] = [];

  // 1. Flow exists
  const { count: flows } = await target
    .from('onboarding_flows')
    .select('*', { count: 'exact', head: true })
    .eq('tenant_id', 'tlp');
  checks.push({ name: 'flows', expected: 1, actual: flows! });

  // 2. Runs count
  const { count: runs } = await target
    .from('onboarding_runs')
    .select('*', { count: 'exact', head: true })
    .eq('tenant_id', 'tlp');
  checks.push({ name: 'runs', expected: 8, actual: runs! });

  // 3. Steps count
  const { count: steps } = await target
    .from('onboarding_steps')
    .select('*', { count: 'exact', head: true })
    .eq('tenant_id', 'tlp');
  checks.push({ name: 'steps', expected: 120, actual: steps! });

  // 4. No orphaned steps (all run_ids valid)
  const { count: orphans } = await target
    .from('onboarding_steps')
    .select('*', { count: 'exact', head: true })
    .eq('tenant_id', 'tlp')
    .not('run_id', 'in', `(SELECT id FROM onboarding_runs WHERE tenant_id='tlp')`);
  // Alternative: just check run_id FK integrity (DB constraint handles this)

  // 5. Every run has exactly 15 steps
  const { data: stepCounts } = await target
    .from('onboarding_steps')
    .select('run_id')
    .eq('tenant_id', 'tlp');
  const perRun = new Map<string, number>();
  for (const s of stepCounts!) {
    perRun.set(s.run_id, (perRun.get(s.run_id) || 0) + 1);
  }
  const allFifteen = [...perRun.values()].every(c => c === 15);
  checks.push({ name: 'steps_per_run', expected: 15, actual: allFifteen ? 15 : -1 });

  // 6. Status consistency
  const { data: statusCheck } = await target
    .from('onboarding_runs')
    .select('id, status, completed_at, cancelled_at')
    .eq('tenant_id', 'tlp');
  for (const run of statusCheck!) {
    if (run.status === 'completed' && !run.completed_at) {
      console.error(`❌ Run ${run.id} is completed but has no completed_at`);
    }
    if (run.status === 'cancelled' && !run.cancelled_at) {
      console.error(`❌ Run ${run.id} is cancelled but has no cancelled_at`);
    }
  }

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

## (g) Rollback Plan

```typescript
async function rollback(target: SupabaseClient) {
  console.log('Rolling back @milo/onboarding migration for tenant=tlp...');

  // Order matters: steps first (FK constraint), then runs, then flow
  const { error: e1 } = await target
    .from('onboarding_steps')
    .delete()
    .eq('tenant_id', 'tlp');
  if (e1) console.error('Steps delete error:', e1.message);

  const { error: e2 } = await target
    .from('onboarding_runs')
    .delete()
    .eq('tenant_id', 'tlp');
  if (e2) console.error('Runs delete error:', e2.message);

  const { error: e3 } = await target
    .from('onboarding_flows')
    .delete()
    .eq('tenant_id', 'tlp')
    .eq('name', 'ppc-publisher-onboarding');
  if (e3) console.error('Flow delete error:', e3.message);

  console.log('✓ Rollback complete');
}
```

**Safety:** Tenant-scoped DELETE only affects TLP data. No other tenants are affected. CASCADE on onboarding_steps (from run FK) would also handle this, but explicit per-table delete is preferred for audit clarity.

---

## Script Entrypoint

```typescript
async function main() {
  const source = createClient(PPCRM_URL, PPCRM_SERVICE_KEY);
  const target = createClient(SHARED_URL, SHARED_SERVICE_KEY);

  // Phase 1: Preflight
  await preflight(target, source);

  // Phase 2: Idempotency check
  const canProceed = await idempotencyCheck(target);
  if (!canProceed) return;

  // Phase 3: Read source data
  const { data: checklists } = await source
    .from('onboarding_checklists')
    .select('*')
    .order('created_at');
  console.log(`Read ${checklists!.length} checklists from PPCRM`);

  // Phase 4: Seed flow
  const flowId = await seedFlow(target);

  // Phase 5: Migrate runs
  const checklistToRun = await migrateRuns(target, checklists!, flowId, PPC_FLOW_STEPS);

  // Phase 6: Generate steps
  const stepCount = await migrateSteps(target, checklists!, checklistToRun);

  // Phase 7: Verify
  const verified = await verify(target);
  if (!verified) {
    console.error('Verification failed — consider rollback');
    process.exit(1);
  }

  // Summary
  console.log('\n=== Migration Complete ===');
  console.log(`Flow:  1 (${flowId})`);
  console.log(`Runs:  ${checklistToRun.size}`);
  console.log(`Steps: ${stepCount}`);
  console.log(`Total: ${1 + checklistToRun.size + stepCount} rows created`);
}
```

---

## Environment Variables Required

```bash
# Source (PPCRM frozen)
PPCRM_SUPABASE_URL=https://jdzqkaxmnqbboqefjolf.supabase.co
PPCRM_SERVICE_ROLE_KEY=<from: npx supabase projects api-keys --project-ref jdzqkaxmnqbboqefjolf>

# Target (shared)
SHARED_SUPABASE_URL=https://tappyckcteqgryjniwjg.supabase.co
SHARED_SERVICE_ROLE_KEY=<from: npx supabase projects api-keys --project-ref tappyckcteqgryjniwjg>
```

---

## Edge Cases

| Case | Handling |
|------|----------|
| Checklist with archived_at set | Run status = 'cancelled', all non-completed steps = 'pending' |
| Checklist fully completed (go_live=true) | Run status = 'completed', all 15 steps = 'completed' |
| Optional step skipped (rider_sent=false but later steps done) | Step status = 'skipped' |
| No timestamp for completed step | Use checklist.updated_at as fallback |
| partner_id not in crm_counterparties | Preflight FAILS — script aborts (should never happen given 601 > 8) |
| Script run twice | Idempotency check catches existing runs, exits cleanly |
