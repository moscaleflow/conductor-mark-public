---
directive: "MOP Realtime subscriptions audit"
lane: research
coder: Coder-3
started: 2026-04-20 ~01:30 MDT
completed: 2026-04-20 ~01:45 MDT
---

# MOP Supabase Realtime Audit — 2026-04-20

## 1. Executive Summary

**MOP has zero Supabase Realtime subscriptions.** No `supabase.channel()`, no `.on('postgres_changes')`, no legacy `.from().on()`, no Realtime type imports, no replication publications in migrations. The Realtime service is enabled in local dev config (default) but completely unused by the application.

MOP uses **client-side polling** (`setInterval` at 60-second intervals) for near-real-time data refresh in 3 locations. This is the only mechanism providing live-ish data updates.

**Risk level: None.** No Realtime subscriptions to break during unfreeze or consumer migration. The polling mechanisms will continue to work as long as the underlying tables exist (or are replaced with compatible views/queries).

## 2. Search Methodology

Searched all source files in `~/MOP/` (344+ TypeScript/TSX files, 140+ SQL migrations):

| Pattern | Matches |
|---------|---------|
| `supabase.channel(` / `.channel(` | 0 |
| `.on('postgres_changes'` / `.on("postgres_changes"` | 0 |
| `.on('broadcast'` / `.on('presence'` | 0 |
| `.on('INSERT'` / `.on('UPDATE'` / `.on('DELETE'` / `.on('*'` | 0 |
| `.subscribe(` (Realtime context) | 0 |
| `.removeChannel(` / `.removeAllChannels(` | 0 |
| `RealtimeChannel` / `RealtimePostgresChangesPayload` imports | 0 |
| `REPLICATION` / `supabase_realtime` / `ALTER PUBLICATION` in SQL | 0 |
| Realtime-specific env vars | 0 |

## 3. What MOP Uses Instead: Polling

Three locations use `setInterval` for periodic data refresh:

| File | Interval | What It Polls | Risk on Migration |
|------|----------|---------------|-------------------|
| `app/ping-post/page.tsx` (lines 593, 948) | 60s | Ping-post sync data | Low — queries ping_post tables, not CRM |
| `app/contract-analyzer/page.tsx` (line 375) | Polling | Analysis job status | Low — queries contract_analyses |
| `components/PageHeaderWithStats.tsx` (line 61) | 60s | Dashboard pulse stats | Medium — may query publishers/buyers tables |

**PageHeaderWithStats** is the only polling location that might be affected by @milo/crm migration, since dashboard stats likely aggregate from publisher/buyer tables. When those tables migrate to `crm_counterparties`, the stats queries need updating — but this is a standard query migration, not a Realtime concern.

## 4. Supabase Client Configuration

Both clients are standard REST-only initialization:

| Client | File | Realtime Config |
|--------|------|-----------------|
| `supabase` (anon key) | `lib/supabase.ts` | None — default `createClient()` |
| `supabaseUntyped` (anon key) | `lib/supabase.ts` | None |
| `supabaseAdmin` (service role) | `lib/supabaseAdmin.ts` | None |

No Realtime-specific options (e.g., `realtime: { params: {} }`) passed to any client.

## 5. Database Replication Status

Local dev config (`supabase/config.toml`):
```toml
[realtime]
enabled = true  # default, not custom
```

- No tables added to any Realtime publication
- No `ALTER PUBLICATION supabase_realtime ADD TABLE ...` in any migration
- No `NOTIFY` / `LISTEN` usage in any SQL

## 6. Risk Analysis

### On Unfreeze

**Risk: None.** No Realtime subscriptions exist to fire on data changes. The freeze (`MOP_WRITES_FROZEN=true`) blocks writes at the application layer — even if Realtime were in use, the frozen state only prevents MOP from writing, not from receiving events from external writes. But since there are no subscriptions, this is moot.

### On @milo/crm Migration

**Risk: None from Realtime.** The polling in `PageHeaderWithStats` will need its queries updated when `publishers`/`buyers` tables migrate to `crm_counterparties`, but this is a standard query-level change, not a Realtime migration issue.

### On Future Realtime Adoption

If MOP (or a successor vertical) wants to adopt Realtime in the future:

1. Tables must be added to the Realtime publication: `ALTER PUBLICATION supabase_realtime ADD TABLE <table_name>`
2. RLS policies apply to Realtime (current permissive `USING(true)` policies would allow all subscriptions)
3. The multi-tenant migration (`20300101`) replaces `USING(true)` with tenant-scoped policies — Realtime subscriptions would need the JWT tenant_id claim

## 7. Recommendations

| Action | Priority | Rationale |
|--------|----------|-----------|
| **No action needed** | — | Zero Realtime subscriptions to migrate, retire, or modify |
| **Note PageHeaderWithStats polling** | Low | When @milo/crm migration touches publisher/buyer tables, update the stats queries in this component |
| **Remove from TODO.md worry list** | Now | This audit confirms MOP Realtime is a non-issue — can be struck from pre-migration risk tracking |

## 8. Open Questions for Mark

1. **Was Realtime ever used in MOP and removed, or was it never adopted?** The polling pattern (60s setInterval) suggests Realtime was either considered and rejected, or simply never needed. Understanding the history helps decide whether future verticals should use Realtime or stick with polling.

2. **Should milo-outreach or future verticals adopt Realtime for lead status updates?** The 60-second polling in MOP is adequate for dashboard refresh but wouldn't suit a real-time notification flow (e.g., "new lead just arrived" toast). If operator-assisted capture grows, Realtime could improve responsiveness.
