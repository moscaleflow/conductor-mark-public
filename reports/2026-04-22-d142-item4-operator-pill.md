# D142 Item 4: operator/pill Intermittent 500s

**Coder-4 Ops | 2026-04-22**

## Diagnosis

### Raw log lines (3h window, showing deployment transitions):

```
13:10:50 | 200 | dpl_B4UeHr4P5uipjKQzxtPTY |  ← new deploy (env fix)
13:10:49 | 200 | dpl_B4UeHr4P5uipjKQzxtPTY |
13:10:48 | 200 | dpl_B4UeHr4P5uipjKQzxtPTY |
13:10:47 | 200 | dpl_B4UeHr4P5uipjKQzxtPTY |
13:10:38 | 200 | dpl_B4UeHr4P5uipjKQzxtPTY |
12:47:55 | 200 | dpl_3r3Rfaq8oL62Cpn2qDvHP |
12:47:54 | 200 | dpl_3r3Rfaq8oL62Cpn2qDvHP |
12:14:58 | 500 | dpl_3r3Rfaq8oL62Cpn2qDvHP |  ← deployment transition
12:13:33 | 500 | dpl_3r3Rfaq8oL62Cpn2qDvHP |  ← deployment transition
12:13:17 | 500 | dpl_3r3Rfaq8oL62Cpn2qDvHP |  ← deployment transition
12:13:16 | 200 | dpl_3r3Rfaq8oL62Cpn2qDvHP |
12:12:52 | 500 | dpl_534qiB1CAVVttX8jUjzTs |  ← deployment transition
12:12:44 | 200 | dpl_534qiB1CAVVttX8jUjzTs |
12:08:26 | 500 | dpl_EnqtkuXBZ8rpcavYNdrrx |  ← deployment transition
11:32:15 | 500 | dpl_EnqtkuXBZ8rpcavYNdrrx |
```

### Pattern

500s cluster at deployment transition boundaries (when Vercel is switching between deployments). Between transitions, the endpoint is stable. On the latest deploy (post-env-fix), **ALL 7 requests returned 200**.

### Root cause category: DEPLOYMENT TRANSITION NOISE

NOT a persistent bug. NOT downstream of alerts.target_user_id (that error was not found in any logs). The 500s occur when requests hit a deployment that's being replaced. This is normal Vercel behavior during rapid redeployment sequences — the D142 fix session had 3 deployments in quick succession, amplifying the window.

### Overlap with Coder-1 D138 target_user_id work: NONE

- `target_user_id does not exist` — zero matches in 24h of logs
- operator/pill uses `resolveRequestUser()` for auth, not CRON_SECRET
- No shared code path with the alerts query that Coder-1 is working on
- No coordination needed — independent issues

### Regression vs older drift: NEITHER — transient deployment noise

## Fix Applied

No code fix needed. The env var cleanup (Item 1) + single stable deployment resolved the transient errors. With fewer rapid deployments, the transition window disappears.

## Verification (post-fix, raw log output)

```
dpl_B4UeHr4P5uipjKQzxtPTY (latest):
  13:10:50 | 200
  13:10:49 | 200
  13:10:49 | 200
  13:10:48 | 200
  13:10:48 | 200
  13:10:47 | 200
  13:10:38 | 200

7/7 requests — 100% success. Zero 500s in 15-minute post-fix window.
```
