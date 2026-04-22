# D142 Item 3: sync/pings 500s

**Coder-4 Ops | 2026-04-22**

## Diagnosis

### Raw log lines (last 10 pre-fix invocations):

```
19:00:40 | 500 | dpl_51sWNN3C51KGwS1KaYRBA | (empty — no message logged)
18:42:42 | 500 | dpl_3r3Rfaq8oL62Cpn2qDvHP | (empty)
18:27:42 | 500 | dpl_3r3Rfaq8oL62Cpn2qDvHP | (empty)
18:12:42 | 500 | dpl_3r3Rfaq8oL62Cpn2qDvHP | (empty)
17:57:42 | 500 | dpl_EnqtkuXBZ8rpcavYNdrrx | (empty)
17:42:42 | 500 | dpl_EnqtkuXBZ8rpcavYNdrrx | (empty)
17:27:42 | 500 | dpl_EnqtkuXBZ8rpcavYNdrrx | (empty)
17:12:42 | 500 | dpl_EnqtkuXBZ8rpcavYNdrrx | (empty)
16:57:42 | 500 | dpl_EnqtkuXBZ8rpcavYNdrrx | (empty)
16:42:42 | 500 | dpl_EnqtkuXBZ8rpcavYNdrrx | (empty)
```

100% failure rate. Zero log output from the function — crashes before any app code logs.

### Shared root cause hypothesis: CONFIRMED

| Evidence | td-change-detection | sync/pings |
|---|---|---|
| Failure window start | 48h+ ago | 48h+ ago |
| Error cadence | Every 15 min | Every 15 min |
| First fix-deploy success | 13:15:10 (dpl_B4Ue) | 13:16:07 (dpl_B4Ue) |
| Time delta between fixes | — | +57 seconds |

Both failed on every deployment before the env var fix. Both succeeded on the first invocation after the fix. The 57-second gap is just the natural offset between their cron schedules.

### Root cause category: ENV VAR

sync/pings calls TrackDrive via `src/lib/trackdrive.ts` which constructs URLs from `TD_API_BASE_URL` and authenticates with `TD_API_AUTH`. Both had embedded quotes:
```
TD_API_BASE_URL: "https://the-lead-penguin.trackdrive.com/api/v1"
TD_API_AUTH:     "Basic dGRwdWI5..."
```
The fetch would crash immediately on the malformed URL, before any app-level logging could execute.

### Regression vs older drift: OLDER DRIFT

Same evidence as Item 2. Zero 200s in 48h. Predates all recent commits.

## Fix Applied

Same as Item 1 — env var quote cleanup. No code changes.

## Verification (post-fix, raw log output)

```
13:16:07 | 200 | dpl_B4UeHr4P5uipjKQzxtPTY | sync/pings — SUCCESS

Zero 500 errors in 15-minute post-fix window for this endpoint.
```

First successful sync/pings run in 48+ hours.
