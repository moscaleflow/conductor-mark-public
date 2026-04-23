# V4.1: Cached Health Check Endpoint for Footer

> Coder-3 stub | D164b | Source: V4.0 spec 02 open question Q1

## What

Add `/api/health-check/summary` that returns the cached result from the last cron health check run instead of re-running all 7 checks live. The footer polls this lightweight endpoint instead of the full health check.

## Why

Spec 02 (footer sync status) has the footer polling `/api/health-check` every 5 minutes. That endpoint runs 7 parallel checks including external API pings (MOP, TrackDrive, ConvoQC). Every open operator tab triggers these pings. With N operators, that's N × 7 external API calls every 5 minutes.

The hourly cron already runs health checks and logs results to `ai_action_log`. The summary endpoint reads the latest log entry — zero external API calls.

## Scope

| File | Change | ~LOC |
|---|---|---|
| `src/app/api/health-check/summary/route.ts` | **New.** Reads latest `action_type='health_check'` from `ai_action_log`, returns `{ overall, checks, timestamp }`. | ~30 |
| `src/components/operator/SyncFooter.tsx` | Change fetch URL from `/api/health-check` to `/api/health-check/summary`. | ~1 |

## Rough LOC

~31 LOC (1 new file, 1 line change).

## Dependencies

- V4.0 spec 02 must ship first (creates SyncFooter)
- Health check cron must continue running hourly (it does — `src/app/api/cron/health-check/route.ts`)
- Staleness: the summary can be up to 1 hour old. If cron fails silently, footer shows stale "healthy" status. Mitigation: include `checked_at` timestamp in the response; footer shows "Checked 47m ago" so operator can see staleness.
