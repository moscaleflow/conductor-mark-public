---
directive: "Re-enable 2 financial crons"
lane: ops
coder: Coder-4
started: 2026-04-20
completed: 2026-04-20
---

# Financial Crons Re-Enable — 2026-04-20

## 1. Executive Summary

**No code change needed. Crons already re-enabled as a side effect of Coder-1's contract-writes freeze deploy.**

Coder-1's deploy of commit `6926a2b5` (Decision 52, selective contract-writes freeze) went live at 2026-04-20T16:51:37Z. That commit includes the original active cron schedules from git — the schedules were never commented in git, only in a local deploy that was never committed. The Netlify deploy from git source restored them.

Both cron endpoints verified live:
- `sync-late-conversions` POST → 504 (expected — long sweep, Netlify scheduled function uses fire-and-forget)
- `sync-unconversions` GET → 200, processed 3-day window successfully

## 2. What Happened

The original MOP freeze (2026-04-19) disabled crons via a local `netlify deploy --prod` that had schedules commented out — but that change was **never committed to git**. All git commits (including `2567466` write-freeze and `6926a2b5` contract-writes freeze) have both cron schedules active in:

- `netlify.toml` lines for `[functions."sync-late-conversions"]` and `[functions."sync-unconversions"]`
- Source file `Config.schedule` in `netlify/functions/sync-late-conversions.mts` and `sync-unconversions.mts`

When Coder-1 deployed `6926a2b5` from git, Netlify rebuilt from source with schedules active.

## 3. Cron Schedules (confirmed active)

| Function | Schedule (UTC) | Schedule (Denver) | Table written | Verified |
|----------|---------------|-------------------|---------------|----------|
| sync-late-conversions | `0 15 * * *` | 9:00 AM daily | call_logs (UPDATE only) | POST → 504 (fire-and-forget, expected) |
| sync-unconversions | `30 15 * * *` | 9:30 AM daily | call_logs (UPDATE only) | GET → 200, `{"success":true,"total_checked":0}` |

## 4. First Scheduled Execution

Next scheduled fire:
- `sync-late-conversions`: 2026-04-21 09:00 AM Denver (15:00 UTC)
- `sync-unconversions`: 2026-04-21 09:30 AM Denver (15:30 UTC)

The 30-day CPA backlog will auto-process over 2-3 daily runs per Coder-3's audit.

## 5. Deploy Reference

| Field | Value |
|-------|-------|
| Netlify deploy ID | `69e65929f392` |
| Commit | `6926a2b5` |
| Published at | 2026-04-20T16:51:37Z |
| Site | tlpmop (dd60b5ca-2af7-4235-8279-ab8d5e260d84) |
| Deploy title | "feat: add selective contract-writes freeze (Decision 52)" |

## 6. Process Note

Coder-3's audit report (f1457d2) said to "uncomment 2 lines in netlify.toml." The discrepancy: crons were disabled via an uncommitted local deploy, not a git commit. Coder-3 was observing the deployed state (correctly), but the fix was already in git. Future freeze/unfreeze of crons should be done via committed changes to avoid this kind of invisible state.
