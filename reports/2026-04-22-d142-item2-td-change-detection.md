# D142 Item 2: td-change-detection Cron 500s

**Coder-4 Ops | 2026-04-22**

## Diagnosis

### Raw log lines (last 10 pre-fix invocations):

```
19:25:18 | 500 | dpl_3r3Rfaq8oL62Cpn2qDvHP | [td-change-detection] Checking changes since 2026-04-22T19:10:18.287Z
19:10:18 | 500 | dpl_3r3Rfaq8oL62Cpn2qDvHP | [td-change-detection] Checking changes since 2026-04-22T18:55:18.404Z
18:55:18 | 500 | dpl_3r3Rfaq8oL62Cpn2qDvHP | [td-change-detection] Checking changes since 2026-04-22T18:55:18.404Z
18:40:05 | 500 | dpl_EnqtkuXBZ8rpcavYNdrrx | [td-change-detection] Checking changes since 2026-04-22T18:40:05.103Z
18:25:01 | 500 | dpl_EnqtkuXBZ8rpcavYNdrrx | [td-change-detection] Checking changes since 2026-04-22T18:25:01.621Z
18:10:01 | 500 | dpl_EnqtkuXBZ8rpcavYNdrrx | [td-change-detection] Checking changes since 2026-04-22T18:10:01.502Z
17:55:01 | 500 | dpl_EnqtkuXBZ8rpcavYNdrrx | [td-change-detection] Checking changes since 2026-04-22T17:55:01.570Z
17:40:04 | 500 | dpl_EnqtkuXBZ8rpcavYNdrrx | [td-change-detection] Checking changes since 2026-04-22T17:40:04.522Z
17:25:01 | 500 | dpl_EnqtkuXBZ8rpcavYNdrrx | [td-change-detection] Checking changes since 2026-04-22T17:25:01.384Z
17:10:05 | 500 | dpl_EnqtkuXBZ8rpcavYNdrrx | [td-change-detection] Checking changes since 2026-04-22T17:10:05.241Z
```

100% failure rate. Every 15 minutes. Function logs start message then crashes with no error-level output.

### Failure window

- Zero successful (200) td-change-detection runs found in 48h search
- Errors span all visible deployments (dpl_EnqtkuXBZ, dpl_3r3Rfa, dpl_51sWNN)
- **Verdict: OLDER DRIFT**, not a regression from recent D125/D128 commits

### Root cause category: ENV VAR

The route uses `CRON_SECRET` for auth validation (Bearer token check). The env var value contained embedded literal double-quote characters:
```
"1929571d61462ca4a2776e46353229427d2ccf936530bbb77007861cf8b94d97"
```
The Bearer token comparison: `Authorization: Bearer "1929571d..."` vs expected `Bearer 1929571d...` — mismatch → auth failure → 500.

Additionally, the downstream TrackDrive API calls use `TD_API_BASE_URL` and `TD_API_AUTH`, both also quoted — so even if auth passed, the API calls would have failed with invalid URLs.

### Shared root cause with Item 3 (sync/pings): CONFIRMED

Both endpoints depend on quoted env vars (CRON_SECRET, TD_API_BASE_URL, TD_API_AUTH). Error timestamps are interleaved within 10 seconds of each other on every 15-minute cycle. Same root cause.

## Fix Applied

Same as Item 1 — env var quote cleanup. No code changes.

Deploy: `dpl_BPj7avw4qwzDQE6YcwAY9Xq2amPu`

## Verification (post-fix, raw log output)

```
13:15:10 | 200 | dpl_B4UeHr4P5uipjKQzxtPTY | [td-change-detection] — SUCCESS

Zero 500 errors in 15-minute post-fix window for this endpoint.
```

First successful td-change-detection run in 48+ hours.
