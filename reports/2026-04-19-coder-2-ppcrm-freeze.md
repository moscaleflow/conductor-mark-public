---
directive: PPCRM freeze (Option B — app-level)
lane: migration (acting as ops — Coder-4 not spun up)
coder: Coder-2
started: 2026-04-19 ~20:00 MDT
completed: 2026-04-19 ~20:35 MDT
---

# PPCRM Freeze Report — 2026-04-19

## Section 1: Backup execution

- **Supabase project ID:** jdzqkaxmnqbboqefjolf (confirmed via `supabase projects list` — linked project)
- **Region:** East US (Ohio)
- **Backup location (primary):** `~/ppcrm-freeze-backup-2026-04-19/`
- **Manifest (secondary):** `conductor-mark/backups/ppcrm-freeze-2026-04-19.manifest`
- **Manifest commit SHA:** `bcd98cb`

| File | Size | SHA256 |
|------|------|--------|
| full-backup.sql | 22.1 MB | 3d0856582c0996a24e80979dd23853f94881959adbfb85b9c1a723b0e1c6b470 |
| full-backup.sql.gz | 4.0 MB | 64b18c0a8c4da2061ffc6ebcaed0a9234e790b73f32866f513f6346ac51d9484 |
| data-only.sql | 21.9 MB | b6583236e6a90abed6bc3fe4b0bb5e5b713c0014b47a474f87bf8f4b7b99ba83 |
| data-only.sql.gz | 4.0 MB | 8e330e302fa6b88e2254655b450f9794d1416531b1dfc874cd2f6ee6ea719bbd |

- **pg_dump exit code:** 0 for both dumps
- **Tool used:** `/opt/homebrew/opt/libpq/bin/pg_dump` (v18.3) with credentials from `supabase db dump --dry-run`
- **Note:** Docker not installed. `supabase db dump` not usable. Used direct pg_dump with connection string from dry-run output.

## Section 2: Integrity verification

### Row count comparison — backup vs live (52 tables)

**All 52 tables: exact match, 0 drift.** Full table available in manifest.

Key tables:

| Table | Backup | Live | Delta |
|-------|--------|------|-------|
| call_logs | 35,455 | 35,455 | 0 |
| audit_log | 3,111 | 3,111 | 0 |
| partners | 601 | 601 | 0 |
| bot_logs | 500 | 500 | 0 |
| partner_channels | 299 | 299 | 0 |
| blacklist | 176 | 176 | 0 |
| dids | 183 | 183 | 0 |
| did_buyers | 115 | 115 | 0 |
| leads | 13 | 13 | 0 |
| onboarding_checklists | 8 | 8 | 0 |

**Total: 40,525 rows across 52 tables.**

### Storage bucket comparison

| Bucket | Files | Backed up? |
|--------|-------|------------|
| dashboard | 1 (index.html) | No — already in git repo |
| onboarding-creatives | 0 (empty) | N/A |

### Spot-check (15 records)

- 5 leads: e7f77b67, 051849ba, cf8b40b6, 84a38a0a, 6a13da6d — all FOUND ✅
- 5 partners: e63da02e, 70411a05, ef5b5fb0, dc57cbc8, db165b91 — all FOUND ✅
- 5 onboarding_checklists: 122321f4, dc430b3b, 97240619, 459e28a2, 1e6ffb81 — all FOUND ✅

### Structural validation

- full-backup.sql: Header ✅ Footer ✅ 52 COPY statements ✅
- data-only.sql: Header ✅ Footer ✅ 52 COPY statements ✅
- Circular FK warnings on data-only: team_members, invoices (expected — documented)

## Section 3: App-level kill-switch

### Architecture adaptation

PPCRM is NOT a Next.js app on Netlify (unlike MOP). It's a **static SPA dashboard on Netlify** that calls **Supabase Edge Functions** for all API operations. There are no Netlify API routes.

**Adapted approach:** Freeze guard implemented as a Supabase Edge Function shared module (`_shared/freeze-guard.ts`) checking `PPCRM_WRITES_FROZEN` Supabase secret. Applied to all 26 Edge Functions.

### Implementation

- **Supabase secrets set:**
  - `PPCRM_WRITES_FROZEN=true`
  - `PPCRM_UNFREEZE_KEY=3a5f0bc42ff22aaea72db2d6b9500239`
- **Freeze guard file:** `supabase/functions/_shared/freeze-guard.ts` (16 LOC)
- **Functions patched:** All 26 Edge Functions (import + checkFreeze call in Deno.serve handler)
- **PPCRM commit SHA:** `5a692b5`
- **Deploy method:** `supabase functions deploy --project-ref jdzqkaxmnqbboqefjolf`

### Freeze behavior

- POST/PUT/PATCH/DELETE → 503 with JSON body: `{"error": "PPCRM is frozen...", "frozen_at": "2026-04-19T20:09:00-06:00", "read_endpoints_available": true}`
- GET/OPTIONS/HEAD → pass through to function handler
- POST with `X-Unfreeze-Key: 3a5f0bc42ff22aaea72db2d6b9500239` → bypass freeze

### Verification

| Test | Expected | Actual |
|------|----------|--------|
| POST /salesbot (no key) | 503 | 503 ✅ |
| POST /onboardbot (no key) | 503 | 503 ✅ |
| GET /milo-query | passthrough | 405 (function's own response) ✅ |
| POST /salesbot (with X-Unfreeze-Key) | passthrough | 200 ✅ |

### Emergency unfreeze procedure

```bash
# Option 1: Disable freeze (requires redeploy)
cd ~/PPC3.18/PPCRM
supabase secrets set PPCRM_WRITES_FROZEN=false --project-ref jdzqkaxmnqbboqefjolf
supabase functions deploy --project-ref jdzqkaxmnqbboqefjolf

# Option 2: Per-request bypass (no redeploy)
curl -X POST https://jdzqkaxmnqbboqefjolf.supabase.co/functions/v1/[function] \
  -H "Authorization: Bearer [ANON_KEY]" \
  -H "X-Unfreeze-Key: 3a5f0bc42ff22aaea72db2d6b9500239" \
  -H "Content-Type: application/json" \
  -d '{...}'
```

## Section 4: Scheduled functions freeze

### Architecture difference from MOP

PPCRM uses **pg_cron** (PostgreSQL extension) for scheduling, NOT Netlify scheduled functions. All 11 cron jobs were unscheduled via `SELECT cron.unschedule('job_name')`.

### 11 pg_cron jobs disabled

| Job Name | Schedule | Target | Type |
|----------|----------|--------|------|
| trackdrive-sync-poll | */5 * * * * | trackdrive-sync/poll | HTTP POST |
| milo-alerts-check | */15 * * * * | milo-alerts | HTTP POST |
| campaign-sync | 0 * * * * | campaign-sync | HTTP POST |
| follow-up-reminders | 0 * * * * | onboardbot/follow-up-reminders | HTTP POST |
| hourly-call-sync | 0 * * * * | auto-sync-calls | HTTP POST |
| outreach-triggers-check | 0 */6 * * * | outreach-triggers | HTTP POST |
| cam-vee-reassign | 0 14 * * * | cam_to_vee_reassign() | DIRECT SQL |
| outreach-followup-daily | 0 15 * * * | outreach-followup | HTTP POST |
| trackdrive-backfill-nightly | 0 9 * * * | trackdrive-sync/backfill | HTTP POST |
| auto-clock-out-5pm-mt | 0 23 * * * | UPDATE va_time_logs | DIRECT SQL |
| milo-weekly-insights | 0 5 * * 0 | milo-insights | HTTP POST |

### Verification

```
cron.job table: EMPTY (0 rows)
```

### No code changes needed for cron

Unlike MOP (which required commenting out `Config.schedule` in Netlify function source files), PPCRM's cron is entirely database-side. `cron.unschedule()` removes the job completely. No source file changes needed. No Netlify function schedules to disable (there are none).

### Re-enable instructions

Full re-enable SQL is in the manifest at `backups/ppcrm-freeze-2026-04-19.manifest`.

## Section 5: Operator communications draft

> Heads up — PPCRM is temporarily frozen for writes as of Saturday April 19, 8:09 PM MDT while we consolidate it into milo-engine. Read operations still work — you can still pull reports, view leads, view partners, and see onboarding status. Write operations (creating/updating leads, advancing onboarding steps, approving requests, etc.) are paused. Scheduled sync functions (TrackDrive call sync, outreach triggers, auto clock-out, campaign sync, alert checks) are also paused. If you hit an urgent write-side need, contact Mark directly for emergency access. Thanks for your patience.

## Section 6: Anomalies

1. **11 cron jobs, not 5.** Initial exploration (from Supabase migration files) identified 5 pg_cron jobs. The actual `cron.job` table had 11. Six additional jobs were created via the Supabase dashboard or direct SQL, not via migrations: `follow-up-reminders`, `hourly-call-sync`, `milo-alerts-check`, `milo-weekly-insights`, `auto-clock-out-5pm-mt`, `cam-vee-reassign`.

2. **Two cron jobs are direct SQL, not Edge Function calls.** `auto-clock-out-5pm-mt` runs `UPDATE va_time_logs` directly. `cam-vee-reassign` calls a stored function `cam_to_vee_reassign()`. These would bypass the Edge Function freeze guard if still running. Disabling them via `cron.unschedule()` is the only way to freeze them (app-level DB triggers were out of scope per directive).

3. **pg_stat_user_tables estimates were stale.** Initial row count estimates showed 0 for `blacklist`, `did_buyers`, `dids`, etc. Exact `count(*)` showed 176, 115, and 183 respectively. The table statistics hadn't been refreshed by autovacuum recently. All exact counts matched perfectly between backup and live.

4. **Docker not installed.** `supabase db dump` requires Docker for its internal pg_dump container. Used local `pg_dump` (libpq v18.3) with connection string from `supabase db dump --dry-run`. Same result, different path.

5. **No pre-existing test suite in PPCRM.** Unlike MOP (which has Playwright tests), PPCRM has no automated test suite to run pre/post freeze. Verification was done via curl against live Edge Function endpoints.
