# D181 Report: Post-D180 Verification + Dashboard-v2 Component Cleanup

**Agent:** Coder-2 | **Date:** 2026-04-22 | **Status:** Complete (build clean, Playwright verified, screenshots on Desktop)

---

## Item 1: D175 Playwright Re-run (Post-D180)

### Context

D175 (contract inline clause expansion) was blocked by a missing `user_profiles.hidden_pills` column. D180 (Coder-1) shipped the migration in `987b705` and committed D175's code changes in `66e4638`.

### Verification

| Check | Result |
|-------|--------|
| Migration applied to Supabase | Yes (`supabase db push --dry-run` → "Remote database is up to date") |
| `hidden_pills` column exists | Yes (confirmed via `user_profiles` SELECT) |
| Dev server restart required | Yes (stale `.next` cache caused `ERR_ABORTED` on all routes) |
| Test user role fix | Changed from `publisher_relations` → `operations` + added `contracts` custom pill (admin role uses executive Top5Frame, not PillDrawer) |
| Playwright spec passes | Yes (1 passed, 19.9s) |

### Screenshots on Desktop

| File | Content |
|------|---------|
| `d175-contract-expansion.png` | BlueCross Partners expanded — 3 clauses: CRITICAL (Section 8.2a), HIGH (Section 3.1), MEDIUM (Section 12.4) |
| `d175-empty-state.png` | TrueConnect Media expanded — "No analysis yet — run Milo on this contract." |

### Test Fixes Applied

- `waitUntil: 'networkidle'` → `'commit'` (Next.js 16 middleware causes `ERR_ABORTED` with networkidle)
- Locator unchanged: `text=Contracts` excluding `/review/i` (correct for PillDrawer pill strip)
- Mobile viewport (375x812) screenshot also captured in `tests/screenshots/`

---

## Item 2: Dashboard-v2 Component Relocation (D164b Item 01)

### Summary

Deleted 29 dead components from `src/components/dashboard-v2/`. One component (AgreedTermsForm) had an external consumer — moved to `src/components/shared/` first. Removed empty directories.

### Changes

**Moved (1 file):**
- `src/components/dashboard-v2/AgreedTermsForm.tsx` → `src/components/shared/AgreedTermsForm.tsx` (674 LOC)
- Updated import in `src/components/shared/EntityDetailPanel.tsx`

**Deleted (28 files, ~7,144 LOC):**
AlertsBlock, ApAgingBlock, BankTransactionsBlock, BillingPipelineBlock, BuyerMonitoringBlock, CampaignHealthBlock, CommissionTrackerBlock, CreativeReviewBlock, DIDTrackerBlock, DashboardBlock, DateRangePicker, DisputeRecoveryBlock, FlaggedCallsBlock, FollowupsBlock, InvoiceBlock, LinkedInActivityBlock, MediaRiteXRefBlock, OutreachStatsBlock, PingBlock, PipelineBlock, PublisherMonitoringBlock, PulseBlock, QualityBlock, SupportTicketsBlock, TasksBlock, TeamTimeBlock, TopPublishersBlock, WeeklyBillingBlock

**Preserved:**
- `src/lib/dashboard-v2-types.ts` — actively imported by 5 files (walkthrough/route.ts, operator/executive/route.ts, OnboardingTour.tsx, PillPicker.tsx, operator-pills.ts)
- 3 components already relocated in D167: EntityDetailPanel, KanbanBlock, ContractAnalysisDisplay

**Directories removed:**
- `src/components/dashboard-v2/` (empty after deletions)
- `src/app/dashboard-v2/` (already empty — page.tsx deleted in D159)

### Verification

| Test | Result |
|------|--------|
| `npm run build` | Clean (0 errors, 0 warnings) |
| No remaining `@/components/dashboard-v2/` imports | Confirmed via grep |
| `dashboard-v2-types.ts` consumers intact | 5 files verified |

### LOC Impact

| Category | LOC |
|----------|-----|
| Deleted (28 components) | -7,144 |
| Moved (AgreedTermsForm) | 0 (relocated, not deleted) |
| Import update | 1 line changed |
| **Net deletion** | **-7,144** |

(Spec estimated -12k LOC — difference is because 3 large components were already relocated in D167, and `dashboard-v2-types.ts` was preserved.)

---

## Git Status

All changes are uncommitted. `git diff --stat` shows 33 files changed, 7,818 deletions net. D175 code changes were committed by Coder-1 in `66e4638`; only the Playwright spec, seed additions, and D181 deletions remain uncommitted.
