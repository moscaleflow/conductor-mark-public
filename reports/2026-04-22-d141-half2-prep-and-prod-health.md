# D141 Report: D134 Half 2 Prep + Production Health

**Coder-4 Ops | 2026-04-22**

---

## Item 1: D134 Half 2 Prep — Aging + Calls Data Paths

### Aging API (/api/invoices/aging)

**File:** `src/app/api/invoices/aging/route.ts`

Response shape confirmed:
```json
{
  "current":   { "count": N, "total": N },
  "thirtyDay": { "count": N, "total": N },
  "sixtyDay":  { "count": N, "total": N },
  "ninetyPlus":{ "count": N, "total": N },
  "total":     { "count": N, "total": N, "ar": N, "ap": N },
  "details": [{ "entity_name", "invoice_type", "amount_due", "due_date", "days_outstanding", "status", "bucket" }]
}
```

Bucket boundaries: current=0-30d, thirty_day=31-60d, sixty_day=61-90d, ninety_plus=91+d.
Filters: status IN `['sent', 'overdue', 'jen_approved']`. Orders by due_date ASC.

### Calls Predicate — DOES NOT EXIST

No "Calls" pill predicate exists in the codebase. The code comments mention "flagged calls" as aspirational, but `flagged_calls` was explicitly removed (lines 22-24, 305-306 in pill/route.ts) because it matched 0 rows against live alert titles.

This means Half 2 for "Calls" is greenfield — Coder-1 must define the predicate from scratch.

### Disputes Predicate — Two Implementations

1. **Direct-source** (line 65, 113-156): `disputes` is in `DIRECT_SOURCE_PILLS` set. Fetches directly from `disputes` table, bypasses alerts entirely.
2. **Alert-based** (line 358): `disputes: (a) => titleMatches(a, [/dispute/i])` — matches any alert with "dispute" in the title.

### Overlap Analysis: Which Predicates Match /dispute/i

| Predicate | Regex includes /dispute/i | Other patterns |
|---|---|---|
| `qa_queue` | YES | `/flagged/i`, `/ivr self[- ]dq/i`, `/self[- ]dq/i`, `/quality flag/i` + action_types: `review_bids`, `dispute_review`, `review_dispute` |
| `qc_queue` | YES | `/flagged/i`, `/quality flag/i`, `/ivr self[- ]dq/i` |
| `disputes_signoff` | YES | `/sign[- ]off/i` + action_types: `dispute_review`, `review_dispute` |
| `disputes` (alert-based) | YES | (none) |
| `dispute_recovery` | YES | (none — admin fallback) |

All five predicates would match an alert titled "Dispute detected — Publisher X". No deduplication.

### B2 Overlap: Alert Rows Matching Both /dispute/i and Hypothetical Calls Predicate

Since no Calls predicate exists, there are zero collisions today. However, if Calls uses patterns like `/call quality/i`, `/call[- ]flag/i`, `/flagged call/i`, the overlap with `qa_queue` (which matches `/quality flag/i` and `/flagged/i`) would be significant.

### Proposed Predicate Boundary for Calls vs Disputes

**Calls** — call-quality alerts:
```typescript
calls: (a) => titleMatches(a, [/\bcall\b/i, /ivr/i, /did\b/i, /caller[- ]id/i])
  && !titleMatches(a, [/dispute/i])
```
Rationale: match call-related titles (call, IVR, DID, caller-ID) but explicitly exclude anything with "dispute" in the title to prevent bleed.

**Disputes** — keep existing `/dispute/i` as-is. It already catches dispute-titled alerts. The exclusion lives on the Calls side, keeping Disputes as the catch-all for anything mentioning "dispute."

**qa_queue** — no change. It intentionally spans both calls and disputes as a triage queue. The overlap is by design (QA reviews both).

---

## Item 2: Production Health Watch

**Source:** Vercel production logs, last 6 hours. Project: milo-for-ppc (tlp.justmilo.app).

### 5 Distinct Error Signatures

| # | Endpoint | Count | Severity | Summary |
|---|---|---|---|---|
| 1 | `/api/cron/td-change-detection` | 44 | HIGH | 100% failure. Logs start message then 500s. No error captured — likely unhandled exception or timeout. Fires every ~15min. |
| 2 | `/api/sync/pings` | 44 | HIGH | 100% failure. Zero log output, just 500 status. May crash before app code runs. |
| 3 | `/api/operator/pill` | 12 | MEDIUM | ~40% failure rate (12 of ~30 requests). Intermittent — some succeed, some 500. Persists across deployments. |
| 4 | `/api/health-check` | 6 | MEDIUM | 503s caused by Teams notification failure (see below). |
| 5 | `/api/cron/mediarite-xref` | 1 | LOW | Missing env var: `GOOGLE_SHEETS_MEDIARITE_SHEET_ID`. |

### Root Cause: Teams Notifications Broken

The Supabase functions URL env var contains literal double-quote characters. The constructed URL is:
```
"https://jdzqkaxmnqbboqefjolf.supabase.co/functions/v1"/send-message
```
Note the `"` embedded in the URL. All Teams/Slack notifications fail silently. Health-check returns 503 because it can't send its alert.

### NOT Found
- **"target_user_id does not exist"** — zero matches in 24h
- **Schema errors** — zero matches in 6h

### Priority Fixes

1. **Fix quoted URL env var** — remove literal `"` from the Supabase functions URL value in Vercel env. Unblocks all notifications + fixes health-check 503s.
2. **td-change-detection** — add error logging in catch block, then diagnose the silent 500.
3. **sync/pings** — crashes before any app code logs. Check middleware or import errors.
4. **operator/pill intermittent 500s** — may be auth/session race. Needs targeted investigation.
5. **Set GOOGLE_SHEETS_MEDIARITE_SHEET_ID** in Vercel env.

---

## Summary

| Item | Status | Key finding |
|---|---|---|
| D134 Half 2 aging | READY | Bucket shape confirmed, API live |
| D134 Half 2 Calls predicate | GREENFIELD | No "Calls" predicate exists — was removed. Proposed boundary: `/\bcall\b/i, /ivr/i, /did\b/i` with explicit `/dispute/i` exclusion |
| D134 Half 2 disputes overlap | DOCUMENTED | 5 predicates share `/dispute/i` — by design for QA triage, clean separation proposed |
| Production health | 5 ERROR SIGNATURES | No target_user_id errors. Biggest issues: td-change-detection + sync/pings 100% down, operator/pill ~40% flaky, Teams notifications broken by quoted URL |
