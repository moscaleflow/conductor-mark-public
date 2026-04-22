# D142 Item 1: Teams Notifications — Quoted Env Vars

**Coder-4 Ops | 2026-04-22**

## Diagnosis

### Root cause: 13 env vars had literal double-quote characters embedded in their values

When these vars were originally added to Vercel (likely pasted with surrounding quotes), Vercel stored the quotes as part of the value. At runtime, URLs became malformed:

```
"https://jdzqkaxmnqbboqefjolf.supabase.co/functions/v1"/send-message
```

### Raw error (from Vercel error-level logs):

```
[Teams] Failed to send to #alerts: [TypeError: Failed to parse URL from
"https://jdzqkaxmnqbboqefjolf.supabase.co/functions/v1"/send-message] {
  [cause]: TypeError: Invalid URL
      at <unknown> (../../opt/rust/nodejs.js:17:19986)
      at o (../../opt/rust/nodejs.js:17:19975) {
    code: 'ERR_INVALID_URL',
    input: '"https://jdzqkaxmnqbboqefjolf.supabase.co/functions/v1"/send-message'
  }
}
```

### Root cause category: ENV VAR — values pasted with surrounding quotes into Vercel dashboard

### Regression vs older drift: OLDER DRIFT

Zero successful runs for td-change-detection or sync/pings in 48h search window. Errors predate all recent commits (D125, D128, etc.). The quoted values were baked in from initial project setup.

## Fix Applied

Removed and re-added all 13 affected env vars with clean values (no surrounding quotes):

| Env Var | Issue |
|---|---|
| PPCRM_MESSAGING_URL | Quoted URL |
| PPCRM_MESSAGING_KEY | Quoted key |
| PPCRM_SUPABASE_ANON_KEY | Quoted JWT |
| CONVOQC_API_URL | Quoted URL |
| CONVOQC_API_KEY | Quoted UUID |
| MOP_API_URL | Quoted URL |
| MOP_BOT_API_KEY | Quoted key |
| LIVE_STATS_API_KEY | Quoted key |
| CRON_SECRET | Quoted secret |
| RESEND_API_KEY | Quoted key |
| TD_API_AUTH | Quoted Basic auth |
| TD_API_BASE_URL | Quoted URL |
| ANTHROPIC_API_KEY | Trailing `\n` |

Deploy: `dpl_BPj7avw4qwzDQE6YcwAY9Xq2amPu` (first pass), then `dpl_H54WDc5gEhBSYNz1pbaUGLrGczJ4` (with Google Sheets vars).

No code changes — env-only fix.

## Verification (15-min post-fix window)

```
ZERO 500 errors in last 15 minutes

=== /api/health-check ===
  Previously: 503 (6 hits/6h) — Teams notification crash
  After fix: no errors

=== Teams notifications ===
  Root cause (quoted PPCRM_MESSAGING_URL) eliminated
  URL now resolves correctly: https://jdzqkaxmnqbboqefjolf.supabase.co/functions/v1/send-message
```
