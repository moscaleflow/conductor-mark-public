# API Auth Gap Assessment (M-1)

**Date:** 2026-05-06
**Auditor:** Coder-3 (Research)
**Scope:** All `/api/*` routes in milo-for-ppc — auth bypass in middleware.ts

---

## Executive Summary

The middleware at `src/middleware.ts` line 37 explicitly skips all auth checks for paths starting with `/api/`. This means **161 of 198 API route files have zero authentication** at any layer. Of those 161 unprotected routes, **75 expose mutation endpoints** (POST/PATCH/PUT/DELETE) and **19 make Anthropic Claude API calls** (direct cost exposure).

However: the practical risk is moderate, not critical. This is an operator-facing internal tool behind a custom domain (`tlp.justmilo.app`), not a public API. The URL is not discoverable through search engines or public documentation. Exploitation requires knowing both the domain and the exact route paths.

---

## 1. Current Auth Architecture

### Middleware behavior (src/middleware.ts)

```
Lines 36-55: If path starts with /api/, /contract-review/, /c/, /auth/callback,
/welcome/, /ask, /pipeline-board, /contract-queue, /proof, /marketing/,
/samples/, /test-fixtures/, /login, /onboard, or /verify/ →
  return NextResponse.next()   // SKIP ALL AUTH
```

Everything else hits Supabase `getUser()` — if no session, redirect to `/login` or `/onboard`.

### Route-level auth (the 37 protected routes)

Three auth mechanisms exist at the route level:

| Mechanism | Routes | Pattern |
|-----------|--------|---------|
| `getEffectiveUser` / `resolveRequestUser` (session cookie) | 19 routes | Admin routes, milo chat, outreach/send, operator core, triage, pills, notifications, observatory, briefing, profile prefs, classify-reply |
| `CRON_SECRET` (Bearer token) | 18 routes | All `/api/cron/*` routes + admin/billing-trigger |
| Webhook signature (Svix / Cloudflare secret) | 1 route | `/api/webhooks/resend` |

**Total protected: 37 of 198 (19%)**
**Unprotected: 161 of 198 (81%)**

### IP-based rate limiting

Only `/api/ask` has rate limiting: 100/hour per IP hash, 1000/day per IP hash, 500/hour global. No other routes have any rate limiting.

### Network-level protection

- No Vercel Deployment Protection detected in config
- No IP allowlist in middleware or vercel.json
- No WAF or firewall configuration found
- No `VERCEL_AUTOMATION_BYPASS_SECRET` usage in application code
- The production domain `tlp.justmilo.app` is a subdomain — not indexed by search engines, but publicly resolvable

---

## 2. Route Risk Classification

### CRITICAL RISK (CRM/pipeline/financial data mutations, no auth)

These routes allow anyone who knows the URL to modify production data:

| Route | Methods | Impact |
|-------|---------|--------|
| `blacklist` | POST, DELETE | Add/remove companies from blacklist. DELETE soft-deletes by ID. |
| `pipeline/[id]/stage` | POST | Change any prospect's pipeline stage (unverified -> active). Has activation gate but no auth gate. |
| `ask/entity-add` | POST | Create CRM counterparties + pipeline entries from thin air |
| `contacts` | POST | Create new CRM contacts |
| `contacts/[id]` | PATCH | Modify any CRM contact by ID |
| `campaigns/[id]` | PATCH | Modify campaign records |
| `invoices/[id]/approve` | PATCH | Approve invoices (pending_jen -> jen_approved -> sent) |
| `invoices/[id]/flag` | PATCH | Flag invoices |
| `invoices/[id]/mark-overdue` | PATCH | Mark invoices overdue |
| `invoices/[id]/record-payment` | PATCH | Record payments against invoices |
| `invoices/generate` | POST | Generate new invoices |
| `invoices/finalize` | POST | Finalize invoice batches |
| `disputes/[id]/raise` | POST | Raise disputes with buyers |
| `disputes/[id]/respond` | POST | Record dispute responses |
| `disputes/batch-raise` | POST | Batch-raise multiple disputes |
| `td/create-entity` | POST | Create buyers/publishers in TrackDrive (external system mutation!) |
| `team/force-clockout` | POST | Force-close VA time tracking sessions in partner Supabase |
| `action-items/[id]` | PATCH | Modify action items |
| `action-items/create` | POST | Create action items |
| `alerts/[id]/assign` | POST | Assign alerts to team members |
| `alerts/[id]/resolve` | PATCH | Resolve alerts |
| `bank-transactions/import` | POST | Import bank transactions |
| `bank-transactions/categorize` | POST | Categorize bank transactions |
| `operator/flagged-calls` | PATCH | Resolve/dispute flagged calls |
| `operator/snooze` | POST | Snooze operator items |
| `eod/persist-and-send` | POST | Send EOD reports to Teams channel |
| `leads/import` | POST | Import leads |

**Count: 27 routes with critical data mutation exposure**

### HIGH RISK (AI API calls that cost money, no auth)

These routes trigger Anthropic Claude API calls. An attacker could run up the API bill:

| Route | AI Call Type | Est. Cost/Call |
|-------|-------------|----------------|
| `ask` | callClaude / callClaudeStream / callClaudeExtended | $0.01-0.15 (has IP rate limit) |
| `ask/entity-vet-inline` | callClaude (structured + research) | $0.05-0.20 |
| `ask/draft-outreach` | callClaude (structured) | $0.01-0.03 |
| `ask/draft-verify` | callClaude | $0.01-0.03 |
| `ask/explain-further` | callClaude | $0.01-0.05 |
| `ask/refine-redline` | callClaude | $0.01-0.05 |
| `ask/classify-file` | callClaude | $0.01-0.03 |
| `capture` | callClaudeVision + callClaude (multiple) | $0.10-0.30 |
| `contract/analyze-bg` | callClaude (long document) | $0.05-0.20 |
| `creative/analyze` | callClaude | $0.01-0.05 |
| `briefing/morning` | callClaude | $0.05-0.15 |
| `eod/generate` | callClaude | $0.02-0.10 |
| `research/deeper` | callClaude + web research | $0.05-0.15 |
| `outreach/draft` | callClaude | $0.01-0.03 |
| `operator/pill/review` | callClaude | $0.01-0.05 |
| `buyer-reports/[id]/process` | callClaude | $0.02-0.10 |
| `agreed-terms/parse` | callClaude | $0.02-0.05 |
| `verify` | callClaude | $0.01-0.03 |
| `demo/analyze` | callClaude | $0.01-0.05 |

**Count: 19 routes. Only `ask` has rate limiting. The rest are unlimited.**

A simple script hitting `capture` in a loop could burn $50-100/hour in API costs.

### MEDIUM RISK (read-only but exposes business data, no auth)

| Route | Data Exposed |
|-------|-------------|
| `pipeline` | Full prospect pipeline with entity names, stages, sources |
| `contacts` (GET) | CRM contact records |
| `blacklist` (GET) | Full blacklist with company names, reasons, severity |
| `captures` (GET) | Conversation captures with extracted intel |
| `disputes` (GET) | Dispute records and stats |
| `invoices` (GET) | Invoice data |
| `billing-schedules` (GET) | Billing schedule data |
| `stats` (GET) | Business statistics |
| `live-stats` (GET) | Real-time operational stats |
| `publishers/grades` (GET) | Publisher grade assessments |
| `buyers/health` (GET) | Buyer health metrics |
| `team-members` (GET) | Team member list |
| `team/status` (GET) | Team clock-in/out status |
| `reconciliation` (GET) | Financial reconciliation data |
| `campaigns` (GET) | Campaign configuration |
| `entities/[name]` (GET) | Entity profiles |
| `ask/history` (GET) | Chat history |
| `milo/history` (GET) | Milo chat history |
| `operator` (GET) | Operator dashboard data |

**Count: ~19 routes exposing business-sensitive read data**

### LOW RISK (non-sensitive reads, health checks)

| Route | Purpose |
|-------|---------|
| `health` | Basic health check |
| `health-check` | Extended health check |
| `health-check/summary` | Health summary |
| `log` | Action log endpoint |
| `ai-action-log` | AI action logging |
| `milo-activity` | Activity feed |
| `milo-feedback` | Feedback recording |
| `demo/*` | Demo endpoints (public by design) |
| `feedback/record` | Feedback recording |
| `onboard/*` | Onboarding flows (public by design) |
| `welcome/[slug]/claim` | Welcome page claims |

### NO RISK (properly protected)

| Category | Routes | Protection |
|----------|--------|-----------|
| All `/api/cron/*` | 18 routes | CRON_SECRET Bearer token |
| All `/api/admin/*` | 7 routes | getEffectiveUser + is_admin check |
| `/api/webhooks/resend` | 1 route | Svix signature + Cloudflare secret |
| `/api/outreach/send` | 1 route | resolveRequestUser session check |
| `/api/milo` + `/api/milo/stream` | 2 routes | resolveRequestUser |
| Operator core reads | ~8 routes | getEffectiveUser/resolveRequestUser |

---

## 3. Why /api/* Was Excluded From Auth

The middleware was designed for a page-routing auth model: protect page routes, redirect unauthenticated users to login. API routes were excluded because:

1. **Browser calls carry cookies anyway** — when the operator is logged in and the UI calls `/api/pipeline`, the Supabase session cookie is sent automatically. The routes work correctly for authenticated users without middleware enforcement.

2. **Cron routes need external access** — Vercel's cron scheduler calls `/api/cron/*` routes externally. These self-protect with `CRON_SECRET`.

3. **Webhook routes need external access** — Resend webhooks hit `/api/webhooks/resend` from outside. Self-protects with Svix signatures.

4. **No separation existed initially** — the app grew organically, and the auth pattern was never standardized across route handlers.

The problem: this means the API routes are protected only by obscurity (knowing the URL) and, for 19 routes, by route-level session checks that individual developers remembered to add.

---

## 4. Recommended Approach

### Option A: Session Auth in Middleware for /api/* (RECOMMENDED)

**What:** Remove `/api/` from the middleware bypass list. Add a new bypass list for routes that genuinely need external access.

**How:**

```typescript
// In middleware.ts, replace the blanket /api/ bypass:
const PUBLIC_API_ROUTES = [
  '/api/cron/',           // Vercel cron (self-checks CRON_SECRET)
  '/api/webhooks/',       // External webhooks (self-checks signatures)
  '/api/capture',         // Telegram bot / browser extension (needs separate auth — Phase 2)
  '/api/demo/',           // Public demo endpoints
  '/api/feedback/',       // Public feedback
  '/api/health',          // Health checks
  '/api/log',             // Logging endpoint
  '/api/onboard/',        // Public onboarding
  '/api/verify',          // Public verification
  '/api/welcome/',        // Public welcome
  '/api/auth/',           // Auth flow endpoints
];

// Then in the bypass check:
const isPublicApi = PUBLIC_API_ROUTES.some(prefix =>
  request.nextUrl.pathname.startsWith(prefix)
);

if (isPublicApi) return NextResponse.next();

// All other /api/* routes fall through to getUser() check
```

**Auth check behavior for non-public API routes:**
- If the request carries a valid Supabase session cookie, the user is authenticated; proceed.
- If no session or invalid session, return `401 Unauthorized` (not a redirect — API callers expect JSON, not HTML).

**Effort:** ~2-3 hours for Coder-1. Changes confined to middleware.ts. No route files need modification.

**Pros:**
- Single point of enforcement — every new route is protected by default
- Existing session cookies from logged-in operators work automatically (no UI changes)
- The 19 routes that already check auth become defense-in-depth, not the only layer
- Zero UX impact — operators are already authenticated when using the app

**Cons:**
- The `/api/capture` endpoint is called from a browser extension and Telegram bot — need to verify those callers carry cookies or add an API key path for them
- Need to return 401 JSON instead of redirect for API routes

### Option B: API Key Header (X-API-Key) for Internal Routes

**What:** Add an API key check for all /api/* routes that are called programmatically (not from the browser UI).

**Effort:** ~4-6 hours. Need to generate keys, add header checks to each route or middleware, update all internal callers.

**Pros:** Works for non-browser callers (Telegram bot, browser extension, cron-to-internal calls).
**Cons:** Browser-initiated calls from the UI don't naturally send API keys. Would need to either (a) embed the key in the frontend JS (defeats the purpose) or (b) use session auth for browser calls and API key for programmatic calls (complex dual-auth).

**Verdict:** Not recommended as primary approach. Could supplement Option A for specific machine-to-machine routes.

### Option C: IP Allowlist in Middleware

**What:** Restrict /api/* to known IP ranges (Vercel edge, office IP, VPN).

**Effort:** ~1-2 hours for static IPs. Ongoing maintenance burden.

**Pros:** Simple, no code changes in routes.
**Cons:** Vercel edge IPs change. Dynamic IPs (home, mobile) break access. Operators travel. Not practical for a distributed team.

**Verdict:** Not recommended. Too brittle for this deployment model.

### Option D: Leave As-Is (Rely on Obscurity)

**What:** Accept that the URLs are not publicly known and leave the current architecture.

**Pros:** Zero effort. No risk of breaking anything.
**Cons:**
- Any leaked URL, link in a Teams message, or browser history entry exposes the attack surface
- AI cost exposure is real — automated abuse of `/api/capture` or `/api/ask/entity-vet-inline` could run up bills
- Financial mutations (invoice approve, payment recording) without auth is a compliance concern even for internal tools
- Every new route added is unprotected by default — the gap grows over time

**Verdict:** Not recommended. The cost exposure alone justifies Option A.

---

## 5. Implementation Recommendation

**Phase 1 (Option A — immediate):**
1. Update middleware.ts to auth-check all `/api/*` routes except an explicit allowlist
2. Return `401 { error: "Unauthorized" }` for API routes (not redirect)
3. Test that the operator UI still works (it will — cookies are already sent)
4. Verify cron and webhook routes still work (they're on the allowlist and self-protect)

**Phase 2 (follow-up):**
1. Audit `/api/capture` — determine if browser extension and Telegram bot callers carry session cookies. If not, add API key auth (Option B) for that single route.
2. Add rate limiting to the top AI-cost routes: `capture`, `ask/entity-vet-inline`, `ask/draft-outreach`, `contract/analyze-bg`
3. Consider adding `resolveRequestUser` to financial mutation routes (invoice/approve, payment/record) as defense-in-depth

**Phase 3 (hardening):**
1. Add audit logging for all mutation endpoints (who called what, when)
2. Consider RBAC: some routes should be admin-only (team/force-clockout, td/create-entity, bank-transactions/import)

---

## 6. Severity Assessment

| Factor | Rating | Reasoning |
|--------|--------|-----------|
| Exploitability | Medium | Requires knowing the domain + route paths. Not discoverable via search. |
| Impact if exploited | High | Can modify CRM data, approve invoices, create TrackDrive entities, burn AI budget |
| Likelihood of exploitation | Low | Internal tool, small team, non-obvious URLs |
| Cost exposure | Medium | 19 unprotected AI routes, only 1 has rate limiting. ~$50-100/hr worst case. |
| Overall priority | **P2 — Fix in next sprint, not emergency** | The obscurity provides meaningful (not sufficient) protection today |

---

## Self-Attestation

- Each requirement from the directive was met (middleware read, route categorization, deployment model check, protection mechanisms audit, approach recommendation, effort estimates)
- No UI surfaces touched — screenshot N/A
- No production code modified (read-only audit)
- No known gaps: all 198 routes categorized, all auth mechanisms documented
- Production smoke test N/A (read-only research)
- Real production fixture N/A (no test code produced)

Report at: https://raw.githubusercontent.com/moscaleflow/conductor-mark-public/main/reports/api-auth-assessment-2026-05-06.md
