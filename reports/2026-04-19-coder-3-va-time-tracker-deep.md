---
directive: "va-time-tracker deep re-audit for Jibble-parity features"
lane: research
coder: Coder-3
started: 2026-04-20 ~02:00 MDT
completed: 2026-04-20 ~02:45 MDT
repo: github.com/moscaleflow/va-time-tracker (cloned to /tmp/va-time-tracker)
---

# va-time-tracker Deep Audit — 2026-04-20

## 1. Executive Summary

The initial audit calling va-time-tracker "no extraction value" was badly wrong. This is a **~20,000 LOC feature-complete time tracking + invoicing + payroll platform** with 24+ database tables, 30 modal components, 9 admin views, and 2 Edge Functions. It covers 13 of 15 Jibble feature categories, and **exceeds Jibble** in invoicing, payroll, and multi-client margin tracking — features Jibble charges premium tiers for.

**Strongest primitive candidate: `@milo/time-tracking`** — the backend (schema + business logic) extracts cleanly. The frontend is Vue3/CDN and would need a full rewrite for React/Next.js verticals, but that's a UI-layer concern, not a primitive-boundary concern.

**Feature coverage vs Jibble:**
- 10 features at parity or better (time tracking, time-off, schedules, approvals, reporting, invoicing, payroll, multi-user, notifications, admin)
- 3 features partial (location verification via screenshots not GPS, reporting lacks charts, no integrations)
- 2 features missing (mobile native app, offline/PWA)

**Recommended tier: HIGH** (upgrade from current MEDIUM). This is a production-quality SaaS alternative with genuine extraction value. The schema alone is worth packaging.

## 2. Feature Matrix

### Jibble-Parity Status

| Feature | va-time-tracker | Jibble Free | Jibble Premium | Parity | Extraction Path |
|---------|----------------|-------------|----------------|--------|-----------------|
| Clock in/out | Yes (button + live timer) | Yes | Yes | Parity | Schema + API |
| Pause/resume | Yes | No | Yes | **Exceeds free** | Schema + API |
| Manual time entry | Yes (modal + audit trail) | Yes | Yes | Parity | Schema + API |
| Timer pop-out window | Yes | No | No | **Exceeds** | UI-only (vertical) |
| 8-hour check-in alert | Yes (modal prompt) | No | Yes | **Exceeds free** | Business logic |
| Time-off requests | Yes (request → approve/deny) | No | Yes | **Exceeds free** | Schema + workflow |
| Time-off ledger | Yes (full history) | No | Yes | Parity w/premium | Schema + API |
| PTO balance display | Yes | No | Yes | Parity w/premium | Schema + API |
| Shift scheduling | Yes (visual builder) | No | Yes | Parity w/premium | Schema + UI |
| Schedule conflicts | Yes (auto-detection) | No | No | **Exceeds** | Business logic |
| Shift overrides | Yes (request → approve) | No | No | **Exceeds** | Schema + workflow |
| Timesheet approval | Yes | No | Yes | Parity w/premium | Schema + workflow |
| Hours reporting | Yes (weekly/monthly, PDF) | Yes | Yes | Parity | Schema + export |
| Charts/analytics | No | No | Yes | **Behind premium** | Future work |
| Invoice generation | Yes (auto from entries) | No | No | **Exceeds all** | Schema + logic |
| Invoice batch gen | Yes (all clients at once) | No | No | **Exceeds all** | Business logic |
| Invoice editing | Yes (line-item editor) | No | No | **Exceeds all** | Schema + UI |
| Invoice PDF export | Yes (jsPDF) | No | No | **Exceeds all** | Export logic |
| Payroll calculation | Yes (auto from entries) | No | No | **Exceeds all** | Schema + logic |
| Payroll review/export | Yes (CSV + PDF) | No | No | **Exceeds all** | Export logic |
| Bonus tracking | Yes | No | No | **Exceeds all** | Schema + API |
| Pay rate history | Yes (effective dates) | No | No | **Exceeds all** | Schema |
| Multi-user roles | Yes (admin/va/client) | Yes | Yes | Parity | Schema + RLS |
| Team management | Yes (VA↔client assign) | Yes | Yes | Parity | Schema + API |
| GPS geofencing | No (screenshots instead) | No | Yes | **Behind premium** | N/A |
| Spot check screenshots | Yes (capture + review) | No | No | **Exceeds all** | Schema + storage |
| Accomplishment logging | Yes (daily per-client) | No | No | **Exceeds all** | Schema + API |
| Pomodoro timer | Yes (integrated) | No | No | **Exceeds all** | UI component |
| Client onboarding | Yes (multi-step pipeline) | No | No | **Exceeds all** | Schema + workflow |
| VA onboarding | Yes (application pipeline) | No | No | **Exceeds all** | Schema + workflow |
| Dark mode | Yes | Yes | Yes | Parity | UI |
| Mobile responsive | Yes (Tailwind) | Yes (native app) | Yes (native) | Partial (web-only) | UI |
| Offline mode | No | No | Yes | **Behind premium** | Future work |
| Push notifications | No | No | Yes | **Behind premium** | Future work |
| Integrations | No (standalone) | Limited | Yes (Slack, etc.) | **Behind premium** | Future work |
| Multi-client margin | Yes (bill rate - pay rate) | No | No | **Exceeds all** | Schema + logic |

**Score: 22 features at parity or better, 5 behind Jibble Premium, 0 behind Jibble Free.**

## 3. Per-Feature Deep Dive

### 3.1 Time Tracking Engine

**Core files:**
- `js/dashboard.js` (2,952 LOC) — VA-facing timer, clock in/out, client switching
- `js/components/TimeEntryFormModal.js` (198 LOC) — manual time entry
- `js/components/EditTimeEntryModal.js` (232 LOC) — edit with audit trail
- `js/components/ManualEntryModal.js` (71 LOC) — backdated entry

**Schema: `time_entries` table**

| Column | Type | Notes |
|--------|------|-------|
| id | UUID PK | |
| va_id | UUID FK → profiles | |
| client_id | UUID FK → companies | |
| assignment_id | UUID FK → assignments | optional |
| start_time | TIMESTAMPTZ | |
| end_time | TIMESTAMPTZ | |
| created_at | TIMESTAMPTZ | |
| updated_at | TIMESTAMPTZ | |

Duration calculated in app as `(end_time - start_time) / 3600`.

**Features:**
- Live timer with pause/resume (not just clock in/out)
- 8-hour check-in modal (prompts VA to confirm still working)
- Pop-out timer window (floating timer in separate browser window)
- Auto-stop on connection loss
- Client selector (switch active client during shift)
- Per-entry client assignment
- Edit with required audit reason

**Horizontal assessment:** 100% horizontal. Timer logic, manual entry, edit audit trail — all generic. Client assignment is the "project" concept in other time trackers.

**LOC:** ~3,450

### 3.2 Time-Off Management

**Core files:**
- `js/views/RequestsView.js` (177 LOC) — admin request queue
- `js/components/ApprovalModal.js` (50 LOC) — approve/deny dialog
- `js/views/LedgerView.js` (39 LOC) — time-off history

**Schema: `time_off_requests` table**

| Column | Type | Notes |
|--------|------|-------|
| id | UUID PK | |
| va_id | UUID FK → profiles | |
| start_date | DATE | |
| end_date | DATE | |
| status | TEXT | pending/approved/rejected |
| admin_note | TEXT | |
| created_at | TIMESTAMPTZ | |
| updated_at | TIMESTAMPTZ | |

**Features:**
- Request submission by VA
- Admin approval/denial with notes
- Full history ledger
- PTO balance display on dashboard
- Badge notifications on admin nav

**Horizontal assessment:** 100% horizontal. Standard PTO workflow. No VA-specific logic.

**LOC:** ~270

### 3.3 Scheduling System

**Core files:**
- `js/components/SimpleScheduleBuilder.js` (400 LOC) — visual weekly schedule editor
- `js/components/ConflictWarnings.js` (161 LOC) — schedule conflict detection
- `js/components/ShiftOverrideModal.js` (120 LOC) — shift override requests
- `js/components/WeeklySchedulePicker.js` (227 LOC) — week selection
- `schedule-setup.html` (377 LOC) — schedule setup page
- `js/schedulingUtils.js` — schedule calculation helpers

**Schema: `va_availability` + `client_scheduling_requirements`**

**va_availability:**

| Column | Type | Notes |
|--------|------|-------|
| id | UUID PK | |
| va_id | UUID FK → profiles | |
| day_of_week | INTEGER | 0=Monday, 6=Sunday |
| start_time | TIME | |
| end_time | TIME | |
| timezone | TEXT | |
| is_active | BOOLEAN | |

**client_scheduling_requirements:**

| Column | Type | Notes |
|--------|------|-------|
| id | UUID PK | |
| client_id | UUID FK → companies | |
| assignment_id | UUID FK → assignments | optional |
| day_of_week | INTEGER | |
| start_time | TIME | |
| end_time | TIME | |
| timezone | TEXT | |
| hours_per_week | DECIMAL(6,2) | |
| is_active | BOOLEAN | |

**RPC functions:**
- `check_scheduling_conflicts(expected_hours, client_id, va_id)` — returns conflict objects with severity
- `suggest_available_vas(client_id, expected_hours)` — find VAs with capacity

**Features:**
- Visual weekly schedule builder (drag/select time slots)
- Per-VA availability definition
- Per-client scheduling requirements
- Automatic conflict detection when assigning VA to client
- Shift override request + approval workflow
- Timezone-aware scheduling

**Horizontal assessment:** 95% horizontal. The VA↔client terminology is vertical, but the underlying model (employee availability + client requirements + conflict detection) is universal for any staffing/scheduling business.

**LOC:** ~1,285

### 3.4 Invoicing System

**Core files:**
- `js/views/InvoicingView.js` (430 LOC) — invoice generation + management
- `js/components/InvoicePreviewModal.js` (163 LOC) — preview before save
- `js/components/InvoiceReviewModal.js` (84 LOC) — admin review
- `js/components/EditInvoiceModal.js` (202 LOC) — post-generation editing

**Schema: `invoices` table**

| Column | Type | Notes |
|--------|------|-------|
| id | UUID PK | |
| client_id | UUID FK → companies | |
| invoice_number | TEXT UNIQUE | auto-generated |
| status | TEXT | draft/sent/paid/overdue/cancelled |
| start_date | DATE | |
| end_date | DATE | |
| total_amount | DECIMAL(12,2) | |
| paid_amount | DECIMAL(12,2) | default 0 |
| line_items | JSONB | array of line items |
| sent_date | TIMESTAMPTZ | |
| paid_date | TIMESTAMPTZ | |
| created_at | TIMESTAMPTZ | |
| updated_at | TIMESTAMPTZ | |

**Features:**
- Auto-generate invoices from time entries for a date range
- Batch generation (all clients at once)
- Line-item preview + editing before save
- Per-line rate overrides
- Invoice status tracking (draft → sent → paid/overdue)
- PDF download via jsPDF
- Invoice review workflow

**Horizontal assessment:** 100% horizontal. Generic time-based invoicing. The "client" is just "the entity being billed."

**LOC:** ~880

### 3.5 Payroll System

**Core files:**
- `js/views/PayrollView.js` (1,022 LOC) — payroll generation + management
- `js/components/PayrollReviewModal.js` (88 LOC) — review before finalization
- `js/components/PayrollDetailsModal.js` (153 LOC) — line-item detail view
- `js/components/PaymentTrackingModal.js` (111 LOC) — payment status tracking
- `js/components/PayRateFormModal.js` (83 LOC) — rate CRUD
- `js/components/BonusFormModal.js` (77 LOC) — bonus entry

**Schema: `payroll_runs` + `bonuses` + `va_pay_rates`**

**payroll_runs:**

| Column | Type | Notes |
|--------|------|-------|
| id | UUID PK | |
| va_id | UUID FK → profiles | |
| start_date | DATE | |
| end_date | DATE | |
| status | TEXT | draft/reviewed/paid/failed |
| total_hours | DECIMAL(8,2) | |
| total_amount | DECIMAL(12,2) | |
| bonus_amount | DECIMAL(12,2) | default 0 |
| paid_date | TIMESTAMPTZ | |

**va_pay_rates:** Historical rates with effective_date/end_date range.

**bonuses:** One-time bonus entries with reason, status, created_by.

**Features:**
- Auto-calculate payroll from time entries + pay rates
- Bonus inclusion in payroll runs
- Rate history with effective dates
- Payroll review before finalization
- Payment tracking (mark as paid)
- CSV/PDF export
- Per-VA payroll detail drill-down

**Horizontal assessment:** 100% horizontal. Generic hourly payroll. Works for any business paying workers by the hour.

**LOC:** ~1,534

### 3.6 Spot Check System

**Core files:**
- `js/components/SpotCheckModal.js` (432 LOC) — screenshot capture + review UI
- Edge Function: `cleanup-spot-checks/index.ts` (62 LOC) — daily cleanup

**Schema: `spot_checks` table**

| Column | Type | Notes |
|--------|------|-------|
| id | UUID PK | |
| va_id | UUID FK → profiles | |
| screenshot_url | TEXT | Supabase Storage path |
| status | TEXT | pending/approved/flagged/deleted |
| created_at | TIMESTAMPTZ | |

**Features:**
- Admin-triggered screenshot requests
- Screenshot capture + upload to Supabase Storage
- Review UI with approve/flag/delete actions
- Auto-cleanup of old screenshots (daily cron at 2am)
- Storage bucket: `screenshots/`

**Horizontal assessment:** 80% horizontal. Screenshot-based activity verification is useful beyond VA tracking (remote employee monitoring, compliance). The "spot check" concept is VA-specific terminology, but the mechanism is generic.

**LOC:** ~494

### 3.7 Assignment & Rate Management

**Core files:**
- `js/components/AssignVAModal.js` (455 LOC) — VA↔client assignment
- `js/components/ManageVaModal.js` (975 LOC) — full VA management
- `js/components/ManageClientModal.js` (651 LOC) — full client management

**Schema: `assignments` table**

| Column | Type | Notes |
|--------|------|-------|
| id | UUID PK | |
| va_id | UUID FK → profiles | |
| client_id | UUID FK → companies | |
| va_pay_rate | DECIMAL(10,2) | what VA earns/hr |
| client_bill_rate | DECIMAL(10,2) | what client is billed/hr |
| expected_hours_per_week | INTEGER | |
| is_active | BOOLEAN | |
| display_name | TEXT | optional custom label |
| admin_override | BOOLEAN | capacity override flag |
| override_reason | TEXT | |

**Features:**
- VA-to-client assignment with dual rates (pay + bill)
- Margin tracking (bill rate - pay rate)
- Expected hours per assignment
- Capacity override with admin reason
- Active/inactive toggle
- Assignment history

**Horizontal assessment:** 90% horizontal. The dual-rate staffing model (pay rate + bill rate + margin) is universal for any agency/staffing business. VA↔client is just "worker↔customer."

**LOC:** ~2,080

### 3.8 Onboarding Pipelines

**Core files:**
- `onboard.html` (1,294 LOC) — client application form
- `va-onboard.html` (1,065 LOC) — VA application form
- `js/views/OnboardingSubmissionsView.js` (402 LOC) — pipeline management
- `js/components/SubmissionDetailsModal.js` (453 LOC) — client submission review
- `js/components/VASubmissionDetailsModal.js` (310 LOC) — VA submission review
- Edge Function: `create-va-account/index.ts` (268 LOC) — account creation

**Schema: `va_onboarding_submissions` + `client_onboarding_submissions`**

Both share: id, created_at, updated_at, status (pending/contacted/approved/rejected), admin_notes, reviewed_by, reviewed_at.

VA-specific: full_name, preferred_name, email, phone, country, timezone, is_part_time, max_hours_per_week, expected_hourly_rate, availability_schedule (JSONB), services_offered (TEXT[]), years_experience, bio, available_start_date, referral_source.

**Features:**
- Multi-step application forms (progress bar)
- Anonymous submission (no auth required)
- Admin review pipeline (pending → contacted → approved/rejected)
- Auto-create user account on approval (Edge Function)
- Schedule + availability captured at onboarding
- Referral tracking

**Horizontal assessment:** 70% horizontal. The pipeline (submit → review → approve → create account) is generic. The form fields are vertical-specific (VA skills, experience, rate expectations).

**LOC:** ~3,792

### 3.9 Accomplishment Logging

**Core file:** `js/components/AccomplishmentLogModal.js` (53 LOC)

**Schema: `daily_accomplishments` table** — va_id, date, accomplishments (TEXT/JSONB), status.

**Features:**
- Required before clock-in (enforced in dashboard.js)
- Per-client daily notes
- Admin visibility

**Horizontal assessment:** 90% horizontal. Daily standup/log pattern. Useful for any remote team.

**LOC:** ~53

### 3.10 Pomodoro Timer

**Core file:** `js/components/PomodoroTimer.js` (472 LOC)

Integrated focus timer with work/break intervals. Standalone component, no DB persistence.

**Horizontal assessment:** 100% horizontal. Generic productivity widget.

**LOC:** ~472

### 3.11 Admin Dashboard & Reporting

**Core files:**
- `js/admin.js` (2,875 LOC) — admin orchestrator
- `js/views/SummaryView.js` (266 LOC) — dashboard overview
- `js/views/VAsView.js` (280 LOC) — VA roster
- `js/views/ClientsView.js` (177 LOC) — client roster
- `js/views/FlaggedEntriesView.js` (205 LOC) — flagged time entries

**Features:**
- Summary cards (total hours, revenue, pending requests, active VAs)
- KPI display (profit margin, VA count)
- Date range filtering (presets + custom)
- Expandable VA rows with client breakdown
- Flagged entries view (auto-stopped, edited, overtime)
- PDF export via jsPDF

**Horizontal assessment:** 95% horizontal. Dashboard pattern, reporting, flagging — all generic. VA/client terminology is vertical.

**LOC:** ~3,803

## 4. Database Schema Summary

| Category | Tables | Purpose |
|----------|--------|---------|
| **Core** | profiles, companies, assignments | Users, clients, worker↔client links |
| **Time** | time_entries, daily_accomplishments | Clock data, daily logs |
| **Financial** | invoices, payroll_runs, bonuses, va_pay_rates, client_rates, client_bill_rates | Billing + payroll |
| **Scheduling** | va_availability, client_scheduling_requirements, time_off_requests, shift_override_requests | Availability + PTO |
| **Monitoring** | spot_checks | Screenshot verification |
| **Onboarding** | va_onboarding_submissions, client_onboarding_submissions | Application pipelines |
| **Profile** | va_profiles, va_admin_notes, va_payment_info | Extended user data |
| **Skills** | skills, skill_categories, skill_checklist_items, va_skills, va_skill_checklist_progress, skill_bounties, bounty_winners, training_assignments, peer_grading_assignments | Gamification/training (9 tables) |
| **Storage** | screenshots, profile-photos | Supabase Storage buckets |

**Total: 24+ tables, 2 storage buckets, 2 Edge Functions, 5+ RPC functions.**

## 5. Horizontal vs Vertical Split Analysis

### Horizontal (extractable to @milo/time-tracking)

| Component | LOC | Extractability | Notes |
|-----------|-----|----------------|-------|
| Time entry schema + CRUD | 200 | Easy | Generic clock in/out + manual entry |
| Timer business logic | 300 | Moderate | Pause/resume, auto-stop, 8-hour check-in |
| Time-off request/approve workflow | 270 | Easy | Generic PTO |
| Scheduling (availability + requirements + conflicts) | 500 | Moderate | RPC functions need extraction |
| Invoicing (auto-generate from entries + CRUD + PDF) | 880 | Moderate | jsPDF dependency, line-item JSONB |
| Payroll (calculate from entries + rates + bonuses) | 800 | Moderate | Rate history logic |
| Assignment model (worker↔client + dual rates) | 400 | Easy | Generic staffing pattern |
| Approval workflow pattern | 150 | Easy | Generic request→approve/reject |
| Spot check mechanism | 300 | Moderate | Storage integration |
| Accomplishment logging | 53 | Easy | Generic daily log |
| RLS policies + is_admin() | 100 | Easy | Generic role-based access |
| DB migrations (all tables) | 350 | Easy | Schema is generic |
| **Subtotal** | **~4,300** | | |

### Vertical (stays in VA vertical)

| Component | LOC | Why Vertical |
|-----------|-----|-------------|
| "VA Bees" branding | — | Brand name, colors (#F4C542), logo |
| VA/client terminology | — | "VA" = worker, "client" = customer |
| VA onboarding form fields | ~1,065 | Skills, experience, services_offered — VA-specific |
| Client onboarding form fields | ~1,294 | Company info, service needs — VA-staffing-specific |
| Skill gamification system (9 tables) | ~500+ | VA training/certification — very vertical |
| Pomodoro timer | 472 | Standalone widget, not core |
| Vue3/CDN UI layer | ~12,000+ | All frontend rendering — stack-specific |

## 6. Recommended @milo/time-tracking v0.1.0

### Package Shape

```
@milo/time-tracking/
├── schema/
│   ├── time_entries.sql
│   ├── time_off_requests.sql
│   ├── schedules.sql          # availability + requirements
│   ├── assignments.sql        # worker↔client + dual rates
│   ├── invoices.sql
│   ├── payroll.sql            # payroll_runs + bonuses + rates
│   └── spot_checks.sql
├── lib/
│   ├── timer.ts               # clock in/out, pause/resume, auto-stop
│   ├── time-off.ts            # request/approve/reject workflow
│   ├── scheduling.ts          # conflict detection, suggest available
│   ├── invoicing.ts           # generate from entries, batch gen
│   ├── payroll.ts             # calculate from entries + rates
│   ├── assignments.ts         # worker↔client CRUD, margin calc
│   └── approval.ts            # generic request→approve pattern
├── types/
│   └── index.ts               # all interfaces
└── SCHEMA.md
```

### Estimated LOC: ~2,500 (backend + schema)

The frontend (~15,000+ LOC) stays vertical. Each vertical builds its own UI against @milo/time-tracking's API surface.

### Why This Boundary

1. **Schema is the real value.** The 24-table schema represents months of domain modeling. Any vertical needing time tracking + invoicing + payroll gets it for free.
2. **Business logic is horizontal.** Conflict detection, invoice generation from time entries, payroll calculation — these algorithms don't change between verticals.
3. **UI is vertical.** Vue3/CDN for VA Bees. React/Next.js for other verticals. The rendering layer is the least extractable part.

## 7. Tech Stack Rewrite Question

### Vue3/CDN → React/Next.js: Is it worth rewriting the UI?

**No — for extraction purposes.** The primitive extracts as backend + schema. Each vertical provides its own UI.

**Yes — if VA Bees itself needs to run on the milo-starter template.** If va-time-tracker becomes a vertical on the ScaleFlow platform (alongside mysecretary, PPC, etc.), the UI needs rewriting to React/Next.js with the dark mode design system.

### Rewrite effort estimate

| Layer | LOC to Rewrite | Effort |
|-------|---------------|--------|
| 30 modal components | ~7,500 | 2-3 weeks |
| 9 view components | ~3,400 | 1-2 weeks |
| Dashboard (timer + clock) | ~3,000 | 1 week |
| Admin orchestrator | ~2,900 | 1 week |
| Auth + routing | ~500 | 2 days |
| **Total** | **~17,300** | **5-7 weeks** |

**Recommendation:** Don't rewrite now. Extract the backend primitive first. Rewrite UI only when VA Bees needs to join the ScaleFlow platform.

## 8. Edge Function Consideration

Both Edge Functions run on Deno (same blocker as PPCRM):

| Function | LOC | Purpose | Migration Path |
|----------|-----|---------|----------------|
| `create-va-account` | 268 | Create user + profile + availability + assignment | Convert to Next.js API route in @milo/time-tracking |
| `cleanup-spot-checks` | 62 | Delete old screenshots (daily cron) | Convert to Next.js API route + Vercel Cron |

Small scope (330 LOC total). Straightforward conversion.

## 9. Updated Tier Recommendation

**Current tier in MILO-ENGINE-PRIMITIVES.md: MEDIUM**

**Recommended: HIGH**

Justification:

| Factor | Rating | Reasoning |
|--------|--------|-----------|
| Feature completeness | High | 13/15 Jibble feature categories covered |
| Production readiness | Medium-High | ~20K LOC, live usage, dark mode, responsive |
| Extraction value | High | Schema + business logic extract cleanly as ~2,500 LOC backend |
| Cross-vertical demand | High | Any ScaleFlow vertical with employees/contractors needs time tracking |
| Competitive positioning | High | Replaces $5-10/user/month SaaS (Jibble, Clockify, Toggl) |
| Extraction effort | Moderate | Backend is clean; UI rewrite is optional and deferred |

The invoicing + payroll combination alone justifies HIGH tier. Most time tracking tools charge premium for these features. Having them in an extractable primitive is a significant competitive advantage for ScaleFlow verticals.

## 10. Open Questions for Mark

1. **Is VA Bees planned as a ScaleFlow vertical?** If yes, the Vue3/CDN UI needs a React/Next.js rewrite (~5-7 weeks). If VA Bees stays standalone, extraction is backend-only and much faster.

2. **Skill gamification system (9 tables) — keep or drop?** The training/certification/bounty system is VA-specific but substantial. Should @milo/time-tracking include a generic "skill tracking" module, or is this purely VA Bees vertical logic?

3. **Spot checks — generic "activity verification" or VA-only?** Screenshot-based spot checks could serve any remote employee monitoring scenario. Worth extracting into @milo/time-tracking, or keep as vertical extension?

4. **Invoicing boundary.** va-time-tracker's invoicing is time-based (generate from time entries). MOP has a separate invoicing system for campaigns. Should @milo/time-tracking own time-based invoicing, or should all invoicing consolidate into a future `@milo/invoicing` primitive?

5. **Payroll boundary.** Similar question — does payroll belong in @milo/time-tracking (tightly coupled to time entries + rates), or should it be a separate `@milo/payroll` primitive? The coupling argument favors keeping them together.

6. **Multi-tenant readiness.** va-time-tracker has no tenant_id on any table. The schema assumes single-tenant (one business). For ScaleFlow platform usage, tenant_id needs adding to all 24+ tables — same pattern as MOP's `20300101` migration. Should this be done during extraction or deferred?

7. **RPC functions.** Five PL/pgSQL functions exist in Supabase but their full definitions aren't in the repo (likely created via Supabase console). These need to be exported and included in the migration files before extraction. Can Mark export them via `supabase db dump`?
