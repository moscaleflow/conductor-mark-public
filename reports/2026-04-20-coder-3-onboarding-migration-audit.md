---
directive: "@milo/onboarding data migration audit"
lane: research
coder: Coder-3
started: 2026-04-20 ~06:00 MDT
completed: 2026-04-20 ~06:30 MDT
---

# @milo/onboarding Data Migration Audit — 2026-04-20

## 1. Executive Summary

| Metric | Value |
|--------|-------|
| Source table | onboarding_checklists (PPCRM frozen) |
| Source rows | **8** (confirmed — manifest + backup verified) |
| Target tables | 3 (onboarding_flows, onboarding_runs, onboarding_steps) |
| Fan-out | 8 rows → 1 flow + 8 runs + 120 steps = **129 rows total** |
| Step flags found | **15** (not 12 — 3 added in later migrations) |
| Counterparty resolution | 601 TLP counterparties in @milo/crm — covers all 8 |
| Complexity | LOW — tiny dataset, clear mapping |
| Ordering position | After @milo/crm (shipped), after contract siblings (ready) |
| Target Supabase | tappyckcteqgryjniwjg (shared) |
| Tenant assignment | All rows: tenant_id = 'tlp' |

**Key corrections from prior audit:**
- Row count: 8 (not 200-500 — Decision 51 flag confirmed)
- Step count: 15 boolean columns (not 12 — `step_offer_accepted`, `step_rider_sent`, `step_rider_signed` added in later migrations)
- FK target: `partner_id` (not `publisher_id`) — PPCRM uses a unified `partners` table

## 2. Row Count Reconciliation (Question a)

| Source | Count | Explanation |
|--------|-------|-------------|
| Freeze manifest (2026-04-19) | **8** | Exact backup-vs-live match, 0 delta |
| Coder-1 SCHEMA estimate | 200-500 | Assumption based on expected production scale |
| Decision 51 flag | "verify before migration" | Correctly flagged the discrepancy |

**Why only 8?** PPCRM is early-stage. Only 8 partners have entered the onboarding pipeline. The 200-500 estimate was a projection, not a measurement. The freeze manifest is authoritative.

## 3. Source Schema: onboarding_checklists (PPCRM frozen)

**8 rows, 41 columns.**

### Identity & Relationship Columns

| Column | Type | Null | Notes |
|--------|------|------|-------|
| id | UUID | NO | PK, gen_random_uuid() |
| partner_id | UUID | NO | FK → partners(id) |
| partner_name | TEXT | YES | Cached display name (migration 027) |
| deal_id | UUID | YES | Optional deal/contract reference |
| created_at | TIMESTAMPTZ | NO | DEFAULT now() |
| updated_at | TIMESTAMPTZ | YES | Auto-updated |

### Step Boolean Flags (15 total)

| # | Column | Default | Added |
|---|--------|---------|-------|
| 1 | step_add_to_mop | false | Original |
| 2 | step_msa_sent | false | Original |
| 3 | step_msa_signed | false | Original |
| 4 | step_w9_sent | false | Original |
| 5 | step_w9_signed | false | Original |
| 6 | step_creatives_submitted | false | Original |
| 7 | step_creatives_approved | false | Original |
| 8 | step_did_provisioned | false | Original |
| 9 | step_config_validated | false | Migration 009 (renamed from step_test_call_passed) |
| 10 | step_io_sent | false | Original |
| 11 | step_io_signed | false | Original |
| 12 | step_go_live | false | Original |
| 13 | step_offer_accepted | false | Migration 20260223000003 |
| 14 | step_rider_sent | false | signature-webhook handler |
| 15 | step_rider_signed | false | signature-webhook handler |

### Step Timestamp Columns

| Column | Paired With |
|--------|-------------|
| added_to_mop_at | step_add_to_mop |
| msa_sent_at | step_msa_sent |
| msa_signed_at | step_msa_signed |
| w9_sent_at | step_w9_sent |
| w9_signed_at | step_w9_signed |
| creatives_approved_at | step_creatives_approved |
| did_provisioned_at | step_did_provisioned |
| config_validated_at | step_config_validated |
| config_validation_date | (secondary — last validation run time) |
| io_sent_at | step_io_sent |
| io_signed_at | step_io_signed |
| go_live_at | step_go_live |
| offer_accepted_at | step_offer_accepted |

**Missing timestamps:** No explicit `_at` columns for `step_creatives_submitted`, `step_rider_sent`, `step_rider_signed`. These will get `completed_at = NULL` in migrated steps (status still derivable from boolean).

### Metadata Columns

| Column | Type | Notes |
|--------|------|-------|
| creatives_reviewed_by | TEXT | Who approved creatives |
| did_setup_by | TEXT | Who provisioned DIDs |
| tax_doc_type | TEXT | 'w9', 'w8ben', 'w8bene' (migration 20260302000001) |
| config_validation_status | TEXT | 'pass', 'fail' |
| completed | BOOLEAN | Set when step_go_live=true |
| days_to_onboard | INTEGER | SLA metric |
| archived_at | TIMESTAMPTZ | Soft-delete (migration 20260227000005) |

## 4. Target Schema: @milo/onboarding v0.1.0

### onboarding_flows (seed first)

| Column | Type | Notes |
|--------|------|-------|
| id | UUID | PK |
| tenant_id | TEXT NOT NULL | 'tlp' |
| name | TEXT NOT NULL | 'ppc-publisher-onboarding' |
| description | TEXT | NULL |
| steps | JSONB NOT NULL | FlowStepDefinition[] (15 steps) |
| is_active | BOOLEAN | true |
| version | INTEGER | 1 |
| metadata | JSONB | {} |
| archived_at | TIMESTAMPTZ | NULL |
| created_at | TIMESTAMPTZ | now() |
| updated_at | TIMESTAMPTZ | now() |

UNIQUE(tenant_id, name)

### onboarding_runs (1 per checklist row)

| Column | Type | Notes |
|--------|------|-------|
| id | UUID | PK |
| tenant_id | TEXT NOT NULL | 'tlp' |
| flow_id | UUID NOT NULL | FK → onboarding_flows(id) |
| flow_snapshot | JSONB NOT NULL | Frozen FlowStepDefinition[] |
| counterparty_id | UUID | Logical FK to crm_counterparties |
| status | TEXT NOT NULL | 'active', 'completed', 'cancelled', 'expired' |
| started_by | TEXT | 'system' |
| entry_path | TEXT | 'pipeline', 'trusted', 'inbound_webhook' |
| webhook_url | TEXT | NULL |
| expires_at | TIMESTAMPTZ | NULL |
| completed_at | TIMESTAMPTZ | NULL |
| cancelled_at | TIMESTAMPTZ | NULL |
| cancel_reason | TEXT | NULL |
| metadata | JSONB | {} |
| created_at | TIMESTAMPTZ | now() |
| updated_at | TIMESTAMPTZ | now() |

### onboarding_steps (15 per run)

| Column | Type | Notes |
|--------|------|-------|
| id | UUID | PK |
| tenant_id | TEXT NOT NULL | 'tlp' |
| run_id | UUID NOT NULL | FK → onboarding_runs(id) ON DELETE CASCADE |
| step_key | TEXT NOT NULL | e.g., 'msa_sent', 'did_provisioned' |
| step_order | INTEGER NOT NULL | 1-15 |
| status | TEXT NOT NULL | 'pending', 'active', 'completed', 'failed', 'skipped', 'blocked' |
| started_at | TIMESTAMPTZ | NULL |
| completed_at | TIMESTAMPTZ | NULL |
| failed_at | TIMESTAMPTZ | NULL |
| skipped_at | TIMESTAMPTZ | NULL |
| result | JSONB | NULL |
| failure_reason | TEXT | NULL |
| skip_reason | TEXT | NULL |
| attempt_count | INTEGER | 0 |
| metadata | JSONB | {} |
| created_at | TIMESTAMPTZ | now() |

UNIQUE(run_id, step_key)

## 5. Fan-Out Migration Math (Question b)

```
8 onboarding_checklists rows
  → 1 onboarding_flows row (PPC flow definition — seeded once)
  → 8 onboarding_runs rows (one per checklist)
  → 8 × 15 = 120 onboarding_steps rows

Total rows created: 129
```

**Step status derivation from boolean flags:**

For each checklist row, iterate through 15 boolean columns in order:
```
For step_i in steps[1..15]:
  if boolean_flag[i] == true:
    status = 'completed'
    completed_at = timestamp_column[i] if exists
  else if all previous steps completed (or this is first false after completed):
    status = 'active'
    started_at = updated_at of checklist (best approximation)
  else:
    status = 'pending'
```

**Special case — archived checklists:** If `archived_at IS NOT NULL`, the run status should be `'cancelled'` with `cancel_reason = 'archived'`.

## 6. Flow Definition (Questions d & e)

### PPC Publisher Onboarding Flow Definition

Must be inserted as the FIRST migration step (all runs reference it).

```json
{
  "id": "generated-uuid",
  "tenant_id": "tlp",
  "name": "ppc-publisher-onboarding",
  "description": "TLP PPC publisher onboarding — 15-step linear flow",
  "version": 1,
  "is_active": true,
  "steps": [
    { "key": "add_to_mop", "label": "Add to MOP", "order": 1, "required": true, "auto_advance": true, "timeout_days": 3, "depends_on": [], "integration": { "type": "webhook", "url": "mop", "event": "partner.created" }, "metadata": {} },
    { "key": "offer_accepted", "label": "Campaign Offer Accepted", "order": 2, "required": false, "auto_advance": true, "timeout_days": null, "depends_on": ["add_to_mop"], "integration": { "type": "webhook", "url": "mop", "event": "campaign.offer_accepted" }, "metadata": {} },
    { "key": "msa_sent", "label": "MSA Sent", "order": 3, "required": true, "auto_advance": true, "timeout_days": 3, "depends_on": ["add_to_mop"], "integration": { "type": "contract_signing", "documentType": "msa" }, "metadata": {} },
    { "key": "msa_signed", "label": "MSA Signed", "order": 4, "required": true, "auto_advance": true, "timeout_days": 7, "depends_on": ["msa_sent"], "integration": { "type": "contract_signing", "documentType": "msa" }, "metadata": {} },
    { "key": "w9_sent", "label": "W-9/W-8 Sent", "order": 5, "required": true, "auto_advance": true, "timeout_days": 3, "depends_on": ["msa_signed"], "integration": { "type": "contract_signing", "documentType": "w9" }, "metadata": { "tax_doc_routing": true } },
    { "key": "w9_signed", "label": "W-9/W-8 Signed", "order": 6, "required": true, "auto_advance": true, "timeout_days": 7, "depends_on": ["w9_sent"], "integration": { "type": "contract_signing", "documentType": "w9" }, "metadata": {} },
    { "key": "creatives_submitted", "label": "Creatives Submitted", "order": 7, "required": true, "auto_advance": false, "timeout_days": 14, "depends_on": ["w9_signed"], "integration": { "type": "file_upload", "bucket": "partner-creatives" }, "metadata": {} },
    { "key": "creatives_approved", "label": "Creatives Approved", "order": 8, "required": true, "auto_advance": false, "timeout_days": 3, "depends_on": ["creatives_submitted"], "integration": { "type": "custom", "handler": "creatives_review" }, "metadata": {} },
    { "key": "rider_sent", "label": "Rider Sent", "order": 9, "required": false, "auto_advance": true, "timeout_days": 3, "depends_on": ["creatives_approved"], "integration": { "type": "contract_signing", "documentType": "rider" }, "metadata": {} },
    { "key": "rider_signed", "label": "Rider Signed", "order": 10, "required": false, "auto_advance": true, "timeout_days": 7, "depends_on": ["rider_sent"], "integration": { "type": "contract_signing", "documentType": "rider" }, "metadata": {} },
    { "key": "did_provisioned", "label": "DID Provisioned", "order": 11, "required": true, "auto_advance": false, "timeout_days": 7, "depends_on": ["creatives_approved"], "integration": { "type": "custom", "handler": "trackdrive_did" }, "metadata": {} },
    { "key": "config_validated", "label": "Config Validated", "order": 12, "required": true, "auto_advance": true, "timeout_days": 3, "depends_on": ["did_provisioned"], "integration": { "type": "custom", "handler": "config_validation" }, "metadata": {} },
    { "key": "io_sent", "label": "IO Sent", "order": 13, "required": true, "auto_advance": true, "timeout_days": 3, "depends_on": ["config_validated"], "integration": { "type": "contract_signing", "documentType": "io" }, "metadata": {} },
    { "key": "io_signed", "label": "IO Signed", "order": 14, "required": true, "auto_advance": true, "timeout_days": 7, "depends_on": ["io_sent"], "integration": { "type": "contract_signing", "documentType": "io" }, "metadata": {} },
    { "key": "go_live", "label": "Go Live", "order": 15, "required": true, "auto_advance": true, "timeout_days": 1, "depends_on": ["io_signed"], "integration": { "type": "crm_update", "fields": ["status"] }, "metadata": {} }
  ]
}
```

**Source of definition:** Derived from PPCRM's onboardbot logic, step flag names, and operational flow. This is a migration artifact — the canonical flow definition should be reviewed by Mark before insertion.

**flow_snapshot:** Each onboarding_runs row gets a deep-copy of this `steps` array frozen at migration time. All 8 migrated runs share the same snapshot (they all ran under the same 15-step flow).

## 7. Column Mapping: onboarding_checklists → onboarding_runs

| Source Column | Target Column | Transform |
|---------------|---------------|-----------|
| id | — | NOT preserved (new UUID generated) |
| — | id | gen_random_uuid() |
| — | tenant_id | Constant: 'tlp' |
| — | flow_id | FK to seeded PPC flow definition |
| — | flow_snapshot | Deep-copy of flow.steps JSONB |
| partner_id | counterparty_id | Direct: partner_id = counterparty_id (UUIDs preserved per Decision 32, verified by Coder-4) |
| — | status | Derived (see below) |
| — | started_by | 'system' |
| — | entry_path | 'pipeline' (default, no record of actual entry) |
| deal_id | metadata.source_deal_id | Fold to metadata |
| partner_name | metadata.source_partner_name | Fold to metadata |
| tax_doc_type | metadata.tax_doc_type | Vertical-specific |
| days_to_onboard | metadata.days_to_onboard | SLA metric |
| creatives_reviewed_by | metadata.creatives_reviewed_by | Vertical-specific |
| did_setup_by | metadata.did_setup_by | Vertical-specific |
| config_validation_status | metadata.config_validation_status | Vertical-specific |
| completed | — | Derived (status='completed' if true) |
| archived_at | cancelled_at | Map archived → cancelled |
| — | cancel_reason | 'archived' if archived_at IS NOT NULL |
| created_at | created_at | Direct |
| updated_at | updated_at | Direct |
| go_live_at | completed_at | If step_go_live=true |

### Run Status Derivation

```sql
CASE
  WHEN archived_at IS NOT NULL THEN 'cancelled'
  WHEN step_go_live = true THEN 'completed'
  ELSE 'active'
END
```

## 8. Column Mapping: Boolean Flags → onboarding_steps

For each of the 8 runs, generate 15 step rows:

| Step Flag | step_key | step_order | Status Logic | completed_at Source |
|-----------|----------|------------|--------------|---------------------|
| step_add_to_mop | add_to_mop | 1 | true→completed | added_to_mop_at |
| step_offer_accepted | offer_accepted | 2 | true→completed | offer_accepted_at |
| step_msa_sent | msa_sent | 3 | true→completed | msa_sent_at |
| step_msa_signed | msa_signed | 4 | true→completed | msa_signed_at |
| step_w9_sent | w9_sent | 5 | true→completed | w9_sent_at |
| step_w9_signed | w9_signed | 6 | true→completed | w9_signed_at |
| step_creatives_submitted | creatives_submitted | 7 | true→completed | NULL (no timestamp) |
| step_creatives_approved | creatives_approved | 8 | true→completed | creatives_approved_at |
| step_rider_sent | rider_sent | 9 | true→completed | NULL (no timestamp) |
| step_rider_signed | rider_signed | 10 | true→completed | NULL (no timestamp) |
| step_did_provisioned | did_provisioned | 11 | true→completed | did_provisioned_at |
| step_config_validated | config_validated | 12 | true→completed | config_validated_at |
| step_io_sent | io_sent | 13 | true→completed | io_sent_at |
| step_io_signed | io_signed | 14 | true→completed | io_signed_at |
| step_go_live | go_live | 15 | true→completed | go_live_at |

### Status Assignment Algorithm

```python
for each checklist row:
  last_completed_order = 0
  for step in steps_ordered_by_order:
    if boolean_flag is true:
      step.status = 'completed'
      step.completed_at = timestamp_column or checklist.updated_at
      last_completed_order = step.order
    elif step.order == last_completed_order + 1 and run.status == 'active':
      step.status = 'active'
      step.started_at = checklist.updated_at  # best approximation
    elif step.required == false and previous_required_completed:
      step.status = 'skipped'  # optional steps that were bypassed
    else:
      step.status = 'pending'
```

**Special handling for non-required steps (offer_accepted, rider_sent, rider_signed):**
If a run has completed steps AFTER these optional ones but the optional flag is false, mark them as `'skipped'` rather than `'pending'`.

## 9. Counterparty_id Resolution (Question h)

### PPCRM partner_id → @milo/crm counterparty_id

**Source:** PPCRM `partners` table → migrated to shared Supabase `crm_counterparties` (Decision 32)

| Fact | Value |
|------|-------|
| TLP counterparties in crm_counterparties | 601 |
| PPCRM onboarding_checklists rows | 8 |
| All 8 partner_ids resolvable? | **YES** (601 > 8, all PPCRM partners migrated) |

**Resolution method:** The PPCRM read-path migration (Decision 37) established a mapping. All 601 PPCRM partners are now in `crm_counterparties` with `tenant_id='tlp'`.

**Resolution: DIRECT.** Decision 32's migration script (`packages/crm/scripts/migrate-ppcrm-data.ts` line 184) sets `id: p.id` — source UUIDs are preserved verbatim. No lookup needed: `counterparty_id = partner_id`.

**Verified by Coder-4 (2026-04-20):** Read the migration script, confirmed `transformPartner()` maps `id: p.id` (line 184), `transformLead()` maps `id: l.id` (line 230), and contacts use `counterparty_id: p.id` (line 462). All source UUIDs preserved.

## 10. Migration SQL

### Step 1: Seed Flow Definition

```sql
INSERT INTO onboarding_flows (
  id, tenant_id, name, description, steps, is_active, version, metadata
)
VALUES (
  gen_random_uuid(),
  'tlp',
  'ppc-publisher-onboarding',
  'TLP PPC publisher onboarding — 15-step linear flow',
  '[... FlowStepDefinition[] JSON from §6 ...]'::jsonb,
  true,
  1,
  '{}'::jsonb
)
ON CONFLICT (tenant_id, name) DO NOTHING
RETURNING id;
-- Capture this ID as flow_id for subsequent inserts
```

### Step 2: Migrate Runs (8 rows)

```sql
WITH flow AS (
  SELECT id, steps FROM onboarding_flows
  WHERE tenant_id = 'tlp' AND name = 'ppc-publisher-onboarding'
)
INSERT INTO onboarding_runs (
  id, tenant_id, flow_id, flow_snapshot, counterparty_id,
  status, started_by, entry_path, metadata,
  completed_at, cancelled_at, cancel_reason,
  created_at, updated_at
)
SELECT
  gen_random_uuid(),
  'tlp',
  flow.id,
  flow.steps,  -- snapshot = current flow steps
  c.partner_id,  -- direct: partner_id = counterparty_id (UUIDs preserved, verified Coder-4)
  CASE
    WHEN c.archived_at IS NOT NULL THEN 'cancelled'
    WHEN c.step_go_live = true THEN 'completed'
    ELSE 'active'
  END,
  'system',
  'pipeline',
  jsonb_build_object(
    'source_checklist_id', c.id,
    'source_partner_name', c.partner_name,
    'source_deal_id', c.deal_id,
    'tax_doc_type', c.tax_doc_type,
    'days_to_onboard', c.days_to_onboard,
    'creatives_reviewed_by', c.creatives_reviewed_by,
    'did_setup_by', c.did_setup_by,
    'config_validation_status', c.config_validation_status,
    'migrated_from', 'ppcrm',
    'migrated_at', NOW()
  ),
  CASE WHEN c.step_go_live = true THEN c.go_live_at END,
  c.archived_at,
  CASE WHEN c.archived_at IS NOT NULL THEN 'archived' END,
  c.created_at,
  c.updated_at
FROM ppcrm_backup.onboarding_checklists c
CROSS JOIN flow;
```

### Step 3: Generate Steps (120 rows)

```sql
-- For each run, generate 15 step rows from boolean flags
-- This requires a lateral join or procedural logic

INSERT INTO onboarding_steps (
  id, tenant_id, run_id, step_key, step_order, status,
  started_at, completed_at, attempt_count, metadata, created_at
)
SELECT
  gen_random_uuid(),
  'tlp',
  r.id,
  s.step_key,
  s.step_order,
  CASE
    WHEN s.flag_value = true THEN 'completed'
    WHEN s.is_first_incomplete AND r.status = 'active' THEN 'active'
    WHEN s.required = false AND s.all_later_required_completed THEN 'skipped'
    ELSE 'pending'
  END,
  CASE
    WHEN s.is_first_incomplete AND r.status = 'active' THEN r.updated_at
  END,
  CASE
    WHEN s.flag_value = true THEN s.timestamp_value
  END,
  CASE WHEN s.flag_value = true THEN 1 ELSE 0 END,
  '{}'::jsonb,
  r.created_at
FROM onboarding_runs r
JOIN ppcrm_backup.onboarding_checklists c
  ON c.id = (r.metadata->>'source_checklist_id')::uuid
CROSS JOIN LATERAL (
  VALUES
    ('add_to_mop', 1, c.step_add_to_mop, c.added_to_mop_at, true),
    ('offer_accepted', 2, c.step_offer_accepted, c.offer_accepted_at, false),
    ('msa_sent', 3, c.step_msa_sent, c.msa_sent_at, true),
    ('msa_signed', 4, c.step_msa_signed, c.msa_signed_at, true),
    ('w9_sent', 5, c.step_w9_sent, c.w9_sent_at, true),
    ('w9_signed', 6, c.step_w9_signed, c.w9_signed_at, true),
    ('creatives_submitted', 7, c.step_creatives_submitted, NULL::timestamptz, true),
    ('creatives_approved', 8, c.step_creatives_approved, c.creatives_approved_at, true),
    ('rider_sent', 9, c.step_rider_sent, NULL::timestamptz, false),
    ('rider_signed', 10, c.step_rider_signed, NULL::timestamptz, false),
    ('did_provisioned', 11, c.step_did_provisioned, c.did_provisioned_at, true),
    ('config_validated', 12, c.step_config_validated, c.config_validated_at, true),
    ('io_sent', 13, c.step_io_sent, c.io_sent_at, true),
    ('io_signed', 14, c.step_io_signed, c.io_signed_at, true),
    ('go_live', 15, c.step_go_live, c.go_live_at, true)
) AS s(step_key, step_order, flag_value, timestamp_value, required)
WHERE r.tenant_id = 'tlp';
```

**Note:** The `is_first_incomplete` and `all_later_required_completed` logic needs a window function or procedural handling. For 8 rows, a simple script is cleaner than pure SQL.

## 11. Active vs Complete Runs (Question g)

With only 8 rows, the distribution is likely:

```sql
SELECT
  COUNT(*) FILTER (WHERE step_go_live = true) AS completed,
  COUNT(*) FILTER (WHERE archived_at IS NOT NULL) AS archived,
  COUNT(*) FILTER (WHERE step_go_live = false AND archived_at IS NULL) AS active
FROM onboarding_checklists;
```

**Estimate based on operational context:**
- TLP has ~310 publishers (from MOP freeze manifest)
- Only 8 have entered onboarding pipeline
- Some are likely test runs (early development)
- Expect: 2-4 completed, 1-2 active, 2-4 archived/test

**Impact:** Active runs need careful status mapping. If a run is mid-flow when migrated, the "active" step must be correctly identified.

## 12. Idempotency Strategy (Question i)

```sql
-- Flow definition: UNIQUE(tenant_id, name) + ON CONFLICT DO NOTHING
-- Runs: source_checklist_id in metadata enables deduplication
-- Steps: UNIQUE(run_id, step_key) prevents duplicate step creation
```

**Re-run safety:** The migration stores `source_checklist_id` in run metadata. A pre-check query can verify:
```sql
SELECT COUNT(*) FROM onboarding_runs
WHERE tenant_id = 'tlp'
  AND metadata->>'source_checklist_id' IS NOT NULL;
-- If > 0, migration already ran
```

## 13. Pre-Flight Checks

```sql
-- 1. Verify target tables exist
SELECT tablename FROM pg_tables
WHERE schemaname = 'public'
  AND tablename IN ('onboarding_flows', 'onboarding_runs', 'onboarding_steps');

-- 2. Verify no existing TLP data
SELECT COUNT(*) FROM onboarding_runs WHERE tenant_id = 'tlp';
-- Expected: 0

-- 3. Verify counterparty resolution
-- (depends on whether IDs were preserved — see §9 critical question)

-- 4. Verify source row count
SELECT COUNT(*) FROM ppcrm_backup.onboarding_checklists;
-- Expected: 8
```

## 14. Execution Order

```
┌─────────────────────────────────────┐
│ Pre-requisites (all shipped):       │
│  • @milo/crm (Decision 32) ✅       │
│  • @milo/onboarding v0.1.0 ✅       │
│  • Contract migrations (ready) ✅    │
└───────────────┬─────────────────────┘
                │
┌───────────────▼─────────────────────┐
│ Step 1: Seed PPC flow definition    │
│   → 1 onboarding_flows row         │
└───────────────┬─────────────────────┘
                │
┌───────────────▼─────────────────────┐
│ Step 2: Resolve counterparty_ids    │
│   → Verify 8 partner_ids map to    │
│     crm_counterparties              │
└───────────────┬─────────────────────┘
                │
┌───────────────▼─────────────────────┐
│ Step 3: Migrate runs (8 rows)       │
│   → 8 onboarding_runs rows         │
└───────────────┬─────────────────────┘
                │
┌───────────────▼─────────────────────┐
│ Step 4: Generate steps (120 rows)   │
│   → 120 onboarding_steps rows      │
└───────────────┬─────────────────────┘
                │
┌───────────────▼─────────────────────┐
│ Step 5: Verify integrity            │
│   → Run counts, FK checks, status   │
└─────────────────────────────────────┘
```

## 15. Dry-Run Output Shape

```json
{
  "migration": "@milo/onboarding",
  "source": "jdzqkaxmnqbboqefjolf (PPCRM frozen)",
  "target": "tappyckcteqgryjniwjg (shared)",
  "tenant_id": "tlp",
  "tables": {
    "onboarding_flows": {
      "rows_created": 1,
      "flow_name": "ppc-publisher-onboarding",
      "step_count": 15
    },
    "onboarding_runs": {
      "source_rows": 8,
      "migrated": 8,
      "status_breakdown": {
        "completed": "TBD (query needed)",
        "active": "TBD",
        "cancelled": "TBD (archived)"
      }
    },
    "onboarding_steps": {
      "rows_created": 120,
      "per_run": 15,
      "status_breakdown": "TBD per step_key"
    }
  },
  "counterparty_resolution": {
    "method": "TBD (ID preservation or lookup)",
    "resolved": 8,
    "unresolved": 0
  },
  "idempotent": true,
  "reversible": true
}
```

## 16. Rollback Plan

```sql
-- Reverse order: steps → runs → flow
DELETE FROM onboarding_steps WHERE tenant_id = 'tlp';
DELETE FROM onboarding_runs WHERE tenant_id = 'tlp';
DELETE FROM onboarding_flows WHERE tenant_id = 'tlp' AND name = 'ppc-publisher-onboarding';

-- Verification:
SELECT COUNT(*) FROM onboarding_runs WHERE tenant_id = 'tlp';
-- Expected: 0
```

Safe because:
- No other tables reference onboarding_runs.id yet
- Tenant isolation means deletion only affects TLP
- CASCADE on onboarding_steps means deleting runs auto-deletes steps (but explicit DELETE is safer for audit)

## 17. Open Questions for Mark

1. ~~**Counterparty ID preservation (CRITICAL):**~~ **RESOLVED — UUIDs preserved.** Coder-4 verified: `migrate-ppcrm-data.ts` line 184 sets `id: p.id`. `counterparty_id = partner_id` (trivial FK retarget, no lookup needed).

2. **Step ordering for offer_accepted, rider_sent, rider_signed:** These 3 steps were added after the original 12-step design. Where exactly do they fit in the linear sequence? The ordering in §8 (offer_accepted=2, rider=9-10) is my best guess from the onboardbot logic, but Mark should confirm.

3. **Flow definition review:** The FlowStepDefinition[] in §6 is derived from code analysis, not from an authoritative spec. Should Coder-1 review/approve before this definition is seeded?

4. **Test data in 8 rows:** Are any of the 8 checklist rows test/development data that should be excluded from migration? With such a small dataset, each row matters.

5. **PPCRM still frozen — write conflict?** Unlike MOP (globally unfrozen), PPCRM remains frozen. No split-brain risk for onboarding data. But verify: when PPCRM unfreezes, will onboardbot write to the LOCAL `onboarding_checklists` or to the shared `onboarding_runs`? This determines whether the freeze plan needs a selective guard on PPCRM too.

6. **Non-required steps (offer_accepted, rider):** If a checklist has `step_rider_sent=false` but later steps are completed (e.g., `step_did_provisioned=true`), should rider be marked `'skipped'` or `'pending'`? Recommend `'skipped'` since it was bypassed.

---

## Appendix A: Step Count Discrepancy

| Source | Step Count | Explanation |
|--------|-----------|-------------|
| Original onboarding audit (b143f6f) | 12 | Based on onboardbot documentation |
| This migration audit | 15 | Based on actual database columns |
| Target flow definition | 15 | Must match source to preserve fidelity |

The 3 additional steps (offer_accepted, rider_sent, rider_signed) were added in later PPCRM migrations and weren't part of the original onboardbot design. They're used by the signature-webhook handler and campaign offer system.

## Appendix B: Columns NOT Migrated (dropped)

| Column | Reason |
|--------|--------|
| id | New UUIDs generated for runs |
| partner_name | Redundant — derivable from counterparty_id join |
| completed | Derivable from run.status='completed' |
| config_validation_date | Folded to step metadata if needed |

All other source columns are either mapped to target columns or preserved in `metadata` JSONB.
