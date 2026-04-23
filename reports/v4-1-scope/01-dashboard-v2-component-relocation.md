# V4.1: Relocate dashboard-v2 Shared Components

> Coder-3 stub | D164b | Prerequisite: V4.0 spec 06 (decommission /dashboard-v2 route) ships first

## What

Move reusable components out of `src/components/dashboard-v2/` into `src/components/shared/`. The dashboard-v2 directory becomes a dead namespace after the route is deleted — keeping shared components there is confusing.

## Why

After spec 06 deletes `src/app/dashboard-v2/page.tsx`, the 32 component files (12,909 LOC) and type file (412 LOC) remain because other pages import from them. Leaving them in a `dashboard-v2` directory after that route no longer exists creates a misleading namespace.

## Scope

**Components to relocate (used outside dashboard-v2):**

| Component | Current path | Consumers |
|---|---|---|
| `EntityDetailPanel` | `components/dashboard-v2/EntityDetailPanel.tsx` | pipeline-board, pipeline, ConversationModal |
| `KanbanBlock` | `components/dashboard-v2/KanbanBlock.tsx` | pipeline |
| `ContractAnalysisDisplay` | `components/dashboard-v2/ContractAnalysisDisplay.tsx` | ConversationModal |

**Types to relocate:**

| File | Current path | Consumers |
|---|---|---|
| `dashboard-v2-types.ts` | `lib/dashboard-v2-types.ts` | operator-pills (RoleName), walkthrough API (ROLE_BLOCKS, BlockId, etc.), OnboardingTour (BlockId) |

**Components to delete (only used by deleted page):** The remaining ~29 dashboard-v2 block components (PulseBlock, AlertsBlock, TopPublishersBlock, etc.) are only consumed by `dashboard-v2/page.tsx`. Once that page is deleted, these become dead code.

## Rough LOC

- Relocate 3 components: ~0 LOC (move files, update imports)
- Update ~8 import paths: ~8 LOC
- Delete 29 dead components: **-12,000 LOC**
- Rename types file: ~0 LOC (move file, update 4 imports)

**Net: ~-12,000 LOC.** Largest single cleanup in the codebase.

## Dependencies

- Spec 06 must ship first (deletes the route, proves nothing breaks)
- Must verify each of the 29 "dead" components truly has no other consumer (`grep -r` each export)
- `/api/dashboard-v2/route.ts` stays until spec 02 (API rename) ships
