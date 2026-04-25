# D102 Report: /ask Pills Wired to @milo/crm Context

**Agent:** Coder-2 | **Date:** 2026-04-24 | **Status:** SHIPPED (c3e29a1 + c50bb24, pushed to main)

---

## Summary

Each /ask pill now receives CRM context before Claude responds. When an entity name appears in the user query, the backend does a CRM lookup and prepends `[CRM CONTEXT: ...]` to the user message — same pattern as the existing blacklist pre-check. Claude's response is shaped by actual relationship data instead of treating every entity as unknown.

---

## Changes

### 1. `src/lib/ask/crm-context.ts` (new, 150 LOC)

**Entity extraction:** Proper-noun phrase detection with stop words (40+ words) and business suffix filtering (LLC, Inc, Corp, etc). Quoted strings take priority. Max 3 candidates per query.

**CRM lookup:** `findCrmContext(query, pill)` —
- Searches `crm_counterparties` via ilike on company name (same raw-query pattern as existing `crm-client.ts:findOrCreateCounterparty` — SDK lacks name search)
- On match: loads contacts (`listContacts`), activities (`listActivities`)
- For buyer/publisher pills: also loads onboarding state (`getOnboardingStatusByCounterparty`)
- Falls back to `crm_leads` search if no counterparty match

**Context formatting:** `formatCrmContext(ctx, pill)` —
- **Vet pill + match:** "Reference this existing relationship — do not treat as a cold unknown."
- **Sales pill + active partner:** "This is an existing active partner — shift to account management framing."
- **Sales pill + lead:** "This is an existing lead — reference status and suggest next action."
- **No match:** "No record of this entity in TLP CRM. Treat as unknown — do not invent relationship history."

### 2. `/api/ask/route.ts` (+10 LOC)

- Import `findCrmContext`, `formatCrmContext`, `CrmContext`
- After blacklist pre-check (line 262), before Claude streaming (line 280):
  ```
  crmCtx = await findCrmContext(userMessage, pill);
  userMessage = `${formatCrmContext(crmCtx, pill)}\n\n${userMessage}`;
  ```
- `context_metadata` JSONB logged to `chat_logs` on every query where CRM match found

### 3. Schema: `context_metadata` column (SCHEMA.md + migration)

- `ALTER TABLE chat_logs ADD COLUMN IF NOT EXISTS context_metadata JSONB;`
- Migration: `20260424400001_context_metadata.sql` — applied
- SCHEMA.md amended per Decision 38

---

## Verification

### Unit tests (5/5 passing)

| Query | Pill | Expected | Result |
|-------|------|----------|--------|
| "Vet Sky Marketing" | vet | Match Sky Marketing (publisher, active) | PASS — 216 chars / ~54 tokens |
| "Vet RandomNewCo LLC" | vet | No match | PASS — 106 chars / ~27 tokens |
| "Draft a follow-up for Naked Media" | sales | Match Naked Media + active partner warning | PASS — 234 chars / ~59 tokens |
| "How is Flex Marketing performing?" | publisher | Match Flex Marketing (publisher, active) | PASS — 209 chars / ~53 tokens |
| "Prep me for a call with Turtle Leads" | buyer | Match Turtle Leads + contacts | PASS — 148 chars / ~37 tokens |

### Build status

Build passes clean. D200 unblocked (Coder-1 fixed engine imports in f87ec3c-fac39fa).

### E2E / Playwright

Playwright visual verification passed (d102-crm-context.spec.ts):
- /ask page loads with 4 pill cards
- Vet pill selects, shows VET MODE description
- Query "Vet Sky Marketing" typed successfully

CRM context injection is server-side (invisible in UI) — verified via unit tests. Stitched screenshot: `/Users/markymark/Desktop/d102-crm-context-verification.png`

---

## Tool-Reuse Audit

| SDK Method | Package | Usage |
|-----------|---------|-------|
| `listContacts` | @milo/crm | Load contacts for matched counterparty |
| `listActivities` | @milo/crm | Load recent activity for matched counterparty |
| `crmClient()` | src/lib/crm-client.ts (wraps @milo/crm) | Get tenant-scoped CRM client |
| `getOnboardingStatusByCounterparty` | src/lib/onboarding-client.ts (wraps @milo/onboarding) | Load onboarding state for buyer/publisher pills |
| `crm.supabase.from().ilike()` | Raw via @milo/crm client | Name-based counterparty search (SDK lacks this — existing pattern) |

**New SDK methods added: 0**
**Primitives modified: 0**

---

## Token Cost

| Metric | Value |
|--------|-------|
| Average context size | 183 chars / ~46 tokens |
| Range | 27-59 tokens per query |
| CRM match rate (test set) | 4/5 (80%) |
| No-match overhead | ~27 tokens ("no record" message) |
| With-match overhead | ~37-59 tokens depending on data density |

Well under the 200-500 token estimate.

---

## Resolved Blockers

1. ~~**D200 build breakage**~~ — Resolved by Coder-1 (f87ec3c-fac39fa).
2. ~~**ANTHROPIC_API_KEY**~~ — Key restored in `.env.local` (D105).
3. ~~**Playwright/screenshots**~~ — Visual verification complete. Stitched to Desktop.

---

## LOC Breakdown

| Component | LOC |
|-----------|-----|
| crm-context.ts | 150 |
| route.ts changes | 10 |
| SCHEMA.md amendment | 25 |
| Migration | 4 |
| **Total** | **~189** |
