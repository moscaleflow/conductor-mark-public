# Alert Assignment, Next-Step Continuity & Session Persistence Audit

**Date:** 2026-05-06
**Repo:** milo-for-ppc
**Auditor:** Coder-3 (read-only)

---

## Section 1: Alert Assignment Capability

### Schema

The `alerts` table has a `target_user_id` column (UUID, nullable). This column is populated during alert generation and is the primary mechanism for per-operator routing.

### Alert Routing Logic (`src/lib/alert-routing.ts`)

The system resolves `target_user_id` through a 5-step chain:

1. **Explicit `routeTo` name** -- from `action_data.route_to`. Resolves display_name to user_id via `user_profiles`.
2. **CRM assigned_to** -- looks up `crm_counterparties.assigned_to` by company name. If an entity (buyer/publisher) is assigned to an operator in CRM, their alerts route to that operator.
3. **Entity-type default routing** -- hardcoded defaults:
   - `publisher` -> Jackie
   - `buyer` -> Fab
   - `caller` -> Jackie
   - `did` -> Malvin
   - `campaign` -> Malvin
4. **Billing/reconciliation alerts** -- `invoice_overdue` and `ap_overdue_review` route to Jen.
5. **Fallback** -- `null` (broadcast to all operators).

### Assignment API

- **`POST /api/alerts/[id]/assign`** -- Reassigns an alert to a different user. Accepts `target_user_id` and `assigned_by`. Writes audit trail to `ai_action_log`.
- **`POST /api/operator/triage/action`** -- Unified triage action with `action: 'assign'` + `target_user_id`. Also writes audit trail.
- **`TeamAssignDropdown`** -- UI component imported in TriageDrawer that provides operator selection dropdown.

### "My Alerts" vs "All Alerts" Filter

**The operator pill endpoint (`/api/operator/pill`) DOES filter by user.** Line 79: `.eq('target_user_id', userId)` -- each operator's pill view only shows alerts assigned to them or broadcast (null target).

**The triage drawer endpoint (`/api/operator/triage`) does NOT filter by user.** It returns ALL active alerts regardless of assignment. The comment in the code says: "Auth removed (Decision 186 + C3 audit 2026-05-05): All callers now receive the admin view (unfiltered alerts)."

**GAP:** The triage drawer is the primary alert review surface, but it shows everyone's alerts. There is no "my alerts" / "all alerts" toggle in the TriageDrawer UI. Operators see every alert, which is the opposite of "alerts are assigned to specific VAs."

### Alert Generation Routing

Every alert created by `/api/alerts/generate` calls `resolveAlertTargetUser()` which populates `target_user_id`. Entity-type inference from title patterns covers: publisher, buyer, campaign, and invoice entities. When entity_type is not explicitly set and the entity_name exists, the title pattern is used to infer entity type (e.g., "Quality flag" -> publisher, "Buyer health" -> buyer).

---

## Section 2: Alert -> Next Action Mapping

The system generates 26+ alert types. Here is the complete mapping:

### Critical Alerts

| Alert Title | action_label | Next Action Available? | What Happens on Click |
|---|---|---|---|
| Converted calls with $0 payout | Investigate | AI-enriched review (WHAT I SEE / IMPACT / MY TAKE), then snooze/resolve/assign/draft | No direct navigation -- stays in TriageDrawer |
| Buyer line not answering | Investigate | AI-enriched review, expanded_data shows entity list with per-buyer counts | No direct entity navigation |
| Buyer health critical | Investigate | AI-enriched review, expanded_data has score/grade/components | No navigation to buyer page |
| Buyer payment overdue | Review | AI-enriched review | No navigation to invoice |
| Floor bid violation | Investigate | AI-enriched review, route_to: fab in action_data | No direct action |
| Revenue variance | Investigate | AI-enriched review, expanded_data has worst_calls | No direct action |
| Config mismatch -- route below floor | Fix Config | AI-enriched review | No navigation to config |
| Invoice overdue | Send Follow-Up | action_data.draft_message present -- DraftMessageCard renders | Can copy follow-up message |
| Buyer report overdue | Send Follow-Up | action_data.draft_message present -- DraftMessageCard renders | Can copy follow-up draft |

### Warning Alerts

| Alert Title | action_label | Next Action Available? | What Happens on Click |
|---|---|---|---|
| Quality flag -- {publisher} | Review | AI-enriched review | No navigation to publisher |
| Call volume down {pct}% | Review | AI-enriched review | No entity to navigate to |
| Campaign revenue disappeared | Check TrackDrive | AI-enriched review, expanded_data has entity list | No navigation to TrackDrive |
| Publisher grade dropped | Review | AI-enriched review | No navigation to publisher |
| RTB bids declining | Review Bids | AI-enriched review, action_data has campaign_id | No navigation |
| RTB bid below $20 | Review Bids | AI-enriched review | No navigation |
| Ping waste | Review Publisher | AI-enriched review, action_data has publisher_name | No navigation |
| Getting outbid | Review Bids | AI-enriched review, action_data has campaign details | No navigation |
| Ping cost exceeds revenue | Review Economics | AI-enriched review | No navigation |
| Duplicate converted call | Review | AI-enriched review, expanded_data has call_uuids | No navigation |
| RTB bid below $20 minimum | Review | AI-enriched review | No navigation |
| Publisher idle | Review | AI-enriched review, action_data has route_id | No navigation |
| High TTC | Review | AI-enriched review | No navigation |
| Reconciliation discrepancy | Review | AI-enriched review, expanded_data has mismatched_cids | No navigation |
| Stale buyer_rate | Verify Rate | AI-enriched review | No navigation |
| No buyer_revenue configured | Configure Rates | AI-enriched review | No navigation |
| Ping-to-call delay + below floor | Investigate | AI-enriched review | No navigation |

### Opportunity Alerts

| Alert Title | action_label | Next Action Available? | What Happens on Click |
|---|---|---|---|
| Publisher capacity available | View | AI-enriched review, expanded_data has entity list | No navigation |
| Weekly invoices ready to generate | Generate Invoices | action_data: `{ action: 'navigate', path: '/invoices' }` | **Has navigation data but TriageDrawer does not implement navigation** |
| Monthly invoices ready to generate | Generate Invoices | action_data: `{ action: 'navigate', path: '/invoices' }` | Same -- navigation data exists but is not consumed |

### Multi-Step Alerts (outreach chain)

| Step | Title | action_label | Next Action |
|---|---|---|---|
| Step 1 | Original outreach alert | Draft outreach | Resolving auto-spawns Step 2 |
| Step 2 | Waiting for reply from {entity} | Check status | Reply detection or stale timeout -> Step 3 |
| Step 3 | No reply from {entity} -- follow up needed | Draft follow-up | Terminal step |

**KEY FINDING: Alert `action_data.action === 'navigate'` and `action_data.path` are stored but NEVER consumed by the TriageDrawer UI.** The "Generate Invoices" alert stores `path: '/invoices'` but the drawer has no handler for navigation actions. The only actions the TriageDrawer dispatches are: `review`, `snooze`, `resolve`, `assign`, and `draft_outreach` (via AI suggestedAction).

**KEY FINDING: No alert type navigates to the contract page, entity page, or onboarding checklist.** The expand-in-place pattern (WHAT I SEE / IMPACT / MY TAKE) provides context but does not navigate the operator to the entity where they take action. The operator must manually navigate.

---

## Section 3: Next-Step Continuity Across Surfaces

### Search Results (StructuredResponseCard)

`StructuredResponseCard` renders structured AI responses with sections like "Key Takeaway" and "Actionable Items." It supports entity discovery (publisher target lists with name/website/contact cards, each with "Add to CRM" buttons).

**GAP:** There is no "Next Step" indicator on entity search results. When a VA searches for an entity, the response shows information cards but does not show "what should I do next with this entity." The VA must parse the AI narrative to determine next steps.

### Vet Completion

When vetting completes:
- Vet results are rendered as verdict cards with action buttons: "Add to CRM", "Draft Outreach", "Mark as Sent", "Research more"
- The `markedSent` state is tracked in `context_metadata.opportunity_actions` and persists across page reload via the history endpoint
- `vet-action` API supports actions: add to CRM, draft outreach, mark as sent

**GOOD:** Vetting has clear next-step actions (add to CRM, draft outreach). The buttons are action-oriented.

**GAP:** After adding to CRM, there is no automatic "next: send onboarding link" indicator. The VA must know the workflow.

### Contract Upload (DocumentDropzone)

The `DocumentDropzone` component has a comment at line 277: "Determine next step based on entity match" -- when a document is uploaded, the code determines the entity and redirects accordingly.

**PARTIAL:** The contract upload flow does suggest next steps contextually, but the logic is in the dropzone component, not in a global "what's next" tracker.

### Onboarding Steps (OnboardingChecklist)

The `OnboardingChecklist` component shows step-by-step progress with:
- `next_step` field displayed prominently ("Next: {next_step}")
- Each step has status indicators (completed/active/pending/skipped/failed/blocked)
- Stalled detection: the `onboarding-stall-check` cron creates action_items when steps stall for 72+ hours

**GOOD:** Onboarding has the best next-step continuity in the system. Each step shows what's next, and stalled steps auto-create action items.

### EntityDetailPanel

The panel shows `partnerOnboarding.next_step` in the onboarding section (line 3674, 3744-3751). When `pct < 100`, it shows "Next: {next_step}".

**GOOD:** Entity detail panel surfaces onboarding next-step.

**GAP:** Entity detail panel does NOT show a universal "next step" for non-onboarding contexts. If the entity has pending invoices, unresolved alerts, or open action items, those are not surfaced as "next step" on the entity page.

---

## Section 4: Session Persistence Assessment

### Chat History (`/api/ask/history`)

- **Scope:** Returns today's messages for the same IP address (SHA-256 hashed)
- **Identification:** IP-based (hashed), no user_id association
- **Data returned:** Up to 30 message pairs from today, including vet results and markedSent state
- **Session gap:** Comment says "Session gap logic removed -- operators should see their full day's work"
- **Vet result restoration:** Yes -- joins `vet_results` table by `chat_log_id` and restores verdict cards
- **markedSent state:** Yes -- restored from `context_metadata.opportunity_actions` on the `chat_logs` row

**GOOD:** Within a single day, a VA closing their browser and reopening sees their full day's conversation history with vet cards and action states preserved.

**GAP: History resets daily.** The system only loads today's messages. If a VA worked on something yesterday and comes back today, they have zero context from the previous session.

**GAP: IP-based identification.** If a VA's IP changes (VPN reconnect, network change), their history is lost. There is no user_id binding on chat history.

**GAP: Action items are NOT displayed in the chat session.** If the VA created action items or had alerts resolved during a previous session, those are not surfaced when they open the chat. The VA must separately check the action items list or triage drawer.

### What Survives Page Refresh

| Data | Survives Refresh? | Survives Next Day? | Notes |
|---|---|---|---|
| Chat messages | Yes | No | IP-based, today only |
| Vet results | Yes | No | Joined from vet_results table |
| Mark-as-sent state | Yes | No | In context_metadata |
| Alert assignments | Yes | Yes | In alerts table (not session-bound) |
| Action items | Yes | Yes | In action_items table (not session-bound) |
| Onboarding progress | Yes | Yes | In onboarding_runs/steps tables |
| EOD reports | Yes | Yes | In eod_reports table |

---

## Section 5: Action Items as Workflow Driver

### Schema and API

The `action_items` table is robust:
- **Fields:** task, owner, assigned_to (UUID -> team_members), due_date, status (open/in_progress/done/cancelled), priority (high/medium/low), source (manual/conversation/briefing/alert/cron/onboarding), source_id, entity_type/entity_id/entity_name, recurrence, notes, metadata
- **Linked to entities:** Yes -- via entity_type, entity_id, entity_name
- **Linked to alerts:** Partially -- via `source: 'alert'` and `source_id` (alert UUID) when created from `/api/action-items/from-alert`
- **Linked to contacts:** Yes -- via contact_id FK to contacts table
- **Recurrence support:** Yes -- daily/weekly/monthly with automatic next-occurrence spawning on completion

### Filtering Capabilities

`GET /api/action-items` supports:
- `assigned_to` -- filter by team member display name
- `entity_name` -- filter by linked entity
- `entity_type` -- publisher/buyer/campaign/invoice/general
- `due` -- overdue/today/week
- `source` -- manual/conversation/briefing/alert/cron/onboarding
- `status` -- open/in_progress/done/cancelled/all
- `contact_id` -- specific contact

### Source Tracking

Action items track their origin:
- `source: 'alert'` -- created from an alert via `/api/action-items/from-alert`
- `source: 'onboarding_stall'` -- created by the onboarding-stall-check cron
- `source: 'conversation'` -- created during chat interaction
- `source: 'manual'` -- operator-created
- `source: 'cron'` -- system-generated recurring tasks

**GAP: There is no "action item widget" on the operator dashboard.** The action items API is full-featured but there is no `ActionItemWidget` or `TaskPanel` component found in the codebase. Action items exist as API data but are not visually surfaced as a persistent widget on the operator's main view.

**GAP: No "all my pending action items" single view.** While the API supports `assigned_to` filtering, there is no dedicated "My Tasks" page or panel. The VA must know to query their tasks explicitly through Milo chat or use the API directly.

**GAP: Action items created from alerts use the raw alert title as the task.** The `from-alert` endpoint copies `alert_text` directly to `task`. There is no transformation to a clear, actionable instruction (e.g., an alert "Buyer line not answering" becomes a task "Buyer line not answering" -- not "Call buyer X to confirm their lines are active").

---

## Section 6: EOD / Handoff State

### EOD Report System

The system has a comprehensive EOD report pipeline:

1. **Auto-generation** (`/api/eod/generate`): Queries the day's data:
   - `milo_activity_log` -- VA's queries and intents
   - `alerts` -- resolved by the VA today
   - `action_items` -- completed today
   - `milo_conversations` -- message count
   - Synthesizes via Claude into structured summary with categories, highlights, and follow-ups

2. **Manual additions** (`/api/eod/submit`): VA can add notes, tasks in progress, questions/blockers

3. **Persistence and delivery** (`/api/eod/persist-and-send`): Upserts to `eod_reports` table, sends via Teams

4. **Review page** (`/app/eod-reports/page.tsx`): Admin can view all EOD reports, mark as reviewed

5. **Batch mode** (`/api/eod/batch`): Generates EOD reports for all active operators

### What the EOD Report Contains

- `auto_generated`: milo_queries, alerts_resolved, tasks_completed, conversation_count, ai_summary
- `manual_additions`: operator-added notes
- `tasks_completed` and `tasks_in_progress`: operator-tracked
- `questions_blockers`: open items
- `notes_suggestions`: freeform

**GOOD:** The EOD system is well-designed for shift summaries. It auto-generates based on actual activity data.

### Handoff Capabilities

**GAP: There is NO explicit handoff mechanism.** There is no:
- "Hand off to [VA name]" action
- "Pending work transfer" between VAs
- "Shift start" briefing that pulls the previous VA's pending items
- Way for an incoming VA to see what the outgoing VA left undone

The warm_handoff_to_tiffani tool exists but is for sales handoffs to the CEO, not for VA-to-VA operational handoffs.

**PARTIAL: Morning assignments cron** (`/api/cron/morning-assignments`) exists, which suggests some daily task distribution happens, but it is not a handoff of in-progress work -- it is a fresh assignment.

---

## Section 7: Gap List -- What Breaks the "Always a Next Step" Principle

### Critical Gaps

1. **Alerts do not navigate to the entity.** When a VA clicks an alert and expands it, they see AI context (WHAT I SEE / IMPACT / MY TAKE) but there is no button to go to the buyer page, publisher page, contract page, or onboarding checklist. The `action_data.path` field exists for invoice alerts but the UI does not consume it. The VA must close the drawer and manually navigate.

2. **TriageDrawer shows ALL alerts, not "my alerts."** Despite the sophisticated target_user_id routing system, the triage drawer returns unfiltered alerts. The operator pill view DOES filter, but the primary alert triage surface does not. A VA sees everyone's alerts and must mentally filter.

3. **No universal "next step" on entity pages.** When a VA looks at a buyer or publisher, there is no "Here is what needs to happen next" summary. Onboarding has next_step, but outside onboarding, the VA must check multiple surfaces (alerts, action items, contracts) to determine what to do.

4. **No action item dashboard widget.** Action items are a robust backend system but have no persistent UI surface. A VA cannot see "I have 5 open tasks due today" without explicitly asking Milo or navigating to a non-existent tasks page.

5. **Chat history resets daily.** A VA loses all yesterday's context. If they were mid-flow on an entity (e.g., vetting -> drafting outreach -> waiting for reply), today they start with an empty chat.

6. **No VA-to-VA handoff.** When a shift ends, there is no mechanism to transfer in-progress work to the next VA. The EOD report summarizes what happened but does not create actionable items for the next person.

### Moderate Gaps

7. **Alert "action_label" is generic.** Most alerts say "Review" or "Investigate" -- these are not specific next actions. "Review" for a quality flag should say "Check ConvoQC flags for {publisher}" or "Pause route if flag rate stays above 25%." The action_label does not tell the VA WHAT to do.

8. **No automatic action_item creation from alerts.** Alerts and action items are loosely linked. The `from-alert` endpoint exists but alerts do not auto-create action items when generated. The VA must manually convert an alert to a task.

9. **Search results lack "Next Step" indicators.** When a VA searches for an entity, the StructuredResponseCard shows data but not "what's pending for this entity." It should show: "3 open alerts, 1 stalled onboarding step, invoice overdue."

10. **AI suggestedAction is the only next-step hint on alerts.** The `/api/operator/pill/review` endpoint returns a `suggestedAction` object with a label and action type, but this is AI-generated at expand time (not pre-computed). If the AI call fails or times out (10s limit), the alert shows no suggested action at all.

11. **Invoice opportunity alerts have navigation data that is never used.** `action_data: { action: 'navigate', path: '/invoices' }` is stored but the TriageDrawer does not read `action_data.action` or `action_data.path` to render a navigation button.

12. **IP-based session identity is fragile.** If a VA's IP changes, their chat history and session context disappear. There is no user_id binding on the history endpoint.

### Minor Gaps

13. **Multi-step alert chains are linear only.** The step chain system supports outreach-followup (3 steps). There is no chain for other workflows (onboarding, contract negotiation, dispute resolution). Each of those workflows has its own separate tracking but not alert-chain integration.

14. **EOD follow_ups are narrative, not action_items.** The AI summary produces `follow_ups` as prose strings. These are not automatically created as action_items for the next day. A VA reading an EOD report must manually create tasks from follow-ups.

15. **Alert review audit exists but review state is not visible.** `POST /api/alerts/[id]/review` logs that an alert was reviewed, but the TriageDrawer does not show "reviewed by Jackie at 2pm" on the alert row. Other operators cannot see who has already looked at an alert.

---

## Summary Assessment

| Area | Rating | Key Issue |
|---|---|---|
| Alert assignment (backend) | STRONG | target_user_id routing chain with 5 levels, assignment API, audit trail |
| Alert assignment (UI) | WEAK | Triage drawer shows ALL alerts, no my/all toggle |
| Alert -> action chain | MODERATE | AI-enriched expand-in-place is good, but no navigation to entity, no direct action buttons |
| Next-step in search | WEAK | No "next step" indicator on search results or entity cards |
| Next-step in vet | STRONG | Clear action buttons (add CRM, draft outreach, mark sent) |
| Next-step in onboarding | STRONG | next_step field, stall detection, auto action_items |
| Session persistence | MODERATE | Today's history + vet results survive refresh; resets daily, IP-based |
| Action items (backend) | STRONG | Full CRUD, assignment, recurrence, source tracking, entity linking |
| Action items (UI surface) | MISSING | No dashboard widget, no "My Tasks" page |
| EOD / handoff | MODERATE | Good EOD auto-generation; no VA-to-VA handoff mechanism |

**The backend infrastructure for "always a next step" is largely in place.** The alert routing, action item schema, onboarding step tracking, and multi-step alert chains provide the data foundation. **The gap is in the UI layer and workflow linkage:** alerts don't navigate to entities, action items have no dashboard surface, and the triage drawer doesn't filter to "my alerts." These are implementation gaps, not architecture gaps.

---

Report at: https://raw.githubusercontent.com/moscaleflow/conductor-mark-public/main/reports/alert-nextstep-audit-2026-05-06.md
