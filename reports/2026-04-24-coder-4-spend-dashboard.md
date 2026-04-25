# D-spend: Spend Dashboard Widget on /review

**Coder:** Coder-4 (Ops)
**Date:** 2026-04-24
**Decision:** 103
**Status:** COMPLETE

## What was built

Real-time token spend dashboard widget at the top of `/review` page. Three components:

### 1. Pricing config (`src/lib/review/pricing.ts`)
- Externalized Anthropic pricing per model in $/MTok
- Models covered: claude-sonnet-4-6, claude-opus-4-6, claude-haiku-4-5, plus dated IDs
- Fallback: prefix matching (contains 'opus' → opus rates, 'haiku' → haiku, else sonnet default)
- `calculateCost()` handles input + output + cache_read tokens

### 2. API extension (`src/app/api/review/route.ts`)
- `?mode=spend_stats` — aggregates chat_logs by today/week/month (Pacific midnight boundaries)
- `?mode=rate_limits` — returns ask_rate_limits with global vs per-IP separation
- Existing review query (no mode) unchanged
- Pacific time: DST-aware via `Intl.DateTimeFormat` offset calculation

### 3. Dashboard UI (inline in `src/app/review/page.tsx`)
- Three time bucket cards: Today, This Week, This Month
  - Total cost, total tokens, query count, per-model breakdown
- Alerts panel: top 3 most expensive, top 3 slowest, auto-flagged count
- Rate limits panel: global hourly bar with color-coded progress (green/yellow/red), top 3 IP consumers
- Auto-refresh every 60 seconds via `useEffect` + `setInterval`
- Graceful degradation on API failure

## Verification

| Check | Result |
|---|---|
| TypeScript | Clean (no new errors; 1 pre-existing pdf-parse error) |
| Build | Pre-existing PPC engine import errors only |
| spend_stats API | 39 queries, $0.52, claude-sonnet-4-6, 2 flagged |
| rate_limits API | 4/500 global hourly, 2 IP consumers |
| Playwright test | Written (3 tests), needs deploy to run against production |

## Files changed

| File | Action | Lines |
|---|---|---|
| `src/lib/review/pricing.ts` | Created | ~30 |
| `src/app/api/review/route.ts` | Rewritten | ~165 |
| `src/app/review/page.tsx` | Extended | +155 |
| `tests/d-spend-dashboard.spec.ts` | Created | ~65 |

## Live API output (local verification)

**spend_stats:**
- Today: 39 queries, $0.52, all on claude-sonnet-4-6
- Top expensive: $0.033 (Axad/AX Law), $0.031 (Apex Digital), $0.031 (JB Persicions)
- Top slow: 37.9s (JB Persicions), 35.6s (Axad), 30.2s (FutureTel)
- Flagged: 2 queries

**rate_limits:**
- Global: 4/500 hourly
- Top IP: b40dd548 (4/hr, 31/day), a23dd729 (1/hr, 1/day)

## Notes

- Decision 102 collision detected (Coder-3 /synth plan + Coder-1 CRM context both used 102). This decision logged as 103.
- Visual screenshot pending deploy — Playwright test captures to Desktop on first production run.
- Pricing config uses simple model-string matching. If Anthropic changes model IDs, update pricing.ts.
