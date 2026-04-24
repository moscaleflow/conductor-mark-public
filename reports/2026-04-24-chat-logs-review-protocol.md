# D182 — /ask chat_logs Review Protocol + Prompt Refinement Loop

**Date:** 2026-04-24  
**Agent:** Coder-3  
**Scope:** Design only. No implementation, no prompt changes.

---

## 1. Query Templates for Supabase Dashboard

All queries run against `chat_logs` on shared Supabase project `tappyckcteqgryjniwjg`. Open the SQL Editor at `https://supabase.com/dashboard/project/tappyckcteqgryjniwjg/sql`.

### 1a. Last 7 days grouped by pill

```sql
SELECT
  pill,
  COUNT(*) AS total,
  ROUND(AVG(latency_ms)) AS avg_latency_ms,
  ROUND(AVG(input_tokens)) AS avg_input_tok,
  ROUND(AVG(output_tokens)) AS avg_output_tok,
  ROUND(AVG(cache_read_tokens)) AS avg_cache_read,
  SUM(CASE WHEN error IS NOT NULL THEN 1 ELSE 0 END) AS errors
FROM chat_logs
WHERE created_at >= NOW() - INTERVAL '7 days'
GROUP BY pill
ORDER BY total DESC;
```

### 1b. Top 10 longest inputs (context-rich queries)

```sql
SELECT
  pill,
  LEFT(input_text, 200) AS input_preview,
  LENGTH(input_text) AS input_length,
  latency_ms,
  created_at
FROM chat_logs
WHERE created_at >= NOW() - INTERVAL '7 days'
  AND input_text IS NOT NULL
ORDER BY LENGTH(input_text) DESC
LIMIT 10;
```

### 1c. Top 10 shortest inputs (quick-hit usage)

```sql
SELECT
  pill,
  input_text,
  LEFT(response_text, 200) AS response_preview,
  latency_ms,
  created_at
FROM chat_logs
WHERE created_at >= NOW() - INTERVAL '7 days'
  AND input_text IS NOT NULL
  AND LENGTH(input_text) > 0
ORDER BY LENGTH(input_text) ASC
LIMIT 10;
```

### 1d. Rate limit hits (from ask_rate_limits — no 429 in chat_logs since blocked requests don't log)

```sql
-- Active rate-limited users (anyone who's hit the limit recently)
SELECT
  ip_hash,
  hourly_count,
  daily_count,
  hourly_window_start,
  daily_window_start
FROM ask_rate_limits
WHERE hourly_count >= 25 OR daily_count >= 80
ORDER BY daily_count DESC;
```

Note: the /ask route returns 429 before logging to `chat_logs`, so rate-limited requests are invisible in the logs. Power-user detection relies on the `ask_rate_limits` table directly and high `daily_count` values approaching the 100/day cap.

### 1e. Empty/null response_text rows (backend errors)

```sql
SELECT
  id,
  pill,
  LEFT(input_text, 120) AS input_preview,
  error,
  latency_ms,
  created_at,
  ip_hash
FROM chat_logs
WHERE created_at >= NOW() - INTERVAL '7 days'
  AND (response_text IS NULL OR response_text = '' OR error IS NOT NULL)
ORDER BY created_at DESC;
```

### 1f. Unusually long response times (>30s)

```sql
SELECT
  pill,
  LEFT(input_text, 120) AS input_preview,
  latency_ms,
  input_tokens,
  output_tokens,
  cache_read_tokens,
  model_used,
  created_at
FROM chat_logs
WHERE created_at >= NOW() - INTERVAL '7 days'
  AND latency_ms > 30000
ORDER BY latency_ms DESC;
```

### 1g. Classifier confidence distribution (auto-routed requests)

The current schema doesn't store whether a request was auto-classified or which confidence level the classifier returned. The route sends `classified: true` via SSE but doesn't persist it. Workaround for now:

```sql
-- Proxy: requests where pill was selected directly should cluster by ip_hash
-- (same user tends to use the same pill). Auto-routed requests will show
-- more variation in pill per ip_hash.
SELECT
  ip_hash,
  COUNT(DISTINCT pill) AS pills_used,
  COUNT(*) AS total_requests,
  ARRAY_AGG(DISTINCT pill) AS pill_list
FROM chat_logs
WHERE created_at >= NOW() - INTERVAL '7 days'
GROUP BY ip_hash
ORDER BY pills_used DESC;
```

**Schema gap identified:** Add `classified BOOLEAN DEFAULT FALSE` and `classifier_confidence TEXT` columns to `chat_logs` in a future migration. This unblocks direct classifier analysis. See §5 below.

---

## 2. Refinement Decision Framework

### When to add an example to a pill's system prompt

| Signal | Action | Example |
|---|---|---|
| Same question pattern appears 3+ times across different ip_hashes, and the response quality is inconsistent | Add a worked example to the system prompt showing the expected output format | Users keep asking "prep me for a call with [buyer]" and getting inconsistent structure → add a call-prep example to `buyer.ts` |
| Response is correct but misses the user's implicit intent | Add a clarifying rule to the RULES or FRAMEWORK section | Users type "John Smith" expecting a full vet but Milo only searches the name without investigating the company → add "If only a person's name is given, always search for their company affiliation" |

### When to widen or narrow a pill's scope

| Signal | Direction | Action |
|---|---|---|
| Users are asking the vet pill for sales drafting (input contains "draft", "write", "email") | Narrow vet | Add explicit boundary: "If the user asks you to write or draft anything, tell them to use the Sales pill instead" |
| Users on the buyer pill ask about publisher performance (wrong entity type) | Narrow buyer | Add: "If the question is about a publisher, not a buyer, redirect to the Publisher pill" |
| Users on the sales pill ask follow-up questions about an entity they just vetted | Widen sales | Allow sales to reference vet-style lookups when drafting outreach for an entity the user is already discussing |

### When a classifier misroute signals a prompt boundary issue

Check the `1g` query. If a single ip_hash is using 3+ pills per session, look at the actual inputs:

| Pattern | Root cause | Fix |
|---|---|---|
| User asks "draft a message to [entity]" → classifier routes to `vet` | Classifier RULES ordering issue | Adjust the classifier prompt — "WRITE/DRAFT/COMPOSE" rule must match before the entity-name rule |
| User asks "how is [publisher] doing on our campaigns" → routes to `buyer` | "campaigns" keyword ambiguity | Add classifier example: `"how is publisher X performing" → publisher` |
| User gives a LinkedIn URL → routes to `sales` | URL presence triggers "outreach" assumption | Add classifier example: `"linkedin.com/in/someone" → vet` |

### When a pattern of questions suggests a NEW pill or feature

| Signal | Threshold | Candidate |
|---|---|---|
| 5+ requests in 7 days that don't fit any pill cleanly (look at errors + short responses) | Weekly | New pill or scope expansion |
| Multiple users asking the same question that requires database context (e.g., "how many calls did [buyer] do this week") | 3+ users | Feature: connect /ask to live data (TrackDrive, MOP) |
| Users pasting screenshots or PDFs into the chat | Any occurrence | Feature: multimodal input (attachments field exists but isn't wired to Claude Vision yet) |
| Users asking about internal processes ("how do we onboard a publisher", "what's our billing cycle") | Pattern across users | New pill: "ops" or "knowledge base" — static FAQ-style prompt with no web search |

---

## 3. Weekly Cadence Proposal

### Recommended: Weekly, Coder-3 solo → report for Mark

| Item | Detail |
|---|---|
| **Frequency** | Weekly (Monday morning review covering Mon-Sun prior week) |
| **Who runs it** | Coder-3 (research lane) — runs the 7 queries, reads the data, writes the report |
| **Mark's role** | Reads the report, approves/rejects/modifies refinement proposals. 5-10 minute review. |
| **Output** | `reports/YYYY-MM-DD-ask-weekly-review-WNN.md` with findings + proposed refinements |
| **Refinements** | Each proposed prompt change becomes a directive for Coder-1 (touches `src/lib/ask/prompts/`) |
| **Escalation** | If classifier accuracy drops below 80% (estimated from misroute signals), Strategy-Claude redesigns the classifier prompt |

### Weekly review template

```markdown
# /ask Weekly Review — W[NN] (YYYY-MM-DD to YYYY-MM-DD)

## Volume
| Pill | Requests | Avg latency | Avg tokens (in/out) | Errors |
|---|---|---|---|---|

## Adoption
- Unique ip_hashes this week: N
- Repeat users (2+ requests): N
- Power users (10+ requests): [list ip_hash prefixes]

## Quality signals
- Empty/error responses: N (list if > 0)
- Latency > 30s: N (list if > 0)
- Blacklist hits on vet: N

## Classifier performance
- Requests across 3+ pills per ip_hash: N (misroute proxy)
- Notable misroutes: [describe if found]

## Prompt refinement proposals
1. [PILL] — [what to change] — [evidence from logs]
2. ...

## New pill / feature candidates
- [description] — [evidence] — [recommend: yes/no/wait]
```

### Why weekly, not bi-weekly

- /ask is new. Early usage patterns change fast.
- 6+ users generating logs means enough data per week to spot patterns.
- Weekly catches misroutes before they become habits ("this pill never works for me, I'll stop using it").
- Scale to bi-weekly once prompts stabilize (4+ consecutive weeks with no refinement proposals).

---

## 4. First-Pass Metrics to Watch

### 4a. Blacklist hit rate on vet queries

```sql
SELECT
  COUNT(*) FILTER (WHERE input_text LIKE '%[BLACKLIST HIT%') AS blacklist_hits,
  COUNT(*) AS total_vet,
  ROUND(100.0 * COUNT(*) FILTER (WHERE input_text LIKE '%[BLACKLIST HIT%') / NULLIF(COUNT(*), 0), 1) AS hit_rate_pct
FROM chat_logs
WHERE pill = 'vet'
  AND created_at >= NOW() - INTERVAL '7 days';
```

**Why it matters:** The blacklist pre-check (100 LOC in route.ts) loads all 343+ entries and fuzzy-matches before every vet query. If hit rate is near 0%, the pre-check is costing latency for no value. If it's >10%, it's earning its keep — users are asking about known-bad entities.

**What to watch for:** Partial-match false positives. The matcher triggers on any word >4 chars that appears in a blacklist company name. "About" and "their" are excluded, but other common words might cause spurious hits. If users report "blacklist hit" on clean entities, tighten the matching.

### 4b. Average tokens per pill (cost tracking)

```sql
SELECT
  pill,
  ROUND(AVG(input_tokens)) AS avg_in,
  ROUND(AVG(output_tokens)) AS avg_out,
  ROUND(AVG(cache_read_tokens)) AS avg_cache_read,
  ROUND(AVG(input_tokens + output_tokens)) AS avg_total,
  ROUND(AVG(cache_read_tokens) * 100.0 / NULLIF(AVG(input_tokens), 0), 1) AS cache_hit_pct
FROM chat_logs
WHERE created_at >= NOW() - INTERVAL '7 days'
  AND error IS NULL
GROUP BY pill
ORDER BY avg_total DESC;
```

**Why it matters:** Each pill's system prompt is cached (`cache_control: { type: 'ephemeral' }`). High `cache_hit_pct` means the ephemeral cache is working (same prompt reused within 5-minute TTL). Low cache hit means users are spread across pills or usage is too sparse for the cache to help.

**Cost estimate per request:** At Sonnet 4.6 pricing, a typical 1,500-token input + 800-token output ≈ $0.012/request. With cache reads, drops to ~$0.005. At 100 requests/day across 6 users, that's ~$0.50-$1.20/day.

### 4c. Repeat-user identification via ip_hash

```sql
SELECT
  ip_hash,
  COUNT(*) AS total_requests,
  COUNT(DISTINCT pill) AS pills_used,
  MIN(created_at) AS first_seen,
  MAX(created_at) AS last_seen,
  ARRAY_AGG(DISTINCT pill ORDER BY pill) AS pill_list
FROM chat_logs
WHERE created_at >= NOW() - INTERVAL '7 days'
GROUP BY ip_hash
ORDER BY total_requests DESC;
```

**Why it matters:** Adoption vs. one-and-done. If Mark sees 6 ip_hashes with 1 request each, /ask isn't sticky. If 3 ip_hashes have 20+ requests, those are the power users whose workflows should drive prompt refinement.

**Limitation:** ip_hash is SHA-256 of IP + salt. Users on the same office network share an IP → same hash. VPN users may have rotating IPs → multiple hashes. It's a proxy, not a user identifier. If this becomes a problem, consider adding an optional `user_name` field (no auth, just a self-reported label).

### 4d. Latency distribution by pill

```sql
SELECT
  pill,
  PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY latency_ms) AS p50_ms,
  PERCENTILE_CONT(0.9) WITHIN GROUP (ORDER BY latency_ms) AS p90_ms,
  PERCENTILE_CONT(0.99) WITHIN GROUP (ORDER BY latency_ms) AS p99_ms,
  MAX(latency_ms) AS max_ms
FROM chat_logs
WHERE created_at >= NOW() - INTERVAL '7 days'
  AND error IS NULL
GROUP BY pill
ORDER BY p50_ms DESC;
```

**Why it matters:** Vet pill uses web_search aggressively (up to 5 searches per request). If p90 latency for vet is >30s, either the prompt is triggering too many searches or web_search is slow. Other pills should be under 15s p90 since they're mostly generation.

---

## 5. Schema Enhancement Recommendations (Future)

These are NOT part of this protocol — they're improvements that would make future reviews easier. Each would be a small directive.

| Enhancement | Column/table | Why |
|---|---|---|
| Track auto-classification | `classified BOOLEAN DEFAULT FALSE` on chat_logs | Distinguish pill-pressed vs auto-routed requests |
| Track classifier confidence | `classifier_confidence TEXT` on chat_logs | Measure HIGH vs LOW ratio directly |
| Track blacklist hits | `blacklist_hit BOOLEAN DEFAULT FALSE` on chat_logs | Avoid regex-matching `input_text` for `[BLACKLIST HIT` |
| User self-ID | `user_label TEXT` on chat_logs | Optional — lets users identify themselves for attribution |
| Feedback signal | `feedback TEXT CHECK (feedback IN ('good','bad','wrong_pill'))` on chat_logs | Minimal feedback loop — one click in the UI |

Priority: `classified` + `classifier_confidence` first (cheapest, most useful). `feedback` is the highest-value but requires UI work.

---

## 6. Three Sample Queries Ready to Run

These are copy-paste-ready for the first review session.

### Sample 1: "Health check" — overall volume and error rate

```sql
SELECT
  DATE_TRUNC('day', created_at AT TIME ZONE 'America/Denver') AS day,
  COUNT(*) AS requests,
  COUNT(DISTINCT ip_hash) AS unique_users,
  SUM(CASE WHEN error IS NOT NULL THEN 1 ELSE 0 END) AS errors,
  ROUND(AVG(latency_ms)) AS avg_latency_ms,
  ROUND(SUM(input_tokens + output_tokens) / 1000.0, 1) AS total_ktokens
FROM chat_logs
GROUP BY 1
ORDER BY 1 DESC
LIMIT 14;
```

### Sample 2: "What are people actually asking?" — recent inputs by pill

```sql
SELECT
  pill,
  LEFT(input_text, 150) AS question,
  latency_ms,
  output_tokens,
  created_at AT TIME ZONE 'America/Denver' AS asked_at
FROM chat_logs
WHERE created_at >= NOW() - INTERVAL '7 days'
  AND error IS NULL
ORDER BY created_at DESC
LIMIT 30;
```

### Sample 3: "Is anyone actually coming back?" — repeat usage

```sql
SELECT
  ip_hash,
  COUNT(*) AS requests,
  COUNT(DISTINCT DATE_TRUNC('day', created_at)) AS active_days,
  ROUND(AVG(latency_ms)) AS avg_latency,
  STRING_AGG(DISTINCT pill, ', ' ORDER BY pill) AS pills_used
FROM chat_logs
WHERE created_at >= NOW() - INTERVAL '7 days'
GROUP BY ip_hash
HAVING COUNT(*) >= 2
ORDER BY requests DESC;
```

---

## Summary

| Component | Status |
|---|---|
| 7 query templates | Ready — copy-paste to Supabase SQL Editor |
| Refinement decision framework | Defined — when to change prompts, widen/narrow scope, add pills |
| Weekly cadence | Weekly, Coder-3 runs queries + writes report, Mark reviews |
| First-pass metrics | 4 metrics with queries + interpretation guidance |
| Schema gaps | 5 enhancements identified, prioritized |
| Sample queries | 3 ready to run on first review |

**Next step:** Mark approves protocol. Coder-3 runs the first review when 7 days of chat_logs data exist. First review fires as a directive.

---

Report at: https://raw.githubusercontent.com/moscaleflow/conductor-mark-public/main/reports/2026-04-24-chat-logs-review-protocol.md
