# D153 — dashboard-v2 Reference Audit

**Date:** 2026-04-22  
**Agent:** Coder-1 (self-assigned idle task)  
**Scope:** All references to `dashboard-v2` from outside `src/app/dashboard-v2/`

---

## Summary

50 references found across the codebase. 7 hard dependencies that would break the app if dashboard-v2 is removed/renamed. The route is deeply wired into admin login flow, impersonation, and setup completion — not a simple delete.

## Hard Dependencies (App Breaks Without)

### 1. Middleware admin routing — `src/middleware.ts:151-155`
```ts
const landingPath = effectiveIsAdmin ? '/dashboard-v2' : '/operator'
```
Admin login lands on `/dashboard-v2`. Without it, admins have no landing page after auth.

### 2. Middleware impersonation guard — `src/middleware.ts:157-165`
Redirects admins impersonating non-admins away from `/dashboard-v2` to `/operator`. Guards the impersonation workflow.

### 3. Setup completion redirect — `src/app/setup/page.tsx:72`
```ts
router.push('/dashboard-v2');
```
Users completing onboarding are sent to dashboard-v2. Without it, setup flow has no exit.

### 4. Topbar stop-impersonation — `src/components/Topbar.tsx:947`
```ts
window.location.href = '/dashboard-v2';
```
Hard-navigates admin back to dashboard-v2 after stopping impersonation.

### 5. ImpersonationBanner stop — `src/components/ImpersonationBanner.tsx:33`
```ts
window.location.href = '/dashboard-v2';
```
Same stop-impersonation flow from the banner component.

### 6. Operator page executive API — `src/app/operator/page.tsx:174`
```ts
fetch(`/api/dashboard-v2?${params.toString()}`)
```
Admin executive cards on `/operator` call the dashboard-v2 API. Breaking this loses the admin Top5Frame.

### 7. Walkthrough API type imports — `src/app/api/walkthrough/route.ts:3`
Imports `ROLE_BLOCKS`, `USER_BLOCKS`, `BLOCK_TITLES`, `BlockId`, `RoleName` from `@/lib/dashboard-v2-types`.

## Soft Dependencies (UX Features)

| File | Line(s) | Reference | Impact |
|---|---|---|---|
| `src/app/contract-review/[id]/page.tsx` | 618, 668, 1024 | 3 back-navigation buttons → `/dashboard-v2` | Medium — back buttons broken |
| `src/components/BackLink.tsx` | 10 | Default `href = '/dashboard-v2'` | Medium — fallback default for all BackLink usage |
| `src/components/SupportPanel.tsx` | 15 | `'dashboard': '/dashboard-v2'` in PAGE_MAP | Low — link generation only |

## Component Imports (From dashboard-v2 Folder)

These import reusable components that live inside `src/app/dashboard-v2/`:

| File | Import |
|---|---|
| `src/app/pipeline/page.tsx` | `KanbanBlock`, `EntityDetailPanel` |
| `src/app/pipeline-board/page.tsx` | `EntityDetailPanel` |
| `src/components/ConversationModal.tsx` | `EntityDetailPanel`, `ContractAnalysisDisplay` |
| `src/components/OnboardingTour.tsx` | `BlockId` type |

**Note:** These are component/type imports from files that happen to live in the dashboard-v2 folder. They depend on the folder existing, not the route being accessible. Safe to move to `src/components/shared/` during decommission.

## Type/Config Dependencies

| File | Import |
|---|---|
| `src/app/api/walkthrough/route.ts` | `ROLE_BLOCKS`, `USER_BLOCKS`, `BLOCK_TITLES`, `BlockId`, `RoleName` from `@/lib/dashboard-v2-types` |
| `src/lib/operator-pills.ts` | `RoleName` type, comments about `ROLE_ALIASES` sync |

## Test References

| File | Lines | Purpose |
|---|---|---|
| `tests/role-qa.spec.ts` | 316, 345 | Asserts admin routes to dashboard-v2 |
| `scripts/e2e/auth-flow-2026-04-21.ts` | 92, 99, 102, 164 | Auth flow assertions |
| `tests/screenshots/d146-api-*.json` | various | Hardcoded test result URLs |

## Decommission Checklist

If/when dashboard-v2 is replaced (per D152 shell refinement):

1. **Middleware** — change admin landing path from `/dashboard-v2` to new route
2. **Setup page** — update completion redirect
3. **Topbar + ImpersonationBanner** — update stop-impersonation destination
4. **Operator page** — keep `/api/dashboard-v2` endpoint OR migrate executive view data to new API
5. **BackLink default** — change default href
6. **Contract review** — update 3 back buttons
7. **Component relocations** — move `EntityDetailPanel`, `KanbanBlock`, `ContractAnalysisDisplay` out of dashboard-v2 folder
8. **Type relocations** — move `dashboard-v2-types.ts` to neutral location
9. **Tests** — update route assertions

**Estimated touch points:** 9 files, ~15 lines of routing code, plus component/type file moves.

---

Report at: https://raw.githubusercontent.com/moscaleflow/conductor-mark-public/main/reports/2026-04-22-d153-dashboard-v2-reference-audit.md
