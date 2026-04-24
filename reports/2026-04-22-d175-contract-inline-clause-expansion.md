# D175 Report: Contract Inline Clause Expansion (V4.1 Item 05)

**Agent:** Coder-2 | **Date:** 2026-04-22 | **Status:** Complete (build clean, API verified E2E)

---

## Summary

In the Contracts drawer, clicking a contract row now expands inline to show flagged clauses from the analysis — no navigation away. Uses accordion pattern (one contract expanded at a time). Empty state for contracts with no analysis: "No analysis yet — run Milo on this contract."

---

## Changes

### 1. API Route: `/api/operator/clauses` (new, 43 LOC)

**File:** `src/app/api/operator/clauses/route.ts`

GET endpoint. Accepts `contractGroupId` query param. Queries `analysis_records` by `contract_group_id`, returns the latest record's `issues` array sorted by severity (CRITICAL → HIGH → MEDIUM → LOW). Returns `{ clauses: ClauseItem[], count: number }`.

### 2. Component: `ContractClauseList` (new, 203 LOC)

**File:** `src/components/operator/ContractClauseList.tsx`

Lazy-fetches clauses from `/api/operator/clauses` on mount. Renders accordion-style clause cards with:
- Risk level badge (color-coded: CRITICAL red, HIGH orange, MEDIUM amber, LOW green)
- Title + clause reference in collapsed view
- Expanded view: clause reference, issue summary, recommendation, negotiation leverage
- Loading state: "Loading clauses..."
- Empty state: "No analysis yet — run Milo on this contract."

Built with inline styles to match the PillDrawer aesthetic. A1 IssueCard was evaluated but uses Tailwind classes incompatible with the drawer's inline-style pattern.

### 3. Pill Route: `contract_group_id` injection

**File:** `src/app/api/operator/pill/route.ts` (+3 LOC)

- Added `contract_group_id?: string` to DrawerItem interface
- `fetchContractsPill` now sets `contract_group_id: r.contract_group_id ?? r.id` on each item

### 4. PillDrawer: expansion wiring

**File:** `src/components/operator/PillDrawer.tsx` (+50 LOC, -10 LOC)

- Added `contract_group_id?: string` and `action_pill_id?: string` to exported DrawerItem interface
- Added `onPillOpen?: (pillId: string) => void` to PillDrawerProps
- Added `expandedContractId` state (accordion — one at a time)
- Contract rows get pointer cursor + chevron indicator
- Click toggles inline ContractClauseList below the row
- Review/Snooze buttons use `e.stopPropagation()` to avoid triggering expansion
- `expandedContractId` resets on pill change

### 5. Seed: analysis_records test data (Pattern 16)

**File:** `tests/seed-v4-pills.ts` (+45 LOC)

- BlueCross Partners (contract_3) gets 1 `analysis_records` row with 3 flagged issues:
  - CRITICAL: Uncapped indemnification (Section 8.2a)
  - HIGH: Exclusive territory blocks 3 publishers (Section 3.1)
  - MEDIUM: Auto-renewal with 90-day notice (Section 12.4)
- TrueConnect Media (contract_4) intentionally has NO analysis_records → empty state

---

## Verification

| Test | Result |
|------|--------|
| Build | Clean (0 errors, 0 warnings) |
| `GET /clauses?contractGroupId=<BlueCross>` | 3 clauses returned, sorted CRITICAL → HIGH → MEDIUM |
| `GET /clauses?contractGroupId=<TrueConnect>` | `{ clauses: [], count: 0 }` |
| Seed script | analysis_records: 1 row inserted successfully |
| Playwright screenshots | Blocked by pre-existing schema gap: `user_profiles.hidden_pills` column does not exist — prevents pill strip from rendering for test users. Not a D175 regression. |

### Pattern 17: Schema verified

- `analysis_records.contract_group_id` — exists, used in query
- `analysis_records.issues` — JSONB column, contains full ClauseItem objects
- Required NOT NULL columns discovered during seed: `tenant_id`, `document_name`, `role`, `raw_text`, `summary`

---

## Architecture Note

The expansion uses a "fetch-on-expand" pattern: ContractClauseList calls `/api/operator/clauses` only when the row is expanded, not when the drawer opens. This avoids N+1 queries when the drawer has many contracts but the user only inspects a few.

---

## LOC Breakdown

| Component | LOC |
|-----------|-----|
| Clauses API route | 43 |
| ContractClauseList component | 203 |
| Pill route changes | 3 |
| PillDrawer expansion wiring | ~50 |
| Seed data | 45 |
| Playwright spec | 145 |
| **Total** | **~489** |

(Spec estimated ~110 LOC for implementation — came in at ~299 due to A1 IssueCard incompatibility requiring a standalone component, plus Playwright spec and seed.)

---

## Known Issue (pre-existing)

`user_profiles.hidden_pills` column referenced by operator page does not exist in the shared Supabase schema. This prevents pills from rendering for any user profile, blocking Playwright visual verification. Not introduced by D175 — requires a migration to add the column or a code fix to make it optional.

---

Report at: https://raw.githubusercontent.com/moscaleflow/conductor-mark-public/main/reports/2026-04-22-d175-contract-inline-clause-expansion.md
