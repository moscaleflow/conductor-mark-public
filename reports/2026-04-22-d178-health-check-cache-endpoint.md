# D178 — Health Check Cache Endpoint

**Coder:** 4 (Ops)  
**Directive:** D178  
**Date:** 2026-04-22  
**Status:** COMPLETE

---

## Summary

Added `/api/health-check/summary` that reads cached health state from `ai_action_log` instead of re-running 7 external API checks. SyncFooter now polls this endpoint. Eliminates N×7 external API calls per poll cycle.

## Changes

| File | Change | LOC |
|---|---|---|
| `src/app/api/health-check/summary/route.ts` | **New.** Reads latest `action_type='health_check'` from `ai_action_log`, returns cached report with `Cache-Control: public, max-age=60`. | +49 |
| `src/components/operator/SyncFooter.tsx` | Fetch URL changed from `/api/health-check` to `/api/health-check/summary`. | 1 line |

## Pattern 17 — Schema Verification

No new table columns. The summary endpoint reads the existing `ai_action_log.data_snapshot` JSONB column, which the hourly health-check cron already populates with the full health report (`{ timestamp, overall, quietHours, checks[], alertsSent }`). Zero schema changes.

## Implementation Details

**Cache layers:**
1. **Vercel edge cache** — `Cache-Control: public, max-age=60`. Verified: second request returns `x-vercel-cache: HIT`. Within the SyncFooter's 5-minute poll interval, most requests are served from edge without invoking the serverless function.
2. **SyncFooter client-side dedup** — `DEDUP_MS = 4 * 60 * 1000` (4 minutes). Even if the tab triggers multiple fetches, only one per 4 minutes reaches the network.
3. **Cron freshness** — health-check cron runs `0 * * * *` (hourly). Summary data is at most ~1 hour old.

**Staleness detection:** `STALE_THRESHOLD_MS = 2 * 60 * 60 * 1000` (2 hours). If the cron misses 2 consecutive runs, the response includes `stale: true`. The SyncFooter shows "Checked Xm ago" so the operator can see staleness.

**Error fallback:** If no health_check entries exist in `ai_action_log`, returns `overall: 'degraded'` with `stale: true` and empty checks. Non-fatal — footer shows degraded state with staleness indicator.

## Verification

### Response time comparison (10x curl)

| Endpoint | Requests | Avg Response | External API Calls |
|---|---|---|---|
| `/api/health-check` (live) | 1 | 1.07s | 7 |
| `/api/health-check/summary` (cached) | 10 | 0.26s | 0 |

**4x faster**, zero external API fanout.

### Cache-Control verification

```
Request 1: x-vercel-cache: MISS (function invoked, DB read)
Request 2: x-vercel-cache: HIT  (served from edge, no function)
```

### Bandwidth impact calculation (from D168 auditor)

**Before (live endpoint):**
- 50 operators × 12 polls/hour = 600 health-check calls/hour
- Each call fans out to 7 external services = 4,200 external API calls/hour

**After (cached summary):**
- 50 operators × 12 polls/hour = 600 summary calls/hour
- With 60s edge cache: ~60 cache misses/hour (1 per minute) → 60 DB reads/hour
- External API calls/hour: 1 (from hourly cron) × 7 = **7 total**
- Reduction: 4,200 → 7 = **99.8% reduction**

### Pre-existing build errors (not introduced by D178)

`npm run build` fails on pre-existing type errors in `pill/route.ts:1059` (`fetchDisputesPill` argument count) and `page.tsx:506` (`onPillOpen` prop). These are from other Coders' in-flight D164b work. D178 changes type-check clean — zero errors in `summary/route.ts` or `SyncFooter.tsx`.

## What stays unchanged

- `/api/health-check` — still runs 7 live checks. Used by the hourly cron + manual debugging.
- Cron schedule — still `0 * * * *`. No changes to `vercel.json`.
- Teams alerting — still triggered by the live endpoint via cron. Summary endpoint is read-only.

## Deferred (per directive)

- Auth + rate-limit on `/api/health-check` — deferred to D179.

## Commits

| SHA | Repo | Description |
|---|---|---|
| 5b60488 | milo-for-ppc | feat: /api/health-check/summary + SyncFooter URL update |
| (this commit) | conductor-mark | D178 report |

---

Report at: https://raw.githubusercontent.com/moscaleflow/conductor-mark-public/main/reports/2026-04-22-d178-health-check-cache-endpoint.md
