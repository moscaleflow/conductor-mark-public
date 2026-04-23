# D159 — Decommission /dashboard-v2

**Date:** 2026-04-22  
**Agent:** Coder-1  
**Scope:** Delete /dashboard-v2 page, redirect all references to /operator

---

## Summary

/dashboard-v2 page deleted (-2,237 LOC). All 7 hard dependencies and 3 soft dependencies from D153 audit updated. 301 redirect added via next.config.ts. Build passes clean. Net: -2,214 LOC across 26 files.

## Changes Made

### Route references updated (10 files)

| File | Change |
|---|---|
| src/middleware.ts | Simplified: all users → /operator. Removed impersonation guard for /dashboard-v2 (-10 lines) |
| src/components/BackLink.tsx | Default href: /dashboard-v2 → /operator |
| src/components/SupportPanel.tsx | PAGE_MAP: /dashboard-v2 → /operator |
| src/components/ImpersonationBanner.tsx | Stop-impersonation: /dashboard-v2 → /operator |
| src/components/Topbar.tsx | Stop-impersonation: /dashboard-v2 → /operator. Updated 3 comments. |
| src/app/contract-review/[id]/page.tsx | 3x router.push: /dashboard-v2 → /operator |
| src/app/setup/page.tsx | Completion redirect: /dashboard-v2 → /operator |
| src/components/admin/AdminSkillBar.tsx | Removed /dashboard-v2 pathname check (-2 lines) |
| src/lib/team-roster.ts | Updated comment |
| next.config.ts | Added redirects(): /dashboard-v2 → /operator (301, permanent) |

### Page deleted (1 file)

| File | LOC |
|---|---|
| src/app/dashboard-v2/page.tsx | -2,237 |

### Comment-only updates (14 files)

| File | Change |
|---|---|
| src/app/operator/page.tsx | Updated 4 comments removing /dashboard-v2 references |
| src/components/AdminExecutiveCards.tsx | Updated component docstring |
| src/components/OperatorTour.tsx | Updated component docstring |
| src/app/api/briefing/route.ts | Updated inline comment |
| src/app/observatory/page.tsx | Updated admin gate comment |
| src/app/api/walkthrough/route.ts | Updated auth helper comment |
| src/app/api/profile/complete-operator-tour/route.ts | Updated docstring |
| src/lib/impersonation-state.ts | Updated consumer list in docstring |
| src/lib/greetings.ts | Updated docstring |
| src/lib/executive-ranker.ts | Updated docstring |
| src/lib/operator-pills.ts | Updated admin pills comment |
| src/components/admin/AddSkillModal.tsx | Updated docstring |

### Test updates (2 files)

| File | Change |
|---|---|
| tests/role-qa.spec.ts | Lines 316, 345: admin route assertions → /operator |
| scripts/e2e/auth-flow-2026-04-21.ts | Lines 92, 99, 102, 164: removed /dashboard-v2 from URL patterns |

### Files kept per spec (not deleted)

| File | Reason |
|---|---|
| src/app/api/dashboard-v2/route.ts (2,066 LOC) | /operator fetches this for admin executive data. Rename to /api/operator/executive deferred to v4.1. |
| src/components/dashboard-v2/*.tsx (32 files) | pipeline, pipeline-board, ConversationModal import EntityDetailPanel/KanbanBlock. Relocation deferred to v4.1. |
| src/lib/dashboard-v2-types.ts (412 LOC) | RoleName type imported by operator-pills.ts. Rename deferred to v4.1. |

## Verification

- **npm run build**: PASS — compiled successfully, /dashboard-v2 absent from route list
- **Redirect rule**: next.config.ts `redirects()` returns 301 for /dashboard-v2 → /operator
- **Current production baseline**: `GET /dashboard-v2` returns 307→/login (unauthenticated). Post-deploy: next.config redirect fires first, returning 301→/operator before middleware runs.

### Post-deploy checks (cannot run until deployed)

- Admin login lands on /operator (not /dashboard-v2)
- GET /dashboard-v2 returns 301 → /operator
- "Back" links from contract-review navigate to /operator
- Stop-impersonation navigates to /operator
- Vercel analytics for /dashboard-v2 traffic — check 24h post-deploy

## Net Impact

- **-2,214 LOC** (2,237 deleted, 23 added/changed)
- **26 files touched** (1 deleted, 10 route edits, 14 comment updates, 2 test updates, 1 config addition)
- **Zero functionality removed** — admin executive view already lived on /operator (Top5Frame). The deleted page was a duplicate path to the same data.
