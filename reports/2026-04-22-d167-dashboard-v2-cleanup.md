# D167 — V4.1 Cleanup: Component Relocation + API Rename

**Date:** 2026-04-22  
**Agent:** Coder-1  
**Scope:** Two v4.1 items from D159 spec 06 deferred scope

---

## Summary

Two commits shipping the deferred v4.1 cleanup from the dashboard-v2 decommission (D159):

1. Relocated 3 shared components from `dashboard-v2/` → `shared/`
2. Renamed API route `/api/dashboard-v2` → `/api/operator/executive`

Build passes clean on both commits. No visual changes — pure refactor.

## Commit 1: Component Relocation (`36d10f8`)

### Files moved

| From | To |
|---|---|
| src/components/dashboard-v2/EntityDetailPanel.tsx | src/components/shared/EntityDetailPanel.tsx |
| src/components/dashboard-v2/KanbanBlock.tsx | src/components/shared/KanbanBlock.tsx |
| src/components/dashboard-v2/ContractAnalysisDisplay.tsx | src/components/shared/ContractAnalysisDisplay.tsx |

### Import paths updated (4 files)

| File | Old import | New import |
|---|---|---|
| src/app/pipeline-board/page.tsx | @/components/dashboard-v2/EntityDetailPanel | @/components/shared/EntityDetailPanel |
| src/app/pipeline/page.tsx | @/components/dashboard-v2/KanbanBlock, EntityDetailPanel | @/components/shared/... |
| src/components/ConversationModal.tsx | @/components/dashboard-v2/EntityDetailPanel, ContractAnalysisDisplay | @/components/shared/... |

### Relative import fix

EntityDetailPanel.tsx had a relative import `./AgreedTermsForm` which broke after the move. Updated to absolute path `@/components/dashboard-v2/AgreedTermsForm` (AgreedTermsForm stays in dashboard-v2/).

### Remaining dashboard-v2 components (29 files)

All other components in `src/components/dashboard-v2/` stay in place. They are only used by other dashboard-v2 block components (internal references) and are effectively dead code since the page was deleted in D159.

## Commit 2: API Route Rename (`971566c`)

### Route moved

| From | To |
|---|---|
| src/app/api/dashboard-v2/route.ts | src/app/api/operator/executive/route.ts |

### Caller updated

`src/app/operator/page.tsx:176` — fetch URL changed from `/api/dashboard-v2?...` to `/api/operator/executive?...`

### 301 redirect added

`next.config.ts` — `/api/dashboard-v2` → `/api/operator/executive` (permanent redirect for any external cron/integration hitting the old path)

### Comment references updated (7 files)

All comment references to `/api/dashboard-v2` updated to `/api/operator/executive`:
- src/app/api/operator/pill/route.ts
- src/app/operator/page.tsx
- src/components/AdminExecutiveCards.tsx
- src/lib/greetings.ts
- src/lib/executive-ranker.ts
- src/lib/operator-pills.ts (2 references)

### Dead code note

`src/components/dashboard-v2/WeeklyBillingBlock.tsx` still contains a live fetch to `/api/dashboard-v2?role=billing`. This is dead code (its parent page was deleted in D159). The 301 redirect handles it if somehow invoked.

## Verification

- **npm run build**: PASS on both commits
- `/api/operator/executive` appears in Next.js route list
- `/api/dashboard-v2` no longer in route list (redirect handles old URLs)
- No visual changes — pipeline, pipeline-board, ConversationModal, and /operator executive view all use the same components and data, just from new paths
