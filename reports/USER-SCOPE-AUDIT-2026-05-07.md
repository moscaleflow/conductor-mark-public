# User-Scope Surface Audit -- Milo for PPC

**Date:** 2026-05-07  
**Author:** Coder-3 (Research)  
**Repo:** `~/Documents/GitHub/milo-for-ppc/`  
**Purpose:** Identify every surface where new team members need to appear for multi-tenant onboarding readiness.

---

## 1. Team Members API -- How It Works

### GET /api/team-members (src/app/api/team-members/route.ts)

- Calls `getCanonicalTeamRoster()` from `src/lib/team-roster.ts`
- Returns: `{ members: [{ display_name, team, roles, is_admin, is_active }] }`

### getCanonicalTeamRoster() (src/lib/team-roster.ts:240-297)

**Critical finding: HARDCODED FILTER BLOCKS NEW MEMBERS**

The function queries `user_profiles` but then **filters results against the hardcoded TEAM_ROSTER constant** (lines 262-267):

```ts
const rosterNames = new Set(
  TEAM_ROSTER.map((r) => r.display_name.toLowerCase()),
);
const filtered = data.filter((row) =>
  rosterNames.has((row.name as string).toLowerCase()),
);
```

This means: **A new team member who joins via invite will have a `user_profiles` row but will be EXCLUDED from getCanonicalTeamRoster() because their name is not in the hardcoded TEAM_ROSTER array.**

The TEAM_ROSTER constant (lines 75-188) contains exactly 11 names: Cam, Fab, Jackie, Jen, Kate, Kent, Lee, Malvin, Tiffani, Vee, Mark.

**Tenant filtering: NONE.** The query has no `.eq('tenant_id', ...)` filter. All user_profiles rows (minus test users and nulls) are returned regardless of tenant.

### Two separate identity tables

The codebase uses TWO tables for team identity:

1. **user_profiles** -- auth-linked, has user_id, tenant_id, roles, is_admin, focus_areas, custom_pills
2. **team_members** -- standalone, has display_name, roles, is_active. Used for action_item assignment (assigned_to FK).

The `onboard/signup` route creates rows in BOTH tables (lines 130-154). The `tenants/invites/claim` route creates a user_profiles row but does NOT create a team_members row. This is a gap.

---

## 2. Assignment Dropdowns -- Every UI That Lists Operators

### A. TeamAssignDropdown (src/components/shared/TeamAssignDropdown.tsx)

- **Used by:** EntityVetCard action tray, TriageDrawer alert cards
- **Data source:** Fetches `/api/team-members` on mount
- **Fallback:** Hardcoded array at line 16: `['Mark', 'Tiffani', 'Cam', 'Fab', 'Jackie', 'Jen', 'Kate', 'Kent', 'Lee', 'Malvin', 'Vee']`
- **Impact:** New members won't appear until getCanonicalTeamRoster() filter is fixed
- **Tenant filtering:** None

### B. EntityVetCard Assign Dropdown (src/components/ask/EntityVetCard.tsx:116)

- **Fallback names:** `['Tiffani', 'Fab', 'Cam', 'Vee']`
- Replaced at runtime by `/api/team-members` fetch, but fallback is hardcoded

### C. CallDetailPanel Operator Selector (src/components/shared/CallDetailPanel.tsx:180)

- **FULLY HARDCODED:** `const OPERATORS = ['Tiffani', 'Vee', 'Cam', 'Malvin', 'QC'] as const;`
- Does NOT fetch from API -- new members will never appear here

### D. Disputes Page Resolved-By Default (src/app/disputes/page.tsx:722)

- **Hardcoded default:** `const [resolvedBy, setResolvedBy] = useState('Fab');`
- No dropdown for other names shown in this context

### E. Pipeline Board Assignee (src/app/pipeline-board/page.tsx:825)

- **Hardcoded:** `{entity.entity_type === 'buyer' ? 'Vee' : 'Cam'}`
- No dropdown, fixed assignment by entity type

### F. KanbanBlock Assignee (src/components/shared/KanbanBlock.tsx:397)

- **Hardcoded:** `{card.entity_type === 'buyer' ? 'Vee' : 'Cam'}`
- Same pattern as Pipeline Board

### G. EntityDetailPanel Outreach Assignment (src/components/shared/EntityDetailPanel.tsx:1115)

- **Hardcoded fallback:** `owner: assignee?.display_name ?? (entityType === 'buyer' ? 'Vee' : 'Cam')`
- Tries to use team_members lookup first, falls back to hardcoded

### H. EntityDetailPanel Onboarding Task (src/components/shared/EntityDetailPanel.tsx:1302-1312)

- Searches team_members for name containing 'tiffani' -- hardcoded name search
- Falls back to `'Tiffani'` string literal

### I. Admin Impersonation Dropdown (src/app/api/admin/users/route.ts)

- Queries user_profiles directly (no TEAM_ROSTER filter)
- Filters: `is_test_user = false`, `name IS NOT NULL`
- **Tenant filtering:** NONE -- shows all users across all tenants
- **This one actually works for new members** since it doesn't filter against TEAM_ROSTER

---

## 3. User-Scoped Data -- Every Feature That Filters by User

### A. Alerts (src/app/api/alerts/route.ts, src/app/api/briefing/route.ts)

- `alerts.target_user_id` -- routes alerts to specific users
- Alert assignment: `POST /api/alerts/{id}/assign` -- takes `target_user_id` UUID
- Alert routing: `src/lib/alert-routing.ts` -- resolves names to user_ids via user_profiles
- **Tenant filtering:** NONE on alerts queries

### B. Action Items (src/app/api/action-items/route.ts)

- `assigned_to` -- FK to `team_members.id` (UUID)
- Resolution: display_name -> UUID via `team_members` table lookup
- `owner` -- string field, not FK (stores display_name directly)
- `created_by` -- string field ('milo', 'operator', etc.)
- **Tenant filtering:** NONE

### C. EOD Reports (src/app/api/eod/route.ts, src/app/api/eod/generate/route.ts)

- `user_name` -- string field, not FK
- `user_id` -- nullable UUID
- Scoped by `user_name` string match on queries
- **Tenant filtering:** NONE

### D. Shift Handoffs (src/lib/handoff-composer.ts)

- `from_user` -- string (display name)
- `to_user` -- string or null
- Looks up team_members by `display_name` ilike match (line 77-79)
- **Tenant filtering:** NONE

### E. Milo Conversations (src/app/api/milo/history/route.ts)

- Keyed by `operator` (email) + `date`
- Scoped by authenticated user's email
- **This one is inherently user-scoped via auth**

### F. Custom Pills (src/lib/milo-tools.ts:1288-1318)

- Stored in `user_profiles.custom_pills` JSONB
- Scoped by `effectiveUserId` from auth
- **This one works correctly for new members**

### G. Pill Preferences (src/app/api/profile/pill-preferences/route.ts)

- Stored in `user_profiles.pill_config`
- Scoped by `user_id` from auth
- **Works correctly for new members**

### H. Support Tickets (src/app/support/page.tsx, src/app/api/support/tickets/route.ts)

- `user_id` -- UUID from auth
- `assigned_to` -- UUID (supports operator assignment)
- **Tenant filtering on tickets:** Need to verify

### I. Notifications Activity (src/app/api/notifications/activity/route.ts, count/route.ts)

- Scoped by `target_user_id` from auth session
- **Works correctly for new members**

---

## 4. Hardcoded Users -- Every Hardcoded Operator Name/Email

### CRITICAL: Hardcoded Name Routing Tables

| File | Line(s) | Hardcoded Names | Purpose |
|------|---------|-----------------|---------|
| `lib/va-orchestrator.ts` | 80-130 | Jen, Tiffani, Fab, Malvin, Vee, Jackie, Mark | VA_PROFILES -- morning assignment routing |
| `lib/va-orchestrator.ts` | 282-824 | All 7 names via `taskMap['Name']` | Task categorization by operator |
| `lib/knowledge-engine.ts` | 194 | Fab, Malvin, Jackie, Jen, Cam, Vee, Tiffani, Mark | VALID_ROUTES set |
| `lib/knowledge-engine.ts` | 553-560 | All operator names | Routing rules in AI prompt |
| `api/notifications/draft/route.ts` | 13-39 | Fab, Jackie, Malvin, Jen, Tiffani, Mark | TEAM_ROUTING map |
| `api/cron/evaluate/route.ts` | 331-1194 | Fab, Jackie, Jen, Tiffani, Mark | Alert routing in cron |
| `api/cron/td-change-detection/route.ts` | 221-832 | Fab, Malvin | routedTo / routeTo |
| `api/cron/publisher-discovery/route.ts` | 363-382 | Fab, Jackie | Entity type routing |
| `api/cron/linkedin-draft/route.ts` | 107-268 | Tiffani | LinkedIn drafts for Tiffani only |
| `api/capture/route.ts` | 700-768 | Mark, Cam | Task owner assignment |
| `api/reconciliation/route.ts` | 50 | Jen | Reconciliation routing |
| `api/bank-transactions/review/route.ts` | 35, 153 | Jen | Bank transaction reviewer |
| `lib/outreach-engine.ts` | 247-302 | Vee | Outreach assignment |
| `lib/alert-routing.ts` | 77-83 | Jackie, Fab, Malvin | ENTITY_TYPE_DEFAULT_ROUTE |
| `lib/dispute-detector.ts` | 779 | Tiffani | Escalation target |
| `components/shared/CallDetailPanel.tsx` | 180 | Tiffani, Vee, Cam, Malvin, QC | Operator selector |
| `components/shared/TeamAssignDropdown.tsx` | 16-17 | All 11 names | Fallback dropdown |
| `components/ask/EntityVetCard.tsx` | 116 | Tiffani, Fab, Cam, Vee | Fallback assign names |
| `app/pipeline-board/page.tsx` | 825 | Vee, Cam | Hardcoded assignee |
| `components/shared/KanbanBlock.tsx` | 397 | Vee, Cam | Hardcoded assignee |
| `components/shared/EntityDetailPanel.tsx` | 1115, 1303 | Vee, Cam, Tiffani | Fallback assignees |
| `app/disputes/page.tsx` | 722 | Fab | Default resolvedBy |

### Hardcoded Emails

| File | Line | Value | Purpose |
|------|------|-------|---------|
| `app/onboard/page.tsx` | 45 | `mark@scaleflow.co`, `tiffani@scaleflow.co` | ADMIN_EMAILS gate |
| `app/api/partner/route.ts` | 109 | `'Tiffani'` | Default rep for partner onboarding |

### Hardcoded Page Routes by Name

| File | Line | Route | Purpose |
|------|------|-------|---------|
| `app/tiffani/page.tsx` | 3-6 | `/tiffani` | Tiffani-specific view |
| `components/SupportPanel.tsx` | 18 | `'tiffani': '/tiffani'` | Nav link |
| `components/JenView.tsx` | -- | `/jen` (implicit) | Jen-specific billing view |
| `lib/milo-support-prompt.ts` | 10 | `/tiffani` | Milo's knowledge of routes |

### Per-User Dashboard Blocks (src/lib/dashboard-v2-types.ts:157-168)

Hardcoded block assignments for 10 specific users: mark, tiffani, fab, malvin, jen, kate, vee, cam, lee, jackie. New users get NO custom blocks -- they fall through to role-based defaults.

---

## 5. Tenant Gaps -- Places That Don't Filter by tenant_id but Should

### CRITICAL: No tenant_id filtering anywhere in the core data path

| Surface | File | Issue |
|---------|------|-------|
| Team roster | `lib/team-roster.ts:243` | No `.eq('tenant_id', ...)` -- returns all users across tenants |
| Admin users list | `api/admin/users/route.ts:36` | No tenant_id filter -- admin sees ALL tenants' users |
| Action items | `api/action-items/route.ts:37` | No tenant_id filter on queries |
| EOD reports | `api/eod/route.ts:29` | No tenant_id filter |
| Alerts | `api/alerts/route.ts` | No tenant_id filter |
| Briefing | `api/briefing/route.ts` | No tenant_id filter |
| VA orchestrator | `lib/va-orchestrator.ts` | Hardcoded profiles, no tenant concept |
| Alert routing | `lib/alert-routing.ts` | Resolves names globally, no tenant scope |
| Knowledge engine | `lib/knowledge-engine.ts` | VALID_ROUTES is global, no tenant scope |
| Team members table | All `from('team_members')` queries | No tenant_id column apparent |
| Notifications draft | `api/notifications/draft/route.ts` | TEAM_ROUTING is global |
| Support tickets | `api/support/tickets/route.ts` | Need to verify |

**Note:** The invite-claim flow (`api/tenants/invites/claim/route.ts:97-108`) correctly sets `tenant_id` on the user_profiles upsert. The data IS there -- it's just not being used to filter downstream queries.

---

## 6. New Member Checklist -- What Must Happen When a New Team Member Joins

### Currently Working (invite flow handles these)

1. Auth user created in Supabase (via `tenants/invites/claim`)
2. `user_profiles` row created with `tenant_id`, `role`, `name` (via `tenants/invites/claim`)
3. Magic link sent for sign-in

### Currently BROKEN -- Things That Block New Member Visibility

| # | Gap | Severity | File:Line | Fix Required |
|---|-----|----------|-----------|--------------|
| 1 | **getCanonicalTeamRoster() filters against hardcoded TEAM_ROSTER** | CRITICAL | `lib/team-roster.ts:262-267` | Remove the `rosterNames` filter; use `tenant_id` filter instead |
| 2 | **No team_members row created on invite claim** | CRITICAL | `api/tenants/invites/claim/route.ts` | Add `team_members.insert()` after user_profiles upsert |
| 3 | **CallDetailPanel OPERATORS is fully hardcoded** | HIGH | `components/shared/CallDetailPanel.tsx:180` | Replace with `/api/team-members` fetch |
| 4 | **VA_PROFILES hardcoded in va-orchestrator** | HIGH | `lib/va-orchestrator.ts:80-130` | Read from team_members/user_profiles with role-based routing |
| 5 | **TEAM_ROUTING hardcoded in notifications/draft** | HIGH | `api/notifications/draft/route.ts:13-39` | Route by role, not by hardcoded name |
| 6 | **VALID_ROUTES hardcoded in knowledge-engine** | HIGH | `lib/knowledge-engine.ts:194` | Build from team_members query |
| 7 | **Knowledge engine AI prompt hardcodes routing rules** | HIGH | `lib/knowledge-engine.ts:553-560` | Build prompt routing section from roster |
| 8 | **ENTITY_TYPE_DEFAULT_ROUTE hardcoded in alert-routing** | MEDIUM | `lib/alert-routing.ts:77-83` | Map entity types to roles, resolve names at runtime |
| 9 | **USER_BLOCKS hardcoded per person** | MEDIUM | `lib/dashboard-v2-types.ts:157-168` | New members get no custom blocks (falls back to role, which works) |
| 10 | **Pipeline/Kanban assignees hardcoded** | MEDIUM | `pipeline-board/page.tsx:825`, `KanbanBlock.tsx:397` | Use team_members lookup by role |
| 11 | **Capture route hardcodes Mark/Cam as task owners** | MEDIUM | `api/capture/route.ts:700-768` | Route by role (admin for high-risk, outreach for low-risk) |
| 12 | **Cron evaluate hardcodes operator names** | MEDIUM | `api/cron/evaluate/route.ts:331-1194` | Route by role via alert-routing |
| 13 | **TD change detection hardcodes Fab/Malvin** | MEDIUM | `api/cron/td-change-detection/route.ts` | Route by role |
| 14 | **Outreach engine hardcodes Vee** | MEDIUM | `lib/outreach-engine.ts:247-302` | Route to outreach role |
| 15 | **LinkedIn drafts hardcoded for Tiffani only** | LOW | `api/cron/linkedin-draft/route.ts` | Keep as-is (Tiffani-specific feature) or make configurable |
| 16 | **ADMIN_EMAILS hardcoded on onboard page** | LOW | `app/onboard/page.tsx:45` | Use is_admin flag from user_profiles |
| 17 | **Dispute detector escalation hardcodes Tiffani** | LOW | `lib/dispute-detector.ts:779` | Route to admin/owner role |
| 18 | **No tenant_id filtering on any core query** | CRITICAL | See section 5 | Add tenant_id filter to all data queries |

### Recommended Implementation Order

**Phase 1 -- Critical (new members invisible without these):**
1. Remove TEAM_ROSTER filter from `getCanonicalTeamRoster()` -- filter by `tenant_id` instead
2. Add `team_members` row creation to `tenants/invites/claim` route
3. Add `tenant_id` filter to `getCanonicalTeamRoster()` query
4. Replace hardcoded OPERATORS in CallDetailPanel with API fetch

**Phase 2 -- High (routing broken for new members):**
5. Refactor VA_PROFILES to read from team_members with role-based routing
6. Refactor TEAM_ROUTING to use role-based lookup
7. Build VALID_ROUTES dynamically from team_members
8. Update knowledge-engine prompt to use dynamic roster

**Phase 3 -- Medium (cosmetic/fallback issues):**
9. Replace hardcoded assignees in pipeline/kanban views
10. Replace hardcoded names in capture/cron routes
11. Add tenant_id filtering to action_items, alerts, EOD, briefing queries
12. Ensure team_members table has a tenant_id column

**Phase 4 -- Tenant isolation (required before second tenant):**
13. Add tenant_id to team_members table (if not present)
14. Add tenant_id filters to ALL data queries listed in section 5
15. Scope admin/users endpoint by tenant_id
16. Scope alert-routing resolution by tenant_id

---

## Appendix: Tables Involved in User Identity

| Table | Key Columns | Used For | Tenant-Aware? |
|-------|-------------|----------|---------------|
| `auth.users` | id, email | Authentication | Yes (Supabase native) |
| `user_profiles` | user_id, name, tenant_id, roles, is_admin, focus_areas, custom_pills, pill_config | Identity, roles, preferences | Has column, not filtered |
| `team_members` | id, display_name, team, roles, is_active | Action item assignment FK | NO tenant_id column visible |
| `team_invites` | id, tenant_id, invite_code, role, status, claimed_by | Invite flow | Yes |

---

Report at: https://raw.githubusercontent.com/moscaleflow/conductor-mark-public/main/reports/USER-SCOPE-AUDIT-2026-05-07.md
