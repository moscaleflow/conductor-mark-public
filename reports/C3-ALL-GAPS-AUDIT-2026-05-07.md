# C3 Build Specs: 6 Stress-Test Gaps

**Auditor:** Coder-3 (Research Lane)
**Date:** 2026-05-07
**Repo:** milo-for-ppc (main)
**Scope:** Read-only characterization + build specs for 6 identified gaps

---

## Gap 1: Alert action_labels Too Generic

### Current Code Locations

| File | Lines | What it does |
|------|-------|-------------|
| `src/app/api/alerts/generate/route.ts` | L117 | **Default fallback**: `action_label ?? (type === 'critical' ? 'Investigate' : type === 'warning' ? 'Review' : 'View')` |
| `src/app/api/alerts/generate/route.ts` | L879, L936, L967, L997, L1055, L1122, L1194, L1229, L1278, L1340, L1385, L1436, L1486, L1526, L1573, L1625, L1665, L1739, L1796, L1871, L1926 | Per-alert-type labels -- some specific ("Check TrackDrive", "Generate Invoices", "Review Bids"), many generic ("Review", "Investigate") |
| `src/app/api/cron/evaluate/route.ts` | L341 | `action_label: 'Review Call'` (bid floor violation -- GOOD, specific) |
| `src/app/api/cron/evaluate/route.ts` | L745 | `action_label: 'Review publisher targeting'` (ping rejection -- GOOD) |
| `src/app/api/cron/evaluate/route.ts` | L878 | `action_label: 'Contact publisher'` (idle publisher -- GOOD) |
| `src/app/api/cron/evaluate/route.ts` | L1005 | `action_label: action` (invoice overdue -- computed: "Chase buyer payment" or "Send publisher payment" -- GOOD) |
| `src/app/api/cron/evaluate/route.ts` | L1175 | `action_label: 'Review for blacklist'` (repeat caller -- GOOD) |
| `src/lib/knowledge-engine.ts` | L811 | `action_label: anomaly.recommended_action` (LLM-generated -- variable quality) |
| `src/components/BriefingPanel.tsx` | L1535 | Renders `item.action_label` as a button/chip label |

### Schema State

`alerts` table columns relevant to action labels:
- `action_type TEXT` -- machine-readable action category (e.g., `bid_floor_violation`, `knowledge_anomaly`, `ping_rejection_high`, `idle_publisher`, `invoice_overdue`, `repeat_caller`)
- `action_label TEXT` -- human-readable button text shown in UI
- `action_data JSONB` -- structured data for the action handler

### Current action_label Inventory

| action_type | Current action_label | Specificity |
|-------------|---------------------|-------------|
| `bid_floor_violation` | "Review Call" | Good |
| `ping_rejection_high` | "Review publisher targeting" | Good |
| `idle_publisher` | "Contact publisher" | Good |
| `invoice_overdue` | "Chase buyer payment" / "Send publisher payment" | Good |
| `repeat_caller` | "Review for blacklist" | Good |
| `knowledge_anomaly` | LLM-generated `recommended_action` | Variable |
| Publisher quality alerts | "Review" | **GENERIC** |
| Buyer health alerts | "Review" | **GENERIC** |
| RTB bid alerts | "Review Bids" | Adequate |
| Call volume alerts | "Investigate" | **GENERIC** |
| Margin alerts | "Investigate" | **GENERIC** |
| Stale relationship | "Send Follow-Up" | Good |
| Buyer line not answering | "Investigate" | **GENERIC** |
| Publisher capacity | "Review" | **GENERIC** |
| Publisher grade dropped | "Review" | **GENERIC** |
| Payment overdue | "Review" | **GENERIC** |
| (default fallback) | "Investigate" / "Review" / "View" | **GENERIC** |

### Build Spec

**File to modify:** `src/app/api/alerts/generate/route.ts`

Replace the default fallback at L117 and all generic "Review" / "Investigate" labels with type-specific labels:

```
action_type                  -> New action_label
----------------------------------------------
publisher_quality_flag       -> "Review QC Flags"
publisher_grade_dropped      -> "Check Publisher Grade"
publisher_capacity           -> "Review Publisher Capacity"  
buyer_health_declining       -> "Review Buyer Health"
buyer_line_not_answering     -> "Check Buyer Line"
call_volume_drop             -> "Review Call Volume"
margin_below_threshold       -> "Review Margin"
buyer_payment_overdue        -> "Chase Payment"
ping_waste                   -> "Review Ping Costs"
ping_cost_exceeds_threshold  -> "Review Ping Economics"
getting_outbid               -> "Adjust Bid Strategy"
rtb_bids_declining           -> "Review RTB Bids"
rtb_bid_below_floor          -> "Adjust Bid Floor"
```

Also update the default fallback at L117 to be smarter:
```typescript
action_label: alert.action_label 
  ?? deriveActionLabel(alert.type, alert.entity_type, alert.title)
```

New helper function `deriveActionLabel(type, entityType, title)` with a lookup map keyed on title prefix patterns.

**Estimated complexity:** Small (1-2 hours). Single file change, mostly string mapping.

---

## Gap 2: No Auto action_item Creation from Alerts

### Current Code Locations

| File | Lines | What it does |
|------|-------|-------------|
| `src/app/api/action-items/from-alert/route.ts` | L1-45 | **Manual** endpoint: creates an action_item from alert text. Called by operators clicking a button. Does NOT auto-fire. |
| `src/app/api/action-items/create/route.ts` | L1-100 | Generic task creation with dedup on source+source_id. |
| `src/app/api/action-items/route.ts` | L98-180 | Full-featured POST with entity fields, recurrence, metadata. |
| `src/app/api/cron/evaluate/route.ts` | (none) | Alert creation code does NOT create action_items. |
| `src/app/api/alerts/generate/route.ts` | (none) | Alert creation code does NOT create action_items. |
| `src/lib/knowledge-engine.ts` | L769-830 | `processAnomalies()` creates alerts but NOT action_items. |
| `src/app/api/outreach/record-manual-send/route.ts` | L142-189 | DOES auto-create follow-up action_items from outreach sends. This is the pattern to follow. |

### Schema State

`action_items` table columns (combined from 4 migrations):
```
id UUID PK
capture_id UUID FK -> conversation_captures
contact_id UUID FK -> contacts
owner TEXT
task TEXT NOT NULL
due_date DATE
status TEXT (open|in_progress|done|cancelled)
priority TEXT (high|medium|low)
notes TEXT
created_at, completed_at, updated_at TIMESTAMPTZ
assigned_to UUID FK -> team_members
created_by TEXT DEFAULT 'milo'
source TEXT DEFAULT 'manual'
source_id UUID
task_type TEXT DEFAULT 'general'
entity_type TEXT (publisher|buyer|campaign|invoice|general)
entity_id UUID
entity_name TEXT
recurrence TEXT (daily|weekly|monthly)
next_recurrence DATE
parent_task_id UUID FK -> action_items
metadata JSONB DEFAULT '{}'
```

### Build Spec

**New file:** `src/lib/alert-to-action-item.ts`

Create a mapping from `action_type` to action_item creation parameters:

```typescript
const ALERT_ACTION_ITEM_MAP: Record<string, {
  task_template: (alert: AlertRow) => string;
  priority: 'high' | 'medium' | 'low';
  task_type: string;
  due_offset_days: number; // 0 = today, 1 = tomorrow, etc.
}> = {
  bid_floor_violation: {
    task_template: (a) => `Review bid floor violation for ${a.entity_name} -- revenue below IO floor`,
    priority: 'high',
    task_type: 'investigation',
    due_offset_days: 0,
  },
  invoice_overdue: {
    task_template: (a) => `${a.action_data?.invoice_type === 'receivable' ? 'Chase' : 'Send'} payment for ${a.entity_name}`,
    priority: 'high',
    task_type: 'billing',
    due_offset_days: 0,
  },
  idle_publisher: {
    task_template: (a) => `Contact ${a.entity_name} -- no calls in 48h despite active routes`,
    priority: 'medium',
    task_type: 'outreach_opportunity',
    due_offset_days: 1,
  },
  ping_rejection_high: {
    task_template: (a) => `Review targeting for ${a.entity_name} -- high ping rejection rate`,
    priority: 'medium',
    task_type: 'investigation',
    due_offset_days: 1,
  },
  repeat_caller: {
    task_template: (a) => `Review CID ${a.entity_name} for possible blacklist -- ${a.action_data?.call_count} calls in 30d`,
    priority: 'medium',
    task_type: 'investigation',
    due_offset_days: 1,
  },
  knowledge_anomaly: {
    task_template: (a) => `Review anomaly: ${a.title}`,
    priority: 'medium',
    task_type: 'investigation',
    due_offset_days: 1,
  },
};
```

**Export function:** `createActionItemFromAlert(alertId: string): Promise<string | null>`

- Fetch alert by ID
- Look up mapping by `action_type`
- Dedup: check for existing open action_item with `source='alert'` and `source_id=alertId`
- Resolve `assigned_to` from alert's `target_user_id` -> `team_members` lookup
- Insert into `action_items` with: `source='alert'`, `source_id=alertId`, entity fields carried from alert
- Return created action_item ID or null if deduped/skipped

**Integration points (files to modify):**

1. `src/lib/knowledge-engine.ts` L802-820: After `supabaseAdmin.from('alerts').insert(...)` succeeds, call `createActionItemFromAlert(newAlertId)`.
2. `src/app/api/cron/evaluate/route.ts`: After each alert insert in the 4 daily sweeps (ping rejection, idle publisher, overdue invoice, repeat caller), call `createActionItemFromAlert()`.
3. `src/app/api/alerts/generate/route.ts` L110-122: After `upsertAlert()` returns `'created'`, call `createActionItemFromAlert()`.

**Estimated complexity:** Medium (3-4 hours). New file + 3 integration points. Dedup logic is critical to avoid flooding.

**Dependencies:** None. Can be built standalone.

---

## Gap 3: No VA-to-VA Handoff Mechanism

### Current Code Locations

| File | Lines | What it does |
|------|-------|-------------|
| `src/lib/eod-composer.ts` | Full file | 7-section EOD report: metrics, yesterday review, conversion delta, invoices, alerts, tasks, priorities. **No handoff section.** |
| `src/app/api/eod/generate/route.ts` | Full file | Per-user EOD generation from activity logs, resolved alerts, completed tasks. |
| `src/app/api/eod/submit/route.ts` | Full file | Manual additions to EOD report + submit. |
| `src/app/api/eod/batch/route.ts` | Exists | Batch EOD generation for all users. |
| `src/components/EODModal.tsx` | Exists | EOD report UI modal. |
| `src/components/BatchEODModal.tsx` | Exists | Batch EOD generation UI. |
| `src/lib/tools/warm-handoff-to-tiffani.ts` | Full file | Prospect-to-CEO handoff package (different concept -- outreach handoff, not shift handoff). |

### Schema State

`eod_reports` table:
```
id UUID PK
user_id UUID FK -> auth.users
user_name TEXT NOT NULL
user_role TEXT
report_date DATE NOT NULL
shift_start, shift_end TIMESTAMPTZ
shift_duration_minutes INT
auto_generated JSONB  -- {milo_queries, alerts_resolved, tasks_completed, conversation_count, ai_summary}
manual_additions JSONB
tasks_completed JSONB
tasks_in_progress JSONB
questions_blockers TEXT
notes_suggestions TEXT
report_type TEXT DEFAULT 'eod'
report_data JSONB DEFAULT '{}'
status TEXT (draft|submitted|reviewed)
reviewed_by, reviewed_at, source, created_at, updated_at
```

### Build Spec

**What's missing:** When VA shift A ends and VA shift B starts, there's no structured summary of:
- Action items left in-progress by A
- Alerts that were viewed but not resolved
- Pipeline entities touched but not advanced
- Any context notes from A for B

**New file:** `src/lib/handoff-composer.ts`

```typescript
export interface ShiftHandoff {
  from_user: string;
  to_user: string;
  handoff_date: string;
  in_progress_tasks: ActionItemSummary[];
  unresolved_alerts: AlertSummary[];
  active_pipeline_touches: PipelineTouchSummary[];
  context_notes: string[];  // from manual_additions + notes_suggestions of EOD
  ai_handoff_summary: string;
}
```

**Function:** `composeHandoff(fromUser: string, reportDate: string): Promise<ShiftHandoff>`

1. Query `action_items` WHERE `status='in_progress'` AND `assigned_to` matches fromUser
2. Query `alerts` WHERE `status='active'` AND `target_user_id` matches fromUser
3. Query `ai_action_log` for fromUser's actions today (entity touches)
4. Query `eod_reports` for fromUser's latest submitted report -- extract `tasks_in_progress` and `notes_suggestions`
5. Call Claude (structured tier) to synthesize a handoff summary
6. Return `ShiftHandoff`

**New API route:** `src/app/api/eod/handoff/route.ts`

- POST: Generate handoff from one user to another
- GET: Fetch latest handoff for current shift

**Migration:** `20260507000001_eod_handoffs.sql`

```sql
CREATE TABLE IF NOT EXISTS shift_handoffs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  from_user TEXT NOT NULL,
  to_user TEXT,
  handoff_date DATE NOT NULL,
  handoff_data JSONB NOT NULL DEFAULT '{}',
  ai_summary TEXT,
  status TEXT DEFAULT 'pending' CHECK (status IN ('pending', 'acknowledged', 'reviewed')),
  acknowledged_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ DEFAULT NOW()
);
CREATE INDEX idx_handoffs_date ON shift_handoffs(handoff_date DESC);
CREATE INDEX idx_handoffs_to ON shift_handoffs(to_user, handoff_date DESC);
```

**Integration with cron:** Add to `src/app/api/cron/morning-assignments/route.ts` -- auto-generate handoff when shift overlap detected (fromUser's EOD exists for yesterday, toUser's morning assignment is being generated).

**Estimated complexity:** Large (6-8 hours). New table, new composer, new API route, cron integration.

**Dependencies:** Depends on Gap 2 (auto action_items from alerts) for completeness, but can be built independently using existing `action_items` data.

---

## Gap 4: Per-VA Outreach Tracking (Missing sent_by)

### Current Code Locations

| File | Lines | What it does |
|------|-------|-------------|
| `supabase/migrations/20260506000003_outreach_log.sql` | L12 | `performed_by text DEFAULT 'milo'` -- exists but always defaults to 'milo' |
| `src/app/api/ask/outreach-log/route.ts` | L116 | Sets `performed_by: 'milo-outreach'` -- hardcoded |
| `src/app/api/ask/vet-action/route.ts` | L120 | Sets `performed_by: 'milo-vet'` -- hardcoded |
| `src/app/api/outreach/send/route.ts` | L94-108 | Logs to `ai_action_log` but does NOT log to `outreach_log`. No user tracking. |
| `src/app/api/outreach/record-manual-send/route.ts` | L46-64 | Logs to `send_log` (milo-outreach DB). Sets source_context but no operator ID. |
| `src/app/api/outreach/draft/route.ts` | Exists | Draft generation -- no operator tracking |
| `src/app/api/outreach/draft-followup/route.ts` | Exists | Follow-up draft -- no operator tracking |

### Schema State

**`outreach_log` table** (milo-for-ppc, migration 20260506000003):
```
id uuid PK
entity_name text NOT NULL
entity_type text DEFAULT 'unknown'
channel text NOT NULL  -- linkedin_intro|linkedin_followup|email_cold|teams|other
action text DEFAULT 'drafted'  -- drafted|copied|sent|responded
message_text text
pipeline_entity_id uuid
crm_counterparty_id uuid
tenant_id text DEFAULT 'tlp'
performed_by text DEFAULT 'milo'   <-- THIS IS THE FIELD
follow_up_at timestamptz
responded_at timestamptz
created_at timestamptz
```

**`send_log` table** (shared Supabase, milo-outreach):
- Does NOT have a `sent_by` or `performed_by` column
- Has `source_context JSONB` which could carry operator info

**`ai_action_log` table** (milo-for-ppc):
- No `performed_by` column; uses `entity_name` for the target entity
- `data_snapshot` JSONB could carry operator info but doesn't

### Build Spec

The `performed_by` column already exists on `outreach_log`. The gap is that:
1. The API routes always hardcode it to `'milo'`, `'milo-outreach'`, or `'milo-vet'`
2. `send_log` (shared DB) has no operator tracking at all

**Files to modify:**

1. **`src/app/api/outreach/send/route.ts`** -- L28-33: Extract user from `resolveRequestUser(req)`. Pass `user.displayName` to the outreach log insert. Also insert into `outreach_log` (currently only logs to `ai_action_log` and `send_log`, NOT `outreach_log`).

2. **`src/app/api/outreach/record-manual-send/route.ts`** -- L25-32: Extract user identity. Pass to `performed_by` on `send_log` insert (add to `source_context`) and insert an `outreach_log` row with the operator name.

3. **`src/app/api/ask/outreach-log/route.ts`** -- Accept `performed_by` from request context instead of hardcoding.

4. **`src/app/api/ask/vet-action/route.ts`** -- Same: accept operator from request context.

5. **`send_log` migration** (in milo-outreach repo or shared migrations):
```sql
ALTER TABLE send_log ADD COLUMN IF NOT EXISTS sent_by TEXT;
```

**Note on auth context:** Auth was removed per Decision 186. However, `resolveRequestUser()` still works -- it returns the logged-in user if present, or null for anonymous. The operator identity is available via the `/ask` page context where `sent_by_admin_id` and `sent_by_admin_name` are already tracked on `/api/milo/route.ts` (L108-112). This same pattern should be threaded through outreach endpoints.

**Estimated complexity:** Small-Medium (2-3 hours). Column already exists on `outreach_log`. Main work is threading operator identity through 4-5 route handlers.

**Dependencies:** None. Independent of other gaps.

---

## Gap 5: EOD Follow-ups --> Auto-Created Action Items

### Current Code Locations

| File | Lines | What it does |
|------|-------|-------------|
| `src/lib/eod-composer.ts` | L786-941 | Section 7 "Tomorrow Priorities" -- identifies open alerts, snoozed alerts resuming, overdue tasks, invoices due, and unresolved yesterday items. **All prose, no action_items created.** |
| `src/app/api/eod/generate/route.ts` | L147-195 | AI summary has `follow_ups: string[]` field. These are free-text strings stored in `auto_generated.ai_summary.follow_ups`. **Never parsed into action_items.** |
| `src/app/api/eod/submit/route.ts` | L56-73 | Accepts `tasks_in_progress` and `manual_additions` as arrays but stores them as raw JSON in `eod_reports`. **Never creates action_items.** |
| `src/app/api/outreach/record-manual-send/route.ts` | L142-189 | **Reference pattern**: auto-creates 3 follow-up action_items from outreach with specific due dates, priorities, and dedup source_ids. |

### Schema State

Same `action_items` schema as Gap 2.

`eod_reports.auto_generated` JSONB structure:
```json
{
  "milo_queries": [...],
  "alerts_resolved": [...],
  "tasks_completed": [...],
  "conversation_count": number,
  "ai_summary": {
    "summary": "string",
    "categories": [...],
    "highlight": "string",
    "follow_ups": ["string", "string", ...]
  }
}
```

`eod_reports.report_data` JSONB structure (from eod-composer):
```json
{
  "line_items": [
    {
      "item_id": "string",
      "section": "priorities",
      "description": "Open alert: ...",
      "entity_name": "string",
      "severity": "warning|critical",
      "source_table": "alerts|action_items|invoices",
      "source_id": "uuid"
    }
  ]
}
```

### Build Spec

**New file:** `src/lib/eod-follow-up-creator.ts`

```typescript
export async function createFollowUpsFromEOD(reportId: string): Promise<number>
```

1. Fetch EOD report by ID
2. Extract follow-up items from two sources:
   - `auto_generated.ai_summary.follow_ups[]` -- free-text, needs parsing
   - `report_data.line_items[]` WHERE `section='priorities'` AND `severity IN ('warning','critical')` AND `resolved !== true`
3. For each item:
   - Dedup: check `source='eod_follow_up'` + `source_id='eod_{reportId}_{item_index}'`
   - Map severity to priority: critical -> high, warning -> medium
   - Extract entity_name from item if available
   - Set `due_date` to tomorrow
   - Resolve `assigned_to` based on entity routing (same pattern as alert routing)
4. Batch insert into `action_items`
5. Return count of created items

**Integration points:**

1. **`src/app/api/eod/submit/route.ts`** -- After marking report as 'submitted', call `createFollowUpsFromEOD(reportId)`. This ensures follow-ups are only created once (on submit, not on draft save).

2. **`src/app/api/eod/persist-and-send/route.ts`** -- If this route handles system-generated EOD persistence, also call `createFollowUpsFromEOD()` after persist.

**AI summary follow-ups parsing:** The `follow_ups` array from Claude is already structured as individual strings. Each string becomes one action_item. The AI system prompt in `eod/generate/route.ts` (L178-185) already asks for follow-ups in a list format. No additional parsing needed -- just map string to task title.

**Estimated complexity:** Medium (3-4 hours). New file + 2 integration points. Dedup and entity routing are the key implementation details.

**Dependencies:** Independent, but shares dedup pattern with Gap 2. If Gap 2 is built first, `alert-to-action-item.ts` dedup logic can be shared.

---

## Gap 6: Kanban Gate counterparty_id Mapping Verification

### Current Code Locations

| File | Lines | What it does |
|------|-------|-------------|
| `src/app/api/pipeline/validate-transition/route.ts` | Full file | Stage transition validation with gate checks |
| `src/app/api/pipeline/[id]/stage/route.ts` | Full file | Stage change execution with activation gate |
| `src/app/api/pipeline/route.ts` | Full file | Pipeline listing -- uses `prospect_pipeline.id`, NOT `counterparty_id` |

### Schema State

**`prospect_pipeline` table:**
```
id UUID PK
entity_type TEXT (publisher|buyer)
entity_name TEXT NOT NULL
contact_id UUID FK -> contacts
stage TEXT
source TEXT
...
stage_changed_at TIMESTAMPTZ
```

**Key finding: There is NO `counterparty_id` column on `prospect_pipeline`.**

The `prospect_pipeline.id` is the entity's primary key. The relationship to CRM data goes through:
- `prospect_pipeline.entity_name` matched against `crm_counterparties.company` (case-insensitive)
- `prospect_pipeline.contact_id` FK to `contacts.id`
- `contacts.counterparty_id` FK to `crm_counterparties.id`

**Other tables that use `counterparty_id`:**
- `outreach_log.crm_counterparty_id` -- optional FK, not enforced
- `vet_results.crm_counterparty_id` -- optional FK
- `onboarding_runs.counterparty_id` -- used by `onboarding-stall-check` cron
- `contacts.counterparty_id` -- FK to `crm_counterparties`

### Analysis of the Alleged Gap

**The validate-transition API (`/api/pipeline/validate-transition/route.ts`):**

1. Accepts `entity_id` (which is `prospect_pipeline.id`) + `from_stage` + `to_stage`
2. Fetches the entity from `prospect_pipeline` by `id`
3. For **outreach -> qualifying** gate: Checks `outreach_log` by `entity_name` (L36-37), then `ai_action_log` by `entity_name` (L47-50)
4. For **onboarding/activation/active** gates: Calls `getOnboardingStatusByCounterparty(entityId)` -- but this function receives `prospect_pipeline.id`, NOT a `crm_counterparties.id`

**The `getOnboardingStatusByCounterparty()` function** (in `src/lib/onboarding-client.ts`) likely queries `onboarding_runs.counterparty_id`. The confusion arises because:
- `validate-transition` passes `entity_id` (= `prospect_pipeline.id`)
- `getOnboardingStatusByCounterparty` expects a counterparty ID
- These are different UUIDs in different tables

**The stage change API (`/api/pipeline/[id]/stage/route.ts`):**

1. For activation gate: Calls `checkSignedDocumentsGate(entityName)` which queries by `entity_name` (string match), NOT by ID
2. This works correctly because it uses name-based matching

### Verdict: There IS a mapping gap, but it's narrower than expected

The gap is specifically in the `validate-transition` route's onboarding/activation/active gate checks. It passes `prospect_pipeline.id` to `getOnboardingStatusByCounterparty()`, but that function expects `crm_counterparties.id`. If the IDs happen to never match (which is likely since they're in different tables), the gate check would always fail with "No onboarding run found."

**However**, this may be masked because:
1. The `[id]/stage` route (which actually executes the stage change) uses `entity_name` for its gate check, which works correctly
2. `validate-transition` may not be called in all flows (the pipeline board might skip validation and go directly to the stage change API)

### Build Spec

**Option A: Fix validate-transition to use entity_name consistently (Recommended)**

Modify `src/app/api/pipeline/validate-transition/route.ts`:

1. `checkOnboardingGate(entityId, entityType)` -- change to also pass `entityName`. Inside the function, call `getOnboardingStatusByCounterparty()` using a counterparty ID resolved from `entity_name`:

```typescript
async function resolveCounterpartyId(entityName: string): Promise<string | null> {
  const { data } = await supabaseAdmin
    .from('crm_counterparties')
    .select('id')
    .ilike('company', entityName)
    .limit(1)
    .maybeSingle();
  return data?.id ?? null;
}
```

2. Update `checkOnboardingGate`, `checkActivationGate`, `checkActiveGate` to resolve counterparty_id from entity_name before calling `getOnboardingStatusByCounterparty()`.

3. Add fallback: if `getOnboardingStatusByCounterparty()` returns null with the resolved counterparty_id, try again with `prospect_pipeline.id` (in case some onboarding_runs used the pipeline ID as counterparty_id during initial setup).

**Option B: Add counterparty_id FK to prospect_pipeline**

```sql
ALTER TABLE prospect_pipeline ADD COLUMN IF NOT EXISTS counterparty_id UUID;
-- Backfill from crm_counterparties by entity_name match
UPDATE prospect_pipeline pp
SET counterparty_id = (
  SELECT id FROM crm_counterparties
  WHERE LOWER(company) = LOWER(pp.entity_name)
  LIMIT 1
);
```

Then update `validate-transition` to use `prospect.counterparty_id`.

**Recommendation:** Option A is safer (no schema change, no backfill risk). Option B is cleaner long-term but requires a migration + backfill script.

**Estimated complexity:** Small-Medium (2-3 hours). 1 file to modify, 1 new helper function.

**Dependencies:** None. Independent of other gaps.

---

## Cross-Gap Dependencies

```
Gap 1 (action_labels) -----> standalone
Gap 2 (auto action_items) -> standalone, but shared dedup pattern with Gap 5
Gap 3 (VA handoff) ---------> benefits from Gap 2 (auto action_items make handoff richer)
Gap 4 (sent_by) ------------> standalone
Gap 5 (EOD follow-ups) -----> shares dedup pattern with Gap 2
Gap 6 (kanban mapping) -----> standalone
```

**Recommended build order:**
1. Gap 1 (Small -- quick win, improves operator experience immediately)
2. Gap 4 (Small-Medium -- independent, no schema migration needed for outreach_log)
3. Gap 6 (Small-Medium -- bug fix with clear scope)
4. Gap 2 (Medium -- foundational for Gap 5)
5. Gap 5 (Medium -- builds on Gap 2 patterns)
6. Gap 3 (Large -- depends on other gaps being solid first)

**Combined opportunities:**
- Gaps 2 + 5 share the action_item dedup pattern. Build a shared `dedup-action-item.ts` utility that both use.
- Gaps 1 + 2 can be done in the same commit since both touch `alerts/generate/route.ts`.

---

## Self-Attestation Checklist

- [x] Each gap characterized with exact file paths and line numbers
- [x] Schema state documented from actual migration SQL files
- [x] Build specs include exact files to modify, new files to create, and SQL where needed
- [x] Complexity estimates provided for each gap
- [x] Dependencies between gaps mapped
- [x] No production code modified (read-only audit)
- [x] Recommended build order provided

## Known Gaps in This Report

- **Gap 6 validation:** Could not run the actual `getOnboardingStatusByCounterparty()` function to confirm the ID mismatch. The mapping gap is inferred from code analysis, not runtime verification. Mark should verify by checking whether any `onboarding_runs.counterparty_id` values match `prospect_pipeline.id` values in production.
- **send_log schema:** The `send_log` table is defined in milo-outreach, not milo-for-ppc. The migration for adding `sent_by` to `send_log` needs to go in the milo-outreach repo.
- **No UI surfaces touched:** Screenshot N/A -- this is a pure research/spec deliverable.

---

Report at: https://raw.githubusercontent.com/moscaleflow/conductor-mark-public/main/reports/C3-ALL-GAPS-AUDIT-2026-05-07.md
