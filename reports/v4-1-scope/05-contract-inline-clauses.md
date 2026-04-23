# V4.1: Contract Drawer Inline Clause Expansion

> Coder-3 stub | D164b | Source: D136 spec + D85 amendment (A1 IssueCard components)

## What

In the Contracts drawer, each contract row (currently a DrawerItem summary) can expand inline to show its flagged clauses. Each clause renders as an A1 IssueCard component inside the drawer.

## Why

D136 spec (committed 28b9187) established the v4.0 Contracts drawer as a list of contracts (grouped by `contract_group_id`), not flat clauses. D136 explicitly called out: "Once A1 IssueCard components wire into the drawer (per D85 amendment), each contract row can expand inline to show its clauses. That's a v4.1 enhancement on top of a v4.0 contract list."

D85 amendment confirms Track A components (A1-1 SeverityBadge, A1-2 IssueCard, A1-3 IssueSidebar) survive and mount inside drawers.

## Scope

| File | Change | ~LOC |
|---|---|---|
| `src/components/operator/PillDrawer.tsx` | Add expandable row: tap contract → show/hide clause list below it. Use accordion pattern (only one contract expanded at a time). | ~40 |
| `src/components/operator/ContractClauseList.tsx` | **New.** Fetches `analysis_result.flagged_clauses` for a given contract, renders as A1 IssueCard components. | ~60 |
| A1 components (SeverityBadge, IssueCard) | May need minor props adaptation for drawer context vs full-page context. | ~10 |

## Rough LOC

~110 LOC across 3 files.

## Dependencies

- V4.0 Contracts drawer must ship first (contract list as DrawerItems)
- A1 components must be importable from their current location
- `analysis_result.flagged_clauses` JSONB must be queryable (it is — already used by contract-review page)
