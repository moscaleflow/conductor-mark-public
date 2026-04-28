# OMADe Engine Audit — Coder-3 Report

**Coder-3 Research | 2026-04-27**
**Directive:** #73
**Scope:** Ground-truth audit of existing Milo infrastructure across 6 repos for OMADe Creator CRM / Outreach Engine planning.

---

## Section 1: Auth & Access

### Auth pattern
- **Supabase Auth via `@supabase/ssr`.** Magic-link primary, password fallback, forgot-password flow.
- **Middleware:** `milo-for-ppc/src/middleware.ts` (160 lines). `createServerClient` from `@supabase/ssr`. Multi-host routing: justmilo.co → justmilo.app → tlp.justmilo.app. Unauthenticated → `/onboard` or `/login`. Authenticated without setup → `/setup`. Default landing → `/operator`.
- **Login page:** `milo-for-ppc/src/app/login/page.tsx` (407 lines). 3 modes: magic_link (default), password, forgot. Uses `signInWithOtp`, `signInWithPassword`, `resetPasswordForEmail`.
- **Auth callback:** `milo-for-ppc/src/app/auth/callback/page.tsx` (110 lines). Client-side. Captures hash tokens before React hydration. Supports implicit flow (access_token in hash) + PKCE flow (code in query). Posts to `/api/auth/set-session`.
- **Session setter:** `milo-for-ppc/src/app/api/auth/set-session/route.ts` (56 lines). `exchangeCodeForSession` (PKCE) or `setSession` (implicit). Sets cookies on response.
- **Auth server helper:** `milo-for-ppc/src/lib/auth-server.ts` (159 lines). `getEffectiveUser(req)` and `resolveRequestUser(req)`. Impersonation-aware: reads `milo_impersonating` cookie, admin-only. Query-param auth bypass removed (D-2026-04-25-008).
- **Auth context (client):** `milo-for-ppc/src/lib/auth-context.ts` (38 lines). `UserProfile` interface: `is_admin`, `roles`, `team`, `focus_areas`, `has_completed_onboarding`, `has_completed_walkthrough`, `has_completed_operator_tour`.

### Admin vs user separation
- `user_profiles.is_admin` boolean flag. Checked in middleware (line 111) and auth-server.ts.
- Admin routes: `/api/admin/impersonate`, `/api/admin/stop-impersonate`, `/api/admin/impersonation-state`, `/api/admin/users`, `/api/admin/skills`.

### Canonical API route auth pattern
```typescript
// auth-server.ts:72-78, used by all protected API routes:
const user = await resolveRequestUser(req);
if (!user.userId) return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
```

### Canonical service-role client
- `milo-for-ppc/src/lib/supabase-admin.ts` (28 lines). Lazy singleton via `getSupabaseAdmin()`. Proxy wrapper. `SUPABASE_SERVICE_ROLE_KEY` env var. `autoRefreshToken: false, persistSession: false`.

### Other repos
- **milo-outreach:** No `src/` directory. Build artifacts in `.next/` only. No auth layer auditable.
- **milo-engine:** Package library, no auth. Packages receive `supabase` client from consumer.
- **mysecretary:** No `src/` directory. `.next/` output only.
- **milo-starter:** Directory does not exist at expected paths.
- **milo-ops:** Frozen reference. Not audited for auth.

### Mobile auth
- Nothing in place. No mobile-specific routes, no React Native, no Capacitor, no PWA auth.

---

## Section 2: Supabase & Database

### Shared Supabase project: tappyckcteqgryjniwjg

### Table prefix groups
| Prefix | Tables |
|---|---|
| `crm_*` | crm_counterparties, crm_leads, crm_contacts, crm_activities |
| `analysis_*` | analysis_records, analysis_jobs |
| `signing_*` | signing_documents, signing_library |
| `negotiation_*` | negotiation_records, negotiation_rounds, negotiation_links |
| `onboarding_*` | onboarding_flows, onboarding_runs, onboarding_steps |
| `feedback_*` | feedback_signals |
| `ask_*` | ask_rate_limits |
| (none) | blacklist_entries, chat_logs, vet_results, prospect_pipeline, invoices, billing_schedules, bank_transactions, call_records, contacts, disputes, action_items, agreed_terms, campaign_routes, conversation_captures, ai_action_log, milo_activity_log, reconciliation_runs, buyer_reports, send_log, tenants, leads |

### crm_* confirmed taken (D34)
- crm_counterparties ✓, crm_leads ✓, crm_contacts ✓, crm_activities ✓

### All tables referenced in milo-for-ppc `.from()` calls (61 total)
action_items, admin_impersonation_log, agreed_terms, ai_action_log, alerts, analysis_records, ask_rate_limits, bank_transactions, billing_schedules, blacklist_entries, buyer_platforms, buyer_report_rows, buyer_report_schedule, buyer_report_templates, buyer_reports, call_records, campaign_qualifiers, campaign_routes, campaigns, chat_logs, contacts, contract_documents, conversation_captures, creative_submissions, crm_contacts, crm_counterparties, crm_leads, disputes, drip_campaigns, entity_knowledge, eod_reports, evaluation_queue, feedback_signals, invoices, milo_activity_log, milo_conversations, milo_decisions, milo_feedback, milo_knowledge, milo_learned_patterns, ops_error_log, partner_sync_runs, ping_posts, ping_source_map, prospect_pipeline, reconciliation_runs, reviewer_decisions, send_log, signing_documents, snoozed_items, support-screenshots, support_tickets, td_change_snapshots, td_sync_state, team_members, tenants, user_profiles, user_skills, vet_results

### Confirmed existing tables (200 response on REST probe)
crm_counterparties, crm_leads, crm_contacts, crm_activities, blacklist_entries, analysis_records, analysis_jobs, signing_documents, signing_library, negotiation_records, negotiation_rounds, negotiation_links, onboarding_flows, onboarding_runs, onboarding_steps, feedback_signals, chat_logs, ask_rate_limits, vet_results, prospect_pipeline, invoices, billing_schedules, bank_transactions, call_records, contacts, disputes, action_items, agreed_terms, campaign_routes, conversation_captures, ai_action_log, milo_activity_log, reconciliation_runs, buyer_reports, send_log, tenants, leads

### Confirmed NOT existing (404)
conversations, daily_validation_runs, publisher_daily_reports, sequences, sequence_steps

### Supabase client setup
- **Browser:** `milo-for-ppc/src/lib/supabase-browser.ts` (21 lines). Singleton `createBrowserClient` from `@supabase/ssr`. Uses `NEXT_PUBLIC_SUPABASE_URL` + `NEXT_PUBLIC_SUPABASE_ANON_KEY`.
- **Re-export:** `milo-for-ppc/src/lib/supabase.ts` (11 lines). Re-exports browser singleton as `supabaseBrowser`.
- **Admin (server):** `milo-for-ppc/src/lib/supabase-admin.ts` (28 lines). Lazy singleton + Proxy.
- **Middleware:** Inline `createServerClient` in middleware.ts (lines 54-73). Cookie get/set handlers.

### RLS policy state (sampled 5 tables)
| Table | RLS Enabled | Policies |
|---|---|---|
| user_profiles | Yes | 4 policies (read own, update own, insert own, admin read all) + SECURITY DEFINER function for recursion fix |
| ping_posts | Yes | anon_read_all |
| partner_sync_runs | Yes | service role full access |
| action_items | Yes | wide-open (anon read/insert/update/delete) |
| chat_logs / ask_rate_limits | No | Explicitly service-role-only |

### Table overlap with target patterns
| Target | Existing | Notes |
|---|---|---|
| campaigns | `campaigns` EXISTS | billing_type, status fields. Used in alerts/generate route |
| contacts/people | `contacts` + `crm_contacts` both EXIST | `contacts` standalone, `crm_contacts` via @milo/crm |
| messages | Does NOT exist | Outreach emails → `send_log` + `ai_action_log` |
| templates | No standalone table | Template logic in code (`TemplateRegistry`) + `buyer_report_templates` |
| attachments | Nothing in place | |
| sales/orders | Nothing in place | Revenue via `invoices` |
| webhooks_received | Nothing in place | `td_change_snapshots` logs external change data |

### Migration workflow
- **milo-for-ppc:** `supabase/migrations/` — 75 SQL files, YYYYMMDD timestamps
- **milo-engine:** Per-package `packages/{pkg}/migrations/` — sequential (00001, 00002...)
- **milo-outreach:** `supabase/migrations/` — 4 files (00001-00004)

### Column naming convention
- snake_case throughout. Confirmed across migrations and `.from()` queries.

### Audit/log table patterns
| Table | Purpose |
|---|---|
| `ai_action_log` | Most API routes log here: action_type, entity_name, details (JSONB), performed_by, confidence |
| `milo_activity_log` | Broader activity logging |
| `chat_logs` + `chat_logs_archive` | Conversation history, 90-day archival |
| `send_log` | Outreach email tracking |
| `ops_error_log` | Error logging via `logOpsError()` |
| `admin_impersonation_log` | Impersonation audit |
| `signing_documents.audit_trail` | JSONB array of AuditEvent objects |
| `archive_job_log` | Tracks chat_logs archival jobs |

---

## Section 3: Edge Functions & Cron

### Supabase Edge Functions
Located at `milo-for-ppc/supabase/functions/`:
1. **`archive-chat-logs/index.ts`** — Deno runtime. Moves chat_logs rows >90 days old to chat_logs_archive in batches of 1000. Tracks in `archive_job_log`.
2. **`contract-analyze/index.ts`** — Deno runtime. Receives prompt + contract text from Vercel app, calls Anthropic Claude (claude-opus-4-20250514, raw fetch, NOT @milo/ai-client), parses JSON, stores in `contract_documents.analysis_result`. JSON repair logic for truncated responses.

**milo-engine:** No `supabase/functions/` directory.

### Vercel Cron (milo-for-ppc) — 11 jobs in vercel.json
| Path | Schedule |
|---|---|
| /api/cron/sync | */15 * * * * |
| /api/cron/reconciliation | 0 13 * * * |
| /api/cron/morning-assignments | 0 14 * * * |
| /api/cron/daily-validation | 0 13 * * * |
| /api/cron/health-check | 0 * * * * |
| /api/cron/billing-prepare | 0 13 * * 1-5 |
| /api/cron/td-change-detection | */15 * * * * |
| /api/cron/evaluate | */5 * * * * |
| /api/cron/detect-disputes | 15 * * * * |
| /api/cron/mediarite-xref | 0 14 * * * |
| /api/cron/demo-cleanup | 0 * * * * |

12 cron route dirs total (includes partner-sync not in vercel.json).

### Vercel Cron (milo-outreach) — 2 jobs
| Path | Schedule |
|---|---|
| /api/cron/research | 0 13 * * 1 (weekly Monday) |
| /api/cron/check-water-mark | 0 */6 * * * |

### Canonical cron route pattern
```typescript
// From health-check/route.ts:
export const maxDuration = 60;
// Verifies: authorization header against Bearer ${process.env.CRON_SECRET}
// Calls internal API, logs to ai_action_log, returns JSON
```

### Background job / queue
- `evaluation_queue` table — jobs inserted by triggers, processed by `/api/cron/evaluate` every 5 min.
- No external queue (no Bull, no SQS, no Inngest). All async: Vercel cron → API route → Supabase.
- contract-analyze offloaded to Supabase Edge Function to dodge Vercel timeout.

### maxDuration values in use
| Timeout | Routes |
|---|---|
| 15s | classify-file |
| 30s | refine-redline, explain-further |
| 60s | most crons, reconciliation, mediarite |
| 120s | demo/analyze, contract/analyze-proxy, contract/process, ask, billing-prepare, evaluate |
| 300s | milo/stream, sync, contract/analyze-bg |

### Cold start / warm-up
- Nothing in place. No warm-up routes, no provisioned concurrency.

---

## Section 4: AI / Claude Integration

### @milo/ai-client v0.1.3
Package at `milo-engine/packages/ai-client/`

**Public API:**
- `createAIClient(options?)` → `AIClient` with `.call(prompt, opts)` and `.raw` (Anthropic SDK)
- `callClaude(prompt, options, client?)` → `Promise<string>`
- `callClaudeExtended(prompt, options, client?)` → `Promise<CallClaudeResult>` (includes usage stats, model, stopReason)
- `callClaudeStream(prompt, options, onChunk, client?)` → `Promise<StreamResult>`
- `callClaudeVision(prompt, options, client?)` → `Promise<string>`
- `withFastInsight(fn, fallback, timeoutMs?)` — race pattern, 4s default
- `buildVoiceSystemPrompt(voice, baseInstructions?)` → string
- `TIER`: `{ judgment: 'claude-opus-4-7', structured: 'claude-sonnet-4-6', classify: 'claude-haiku-4-5', default: 'claude-sonnet-4-6' }`
- `AIError` class with status and retryable flag

**Types exported:** AIClientOptions, CallClaudeOptions, CallClaudeVisionOptions, CallClaudeResult, FastInsightOptions, StreamResult, UsageStats, TierName, ModelId, VoiceConfig

### MODEL-POLICY.md
`milo-engine/docs/MODEL-POLICY.md` — Locked 2026-04-19. Anthropic-only. 4 tiers: judgment (Opus), structured (Sonnet), classify (Haiku), default (Sonnet). Raw model overrides allowed but logged.

### Token/cost tracking
- Usage stats returned by `callClaudeExtended` and `callClaudeStream`: input_tokens, output_tokens, cache_read_input_tokens, cache_creation_input_tokens.
- `/api/ask/route.ts` (lines 245-260, 614-655): `aggregateUsage()` sums token counts across multiple Claude calls. Stored in `chat_logs` table.
- **No dollar-cost calculation layer.**

### Prompt caching
- `cacheSystemPrompt: boolean` option in CallClaudeOptions. Wraps system prompt in `cache_control: { type: 'ephemeral' }`.
- Supported in `callClaude` (client.ts:30-38) and `callClaudeStream` (streaming.ts:55-65).
- Cache hit/miss tracked via `cache_read_input_tokens` and `cache_creation_input_tokens`.

### Tool-use patterns
- **Anthropic web_search tool:** `/api/ask/route.ts` lines 509, 817 — `tools: [{ name: 'web_search', type: 'web_search_20250305', max_uses: 3-5 }]`
- **Custom operator tools:** 22 tool definitions in `milo-for-ppc/src/lib/tools/`. Each follows `_tool-contract.ts` interface: name, description, inputSchema (JSON Schema), execute function. Registry at `_registry.ts`.

### Rate limiting / retry
- `withRetry` in retry.ts: 3 retries, 15s timeout, exponential backoff (1s, 2s, 4s). Retries on 429, AbortError, retryable AIErrors.
- `ask_rate_limits` table for per-IP rate limiting of /ask endpoint.

### System prompt locations
Dedicated files:
- `milo-for-ppc/src/lib/contract-analysis-prompt.ts`
- `milo-for-ppc/src/lib/milo-support-prompt.ts`
- `milo-for-ppc/src/lib/milo-prompt.ts`
- `milo-for-ppc/src/lib/ppc/system-prompt.ts`
- `milo-for-ppc/src/lib/ask/classifier-prompt.ts`
- `milo-for-ppc/src/lib/ask/prompts/` (directory: vet.ts, vet-extraction.ts, etc.)

Plus inline system prompts in 13 API route files.

---

## Section 5: Email & Messaging

### Resend integration (3 callsites in milo-for-ppc)
1. **`/src/app/api/demo/capture/route.ts`** (lines 70-100) — Dynamic import. From: `process.env.RESEND_FROM_EMAIL || 'Milo <milo@justmilo.app>'`. Demo lead capture notification.
2. **`/src/app/api/outreach/send/route.ts`** (lines 41-110) — Dynamic import. From: `process.env.RESEND_FROM_EMAIL || 'The Lead Penguin <support@theleadpenguin.com>'`. Reply-to: `support@theleadpenguin.com`. Logs to `ai_action_log` + `send_log`. Updates contact `last_contact_at`.
3. **`/src/app/api/operator/pill/route.ts`** (line 396) — Reference only (action prompt suggesting resend follow-up).

### Resend in @milo/contract-signing
- `milo-engine/packages/contract-signing/src/email.ts` (76 lines). `ResendEmailProvider` class. Default from: `contracts@{company-name}.com`. Inline HTML template. `resend` listed as package dependency.

### Verified sender domains
- `milo@justmilo.app` (demo capture)
- `support@theleadpenguin.com` (outreach send, reply-to)
- `contracts@{dynamic}.com` (contract-signing, derived from companyInfo.name)

### milo-outreach
- No `src/` directory with application code. Cannot audit Resend usage.

### Inbound email parsing
- Nothing in place. No webhook endpoints for receiving email. No Resend inbound, no Mailgun, no SendGrid.

### Email templates / template engine
- No dedicated engine. All emails use inline HTML strings.
- `ResendEmailProvider` (contract-signing) has hardcoded HTML template (lines 32-74).
- Outreach send route builds HTML inline: `body.replace(/\n/g, '<br/>')`.

### Bounce/complaint handling
- Nothing in place. No Resend webhook for bounces, no complaint processing.

### Email warm-up tracker
- Nothing in place.

### SMS
- Nothing in place. No Twilio, no SMS integration in any repo.

### Notification to Mark
- **Teams:** `milo-for-ppc/src/lib/teams-client.ts` (247 lines). Full messaging via PPCRM Edge Function relay.
  - `sendToChannel(channel, message)` — 6 channels: operations, alerts, billing, sales, huddle, onboarding
  - `sendCard(channel, card)` — Adaptive Card messages
  - `sendAlert(type, data)` — critical/warning/info/opportunity to #alerts
  - `sendBriefing(summary)` — morning briefing to #operations
  - `sendDM(aadObjectId, message)` — direct message
  - `ping()` — connectivity check
  - Auth: `PPCRM_MESSAGING_URL`, `PPCRM_MESSAGING_KEY`, `PPCRM_SUPABASE_ANON_KEY`

---

## Section 6: Social / Outreach Automation

### Apify
- Zero grep hits for "apify" in `milo-for-ppc/src/`, `milo-outreach/`, `milo-engine/packages/`, `mysecretary/`. Only hit: `node_modules/ai/docs/` (Vercel AI SDK docs mentioning Apify as a third-party integration).

### Existing scrapers
- Nothing in place.

### LinkedIn integration
- **Data model only, no API integration.**
  - `pipeline-board/page.tsx:39` — platform type `'email' | 'linkedin' | 'teams'`
  - `contacts` table has `linkedin_url` field
  - `blacklist_entries` has `linkedin_urls` JSONB column (@milo/blacklist v0.2.0)
  - `leads/import/route.ts` — maps LinkedIn column from CSV imports
  - `outreach/draft/route.ts:145` — drafts LinkedIn connection messages (<300 chars)
  - `research-prospect.ts:272,298` — accepts LinkedIn URL as reference, explicitly states "NEVER scraped"
  - `capture/route.ts:86` — classifies screenshots as 'linkedin' category
  - `blacklist/screen/route.ts:10` — screens against linkedin_url
  - `fetchLinkedInActivity()` in executive route (line 998) reads from `send_log`, NOT LinkedIn API

### Browser automation
- Nothing in place. No Playwright, Puppeteer, or Browserbase in any `src/` directory.

### Handle/profile validation
- `@milo/blacklist` has `normalizeLinkedInUrl()` — handles /in/, /pub/, /profile/view variants + query param stripping. Screening only, not live validation.

### Scraping
- Explicitly absent by design. `research-prospect.ts:298`: "NEVER scrapes LinkedIn."

---

## Section 7: E-Signature & Contracts

### @milo/contract-signing v0.1.0
Package at `milo-engine/packages/contract-signing/`

Dependencies: `@supabase/supabase-js`, `resend`

**SigningClient public API (types.ts:257-288):**
- `createDocument(opts)` → SigningDocument
- `getDocument(id)`, `getDocumentByToken(token)`, `listDocuments(opts?)`
- `captureView(token, viewerInfo)` — records first view with IP/UA
- `submit(token, submission)` — signing with consent, SHA-256 hashing, webhook fire, email confirmation
- `decline(token, reason, ip?)`, `void(documentId, reason?)`
- `signPayload(body)`, `verifySignature(body, sig, timestamp)` — HMAC-SHA256 webhook signing
- `renderDocument(documentType, templateData, signerData?)` — template rendering
- `generateCertificate(documentId)` — Certificate of Execution HTML
- `resolveShortCode(code)` — short URL resolution
- Library CRUD: `uploadLibraryDocument`, `getLibraryDocument`, `listLibraryDocuments`, `archiveLibraryDocument`

Source files: client.ts, documents.ts, tokens.ts, audit.ts, webhooks.ts, templates.ts, email.ts, certificate.ts, short-url.ts, tax-routing.ts, types.ts

Config via `SigningClientConfig`: supabase client, tenantId, webhookSecret, baseUrl, resendApiKey, templates, companyInfo, optional emailProvider/tokenExpirationDays/storageBuckets

### Third-party e-sig SDKs
- Nothing in place. No DocuSign, HelloSign, PandaDoc, Adobe Sign.

### PDF generation
- **No server-side PDF generation.** No react-pdf, no puppeteer-pdf.
- `/api/invoices/[id]/pdf/route.ts` (175 lines) — generates HTML invoice, `Content-Disposition: inline`, auto-triggers `window.print()` for browser "Save as PDF".
- `pdf-parse` in `/api/ask/route.ts:233` — for **reading** uploaded PDFs only (text extraction).
- Certificate output is HTML only (`certificate.ts generateCertificateHtml`).

### Supabase Storage buckets
- @milo/contract-signing defines 3: `contract-originals`, `finalized-contracts`, `partner-documents` (client.ts:28-32)
- `client.ts:180-186` — `supabase.storage.from(buckets.documents).upload()` for library docs
- `/api/cron/demo-cleanup/route.ts:89` — `supabase.storage.from(bucket).remove(paths)` for cleanup
- milo-engine contract-analysis migration creates `contract-originals` bucket

### Audit trail
- JSONB array in `signing_documents.audit_trail`. Events: document_created, document_viewed, consent_accepted, document_signed, document_declined, document_voided, webhook_sent, webhook_failed, email_sent.
- `appendAuditEvent()` pure function in audit.ts — immutable append.
- SHA-256 hashing of signature data and rendered document for tamper detection.

### Template variable substitution
- `TemplateRegistry` class in templates.ts. `TemplateDefinition` objects with `render(ctx)` and optional `renderSigned(ctx)`. Code-driven (functions), not mustache/handlebars.

### Tax routing
- `selectTaxDocType(partner)` in tax-routing.ts — returns 'w9' (US), 'w8ben' (foreign individual), 'w8bene' (foreign entity).

---

## Section 8: Payments & Affiliates

### Stripe
- Nothing in any repo. Zero grep hits for stripe/Stripe/STRIPE across milo-for-ppc/src, milo-outreach, milo-engine/packages. No Stripe SDK in any package.json.

### Shopify
- Nothing in any repo. Zero grep hits for shopify/Shopify/SHOPIFY across all repos.

### Affiliate/commission tracking
- **Code-level only, no dedicated tables.** `fetchCommissionTracker()` in `/src/app/api/operator/executive/route.ts` (line 1015). Derives margin from `invoices` table: groups by entity, sums receivable vs payable, calculates margin.
- `commission_tracker` is a dashboard widget type in `dashboard-v2-types.ts` (line 29).
- Contract analysis prompt (line 323) checks for sub-affiliate compliance gaps.

### Discount/coupon code generation
- Nothing in place.

---

## Section 9: Shipping & Fulfillment

Nothing in place. Zero grep hits for Shippo, EasyPost, or ShipStation across all repos. No fulfillment workflow, no address validation utility.

---

## Section 10: UI / Frontend

### Routing structure (milo-for-ppc/src/app/ — 2 levels)
**Pages:** ask, auth/callback, c/[slug], campaigns, contract-review/[id], disputes, eod-reports, feedback, invoices, jen, login, malvin, observatory, onboard, operator, partners, ping-post, pings, pipeline, pipeline-board, proof, reconciliation, reports, review, setup, support, tiffani, welcome/[slug]
**API:** 68 route directories (accuracy, action-items, admin, agreed-terms, alerts, ask, auth, bank-transactions, billing-schedules, blacklist, briefing, buyer-reports, buyers, campaigns, capture, captures, contacts, contract, creative, cron, daily-validation, dashboard, dashboard-v2, demo, disputes, entities, eod, feedback, health, health-check, invoices, leads, live-stats, log, malvin, matching, milo, milo-activity, milo-feedback, morning-assignments, notifications, observatory, onboard, operator, outreach, partners, ping-post, pings, pipeline, ppcrm, predictions, profile, publisher-onboarding, publishers, reconciliation, research, review, rtb, stats, support, sync, td, td-changes, team, team-members, walkthrough, welcome)

### Component library
- **No shadcn/ui.** No `components.json`, no `@/components/ui/` directory.
- **No Tailwind config file.** Tailwind v4 via `@import "tailwindcss"` in globals.css. No tailwind.config.ts.
- **Custom components (30 files):** AdminExecutiveCards, AuthProvider, BackLink, BatchEODModal, BootingUpSequence, BriefingPanel, ChatInterface, ConversationFeed, ConversationModal, EODModal, FeedbackThumbs, ImpersonationBanner, JenView, LayoutShell, LeadImportModal, MalvinView, NotificationModal, OperatorTour, QuickReplies, StatBar, SupportBubble, SupportPanel, ThemeProvider, ThemeToggle, TiffaniView, Top5Frame, Topbar, TypedLine, UniversalDropZone, admin/

### Design tokens
- Dark theme: `#0a0a0f` background in `layout.tsx`. SF Pro Display font family.
- Tailwind v4 — colors defined inline via utility classes, not in a config file.
- Custom CSS animations in globals.css: fadeSlideIn, softPulse, bounce. Hidden scrollbars. Font smoothing.

### Reusable UI patterns
| Pattern | Location | Notes |
|---|---|---|
| Drawers/sheets | PillDrawer pattern in operator views | Custom, not shadcn Sheet |
| Toast | Local `useState<string\|null>` per component | No global toast system |
| Tables | Custom per-page | No shared table primitive |
| Forms | Zod validation (57 imports across src/) | No react-hook-form or formik |
| Realtime subs | ConversationFeed.tsx uses Supabase Realtime `postgres_changes` | Observatory page |
| Loading states | Per-component | No shared skeleton/spinner |
| Cards | Custom per-page | No shared card primitive |

### Server actions vs API routes
- No `'use server'` directives found. All mutations go through API routes.

### Global state management
- No zustand, jotai, recoil, or Redux. React Context only: AuthProvider, ThemeProvider.

---

## Section 11: Webhooks & Integrations

### Webhook endpoints exposed
| Endpoint | Receives | Repo |
|---|---|---|
| `/api/td/*` | TrackDrive call/campaign data | milo-for-ppc |
| `/api/sync/*` | TrackDrive sync data | milo-for-ppc |
| `/api/td-changes/*` | TrackDrive change detection | milo-for-ppc |

### Webhook signature verification
- `@milo/contract-signing` has `signPayload(body)` + `verifySignature(body, sig, timestamp)` — HMAC-SHA256. Used for outbound webhook signing to consumers.
- **No generic inbound webhook signature verification utility.**
- `ONBOARDING_WEBHOOK_SECRET` env var used in `milo-for-ppc/src/lib/onboarding-client.ts:36` — passed to @milo/onboarding client config. Default: `'tlp-onboarding'`.

### Webhook retry / dead-letter queue
- Nothing in place.

### Active incoming webhooks
| Source | Status |
|---|---|
| TrackDrive | Active (sync, td, td-changes routes) |
| Shopify | Nothing |
| Stripe | Nothing |
| Resend | Nothing (no bounce/event webhooks) |

---

## Section 12: Observability & Errors

### Logging stack
- **No Sentry.** Zero grep hits for sentry/Sentry.
- **No Logtail.** Zero grep hits for logtail/Logtail.
- **No Datadog.** Zero grep hits.
- **Custom:** `ops_error_log` table + `logOpsError()` in `milo-for-ppc/src/lib/ops-error-log.ts`. Fire-and-forget inserts. Used in 10+ API routes (ask, classify-file, refine-redline, etc.).
- `ai_action_log` table for action-level logging.

### Error notification to Mark
- Teams client `sendAlert('critical', ...)` fires to #alerts channel. Used in cron failures and critical errors.

### Performance monitoring
- Nothing in place. No Web Vitals tracking, no Vercel Speed Insights, no custom latency tracking beyond `latency_ms` in chat_logs.

### Database query monitoring
- Nothing in place.

---

## Section 13: Deployment & Env

### Vercel projects (relevant to Milo)
- milo-for-ppc (primary product)
- milo-outreach
- `vercel env ls` returned empty output — env vars confirmed via `.env.example` instead.

### Branch deploy / preview
- Standard Vercel preview deployments on PRs. No custom preview configuration.

### Env var inventory (milo-for-ppc .env.example — 39 keys)
```
ANTHROPIC_API_KEY
APP_URL
ASK_CLASSIFIER_AUTO_SUBMIT_THRESHOLD
BLACKLIST_SUPABASE_SERVICE_KEY
BLACKLIST_SUPABASE_URL
COMPANY_NAME
CONVOQC_API_KEY
CONVOQC_API_URL
CRON_SECRET
DEMO_CLEANUP_DRY_RUN
DEMO_CLEANUP_ENABLED
GOOGLE_SHEETS_CLIENT_EMAIL
GOOGLE_SHEETS_MEDIARITE_SHEET_ID
GOOGLE_SHEETS_PRIVATE_KEY
IP_HASH_SALT
LIVE_STATS_API_KEY
MILO_ENGINE_SUPABASE_SERVICE_KEY
MILO_ENGINE_SUPABASE_URL
MOP_API_URL
MOP_BASE_URL
MOP_BOT_API_KEY
MOP_SUPABASE_ANON_KEY
MOP_SUPABASE_URL
NEXT_PUBLIC_APP_URL
NEXT_PUBLIC_SITE_URL
NEXT_PUBLIC_SUPABASE_ANON_KEY
NEXT_PUBLIC_SUPABASE_URL
ONBOARDING_WEBHOOK_SECRET
PPCRM_BASE_URL
PPCRM_MESSAGING_KEY
PPCRM_MESSAGING_URL
PPCRM_SUPABASE_ANON_KEY
RESEND_API_KEY
RESEND_FROM_EMAIL
SUPABASE_ACCESS_TOKEN
SUPABASE_SERVICE_ROLE_KEY
TD_API_AUTH
TD_API_BASE_URL
TENANT_ID
VET_CACHE_DAYS
```

### Secret rotation policy
- Nothing in place. CREDENTIALS-MAP.md documents locations (D105) but no rotation schedule.

---

## Section 14: Conflicts & Patterns Not to Break

### Architectural locking decisions (from DECISION-LOG.md, 145+ decisions)
| Decision | Lock |
|---|---|
| D4 | Magic-link auth only — all verticals |
| D5 | milo-starter locked at b144121 |
| D11 | Thesis 2.5 locked — starter template + extracted infra |
| D14 | milo-outreach v0 scope locked |
| D28 | Anthropic-only model policy locked for milo-engine |
| D30 | Anthropic-only policy enforced across all consumers |
| D67 | Option C locked — denormalized display names |
| D69 | Milo v1 Funnel Vision locked |
| D82 | Feedback surface — write-only via Supabase Dashboard |
| D105 | ANTHROPIC_API_KEY drift fixed, CREDENTIALS-MAP.md canonical |
| D106 | Pill voice consistency audit locked |
| D114 | Directive order locked, Coder-1 cleared |

### Drift guard patterns (STRATEGY-CLAUDE-DRIFT-GUARDS.md)
- 23 patterns documented. 30 guard-related lines.
- Key guards: no horizontal engines yet, no monorepo, no password auth, Anthropic-only, no billing type aggregation without differentiation, timezone helper usage, no direct service-role key in client code.

### Currently fragile areas
- `vet_results.chat_log_id` backfill is broken — all NULL (see Coder-3 bleed audit report).
- `response_text` truncated at 10,000 chars in chat_logs — loses multi-entity vet data.
- `evaluation_queue` processed every 5 min with no dead-letter — stuck jobs block queue.
- 3-gate outreach safety depends on `MILO_OUTREACH_ALLOW_REAL_SENDS` env var that doesn't exist in .env.example.

### In-flight work overlapping with creator outreach
- D-vet entity-mixing fix (9ce9c26) — prompt isolation + associated_entities.
- Pipeline board UX improvements (D-010, D-011, D-014).
- Outreach send route at `/api/outreach/send/` — active.
- Fuzzy search (D-015) — may affect entity lookup patterns.
- Resend outreach send wiring (D-018) — directly overlaps.

---

## Reusable as-is

- **@milo/ai-client v0.1.3** — callClaude, callClaudeExtended, callClaudeStream, callClaudeVision, withFastInsight, buildVoiceSystemPrompt, TIER constants, AIError, VoiceConfig, prompt caching, retry logic
- **@milo/blacklist v0.2.0** — entity screening, normalizeLinkedInUrl(), ip_address + linkedin_url screening, 56 tests
- **@milo/contract-signing v0.1.0** — full document lifecycle (create, view, sign, decline, void), HMAC webhooks, template registry, audit trail, certificate generation, tax routing, ResendEmailProvider, Supabase Storage integration
- **@milo/contract-analysis** — extraction, scoring, post-processing, jurisdiction, text-merge, revision-compare
- **@milo/contract-negotiation** — state machine, rounds, links, export
- **@milo/crm** — counterparties, leads, contacts, activities (crm_* tables)
- **@milo/onboarding** — flows, runs, steps, DAG-capable, flow_snapshot
- **@milo/feedback** — feedback_signals table, write-only
- **Supabase Auth + @supabase/ssr** — magic-link flow, middleware pattern, session management, impersonation
- **supabase-admin.ts** — lazy singleton service-role client with Proxy
- **auth-server.ts** — getEffectiveUser(), resolveRequestUser(), impersonation-aware
- **Resend email sending** — dynamic import pattern, 3 verified sender domains
- **Teams notification client** — 6 channels, alert/briefing/DM/card, relay through PPCRM Edge Function
- **Vercel cron pattern** — CRON_SECRET bearer auth, maxDuration, JSON response
- **ops_error_log + logOpsError()** — fire-and-forget error logging
- **ai_action_log pattern** — action-type structured logging
- **send_log pattern** — outreach email tracking
- **evaluation_queue + cron processor** — simple job queue via Supabase table + Vercel cron
- **Anthropic web_search tool integration** — web_search_20250305, max_uses control
- **Custom operator tool framework** — _tool-contract.ts interface, _registry.ts, 22 existing tools as examples
- **snake_case column convention** — universal across all tables
- **YYYYMMDD migration file convention** — milo-for-ppc pattern
- **Supabase Storage buckets** — contract-originals, finalized-contracts, partner-documents

## Green-field for OMADe

- **Apify / scraping integration** — zero existing code, zero accounts
- **Instagram API / TikTok API** — nothing in place
- **Shopify integration** — zero code, zero SDK
- **Stripe / payment processing** — nothing in any repo
- **Affiliate code generation** — nothing in place
- **Discount/coupon system** — nothing in place
- **Shipping / fulfillment** (Shippo, EasyPost, ShipStation) — nothing in place
- **Address validation** — nothing in place
- **SMS / Twilio** — nothing in place
- **Inbound email parsing** — no Resend inbound, no webhook for receiving email
- **Email bounce/complaint handling** — no Resend event webhooks
- **Email warm-up tracking** — nothing in place
- **Email template engine** — all emails are inline HTML strings
- **Browser automation** (Playwright, Puppeteer, Browserbase) — nothing in place
- **LinkedIn API integration** — data model only, explicitly no scraping, no send capability
- **Post-detection system** — nothing in place (no social monitoring)
- **Sales attribution tracking** — nothing beyond `invoices` table margin calc
- **PDF generation (server-side)** — no react-pdf, no puppeteer-pdf; only browser print
- **Mobile auth / PWA** — nothing in place
- **Global toast/notification UI** — each component rolls its own useState
- **Shared table/card/form components** — no component library; all custom per-page
- **Server-side rendering of email templates** — no MJML, no react-email
- **Webhook retry / dead-letter queue** — nothing in place
- **Inbound webhook signature verification (generic)** — only @milo/contract-signing has outbound HMAC
- **Performance monitoring / APM** — no Sentry, Logtail, Datadog, Vercel Speed Insights
- **Dollar-cost tracking for AI usage** — tokens logged but not costed
- **Creator-specific CRM tables** (creator_*, omade_*) — namespace available, nothing exists
- **Sequence/drip automation tables** — `sequences` and `sequence_steps` confirmed 404
- **Cold start / warm-up for serverless** — nothing in place
- **Secret rotation policy** — CREDENTIALS-MAP.md exists but no rotation schedule

---

Cross-references: Directive #73, DECISION-LOG.md, CREDENTIALS-MAP.md, STRATEGY-CLAUDE-DRIFT-GUARDS.md
