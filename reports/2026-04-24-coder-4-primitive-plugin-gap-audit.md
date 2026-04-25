# Primitive Plug-in Gap Audit — 8 @milo/* Packages

**Coder:** 4 (Ops)
**Date:** 2026-04-24
**Status:** COMPLETE — audit only, no code changes
**Decision:** 100

---

## Tool-reuse audit summary

| Primitive | Exported methods/types | Used in milo-for-ppc | Unused |
|---|---|---|---|
| @milo/ai-client | 11 exports | 4 used (callClaude, callClaudeVision, withFastInsight, TIER) | 7 unused (callClaudeStream, callClaudeExtended, createAIClient, buildVoiceSystemPrompt, AIError, callClaudeVision partially) |
| @milo/blacklist | 12 exports | 1 used (check via wrapper) | 11 unused (addEntry, removeEntry, restoreEntry, listEntries, getEntry, updateEntry, screenAgainst, normalizeCompanyName, normalizeLinkedInUrl, levenshtein, BLACKLIST_SEVERITIES) |
| @milo/contract-analysis | 14 exports | 3 used (createAnalysisEngine, BASE_RULES, + types) | 11 unused (detectJurisdiction, extractText, mergeDocument, buildFinalNegotiationState, scoreIssues, postProcessIssues, computeDocumentHash, uploadContract, compareRevisions, resolveRules, enforceMandatoryChecks) |
| @milo/contract-signing | 12 exports | 0 used (only referenced in demo-cleanup cron) | 12 unused |
| @milo/contract-negotiation | 5 exports | 0 used (only referenced in demo-cleanup cron) | 5 unused |
| @milo/crm | 8+ exports | 5 used (createCrmClient, getContact, listCounterparties, updateContact, listContacts) | 3+ unused (logActivity, listActivities, upsertLead) |
| @milo/feedback | 4 exports | 0 used (API route is raw Supabase, not via SDK) | 4 unused |
| @milo/onboarding | 4+ exports | 3 used (createOnboardingClient, startRun, getStatus) | 1+ unused (EventBus, completeStep) |

**Total:** 70+ exports across 8 primitives. ~16 used. ~54 unused.

---

## 1. @milo/ai-client

**What it does:** Central model calling with 4-tier policy (judgment/structured/classify/default), MODEL_QUIRKS table, prompt caching, streaming, tool use, vision.

**Where it's wired today:**
- `src/lib/anthropic.ts` — re-exports callClaude, callClaudeVision, withFastInsight
- `src/lib/models.ts` — re-exports TIER
- 18+ API routes via callClaude (demo/analyze, capture, eod/generate, briefing/morning, buyer-reports, creative/analyze, outreach/draft, + 10 lib/tools/*)
- `src/lib/reviewer-agent.ts` — callClaude for contract review

**Where it's NOT wired (critical gap):**
- `/ask` route (`src/app/api/ask/route.ts`) — **uses raw `new Anthropic()` directly**, bypassing ai-client entirely. No MODEL_QUIRKS protection, no tier policy, no caching, no upstream error logging.
- `/ask` classifier (`classifyInput()`) — raw Anthropic SDK call, not tier='classify'

**Plug-in opportunities:**

| Surface | Opportunity | Effort |
|---|---|---|
| /ask | Replace `new Anthropic()` with `callClaudeStream` / `callClaudeExtended` — gets MODEL_QUIRKS, tier policy, prompt caching, upstream error logging for free | Multi-file (route.ts + prompts) |
| /ask classifier | Replace raw `sdk.messages.create()` with `callClaude(tier='classify')` — enforces Haiku routing via tier table | Single-file |

- **HIGH:** /ask → callClaudeStream migration. Currently the most-used surface in the product bypasses all primitive safeguards. If Anthropic deprecates another param (Pattern 15 repeat), /ask breaks while /operator stays fine.
- **HIGH:** /ask classifier → tier='classify'. Ensures Haiku model selection stays centralized.

---

## 2. @milo/crm

**What it does:** Counterparties, leads, contacts, activities. Multi-tenant. find-or-create pattern.

**Where it's wired today:**
- `src/lib/crm-client.ts` wrapper — createCrmClient, listCounterparties, updateContact, listContacts
- `src/app/api/research/entity/route.ts` — getContact
- `src/app/api/demo/capture/route.ts` — createCrmClient for lead creation
- `src/lib/web-research.ts` — getContact for enrichment
- `src/lib/milo-queries.ts` — getContacts (raw Supabase, not via CRM SDK)
- `src/lib/milo-tools.ts` — getContacts via milo-queries

**Where it's NOT wired:**
- `/ask` — no CRM context lookup. When a user asks about an entity, /ask has no counterparty/contact history.
- `logActivity` / `listActivities` — never called. Activity logging is done via raw Supabase in various places.

**Plug-in opportunities:**

| Surface | Opportunity | Effort |
|---|---|---|
| /ask Vet pill | Pre-check entity against CRM before vet analysis — surface existing contract history, relationship status | Multi-file (route.ts + vet prompt) |
| /ask Sales pill | Pull counterparty context for drafting scenarios — existing relationship, past contracts, payment history | Multi-file |
| /operator | Replace raw Supabase activity logging with logActivity() — unified audit trail | Multi-file (scattered writes) |

- **MEDIUM:** /ask CRM context injection (Coder-2 reportedly in-flight on this)
- **LOW:** logActivity migration — works fine as raw Supabase today, migration is polish not function

---

## 3. @milo/blacklist

**What it does:** Entity screening with fuzzy matching, severity tracking, source attribution.

**Where it's wired today:**
- `src/lib/blacklist-client.ts` wrapper — only `check()` method used
- 7 consumer files via `screenEntity()` wrapper (capture, leads/import, blacklist/screen, entities/[name], publisher-onboarding, milo-queries, setup-new-buyer)

**Where it's NOT wired (critical gap):**
- `/ask` route — **uses raw Supabase `from('blacklist').select()`**, not `@milo/blacklist.check()`. No fuzzy matching, no severity, no source tracking. Just exact company_name match.
- `/ask` blacklist hit — no severity context. The primitive has BLACKLIST_SEVERITIES but /ask doesn't differentiate between "do not work with" and "proceed with caution."
- Admin operations (addEntry, removeEntry, listEntries) — no UI surface. Blacklist management is Supabase Dashboard only.

**Plug-in opportunities:**

| Surface | Opportunity | Effort |
|---|---|---|
| /ask | Replace raw Supabase query with `screenEntity()` — gets fuzzy match, severity, source context | Single-file |
| /ask response | Include severity in blacklist visual tell — red border for hard block, yellow for caution | Single-file |
| /operator | Blacklist admin pill — addEntry/removeEntry/listEntries for operators to manage without Dashboard | Multi-file + new pill |

- **HIGH:** /ask → screenEntity(). Raw query misses fuzzy matches (misspelled company names bypass the check entirely).
- **MEDIUM:** Severity-aware blacklist tells on /ask
- **LOW:** Blacklist admin pill — operators don't currently manage this

---

## 4. @milo/contract-analysis

**What it does:** 77-rule analysis engine with VerticalRulePack plugin, scoring, jurisdiction detection, text extraction, revision comparison.

**Where it's wired today:**
- `src/lib/ppc/engine.ts` — createAnalysisEngine + BASE_RULES for PPC vertical
- `src/lib/ppc/*.ts` — rules, patterns, profiles, overrides, display-config
- `src/components/contract/` — IssueCard, SeverityBadge, RoleSelector, IssueSidebar (types only)
- `src/lib/display-config.ts` — type imports

**Where it's NOT wired:**
- `detectJurisdiction()` — never called. Jurisdiction detection exists in the primitive but no surface uses it.
- `compareRevisions()` — never called. Revision comparison for redline diffs is unexercised.
- `uploadContract()` / `computeDocumentHash()` — never called from app code.

**Plug-in opportunities:**

| Surface | Opportunity | Effort |
|---|---|---|
| /ask Vet pill | Coder-1 wiring this NOW (in-flight). Full analysis engine behind Vet responses. | In-flight |
| /operator Contracts drawer | detectJurisdiction() on each contract — show jurisdiction badge | Single-file |
| /ask Vet | compareRevisions() when user drops a second version of same contract | Multi-file |

- **HIGH:** Vet → analysis engine (IN-FLIGHT, Coder-1)
- **MEDIUM:** Jurisdiction badge in Contracts drawer
- **LOW:** Revision comparison — requires multi-upload UX that doesn't exist yet

---

## 5. @milo/contract-signing

**What it does:** Full signing workflow — tokens, short codes, webhook verification, audit trail, email via Resend, certificate generation, template registry.

**Where it's wired today:**
- `src/app/api/cron/demo-cleanup/route.ts` — reference only (cleanup cascade)
- No active consumer. Zero methods called from app code.

**Where it's NOT wired:**
- Everywhere. The primitive is vendored but completely unwired. All signing functionality in milo-for-ppc is either MOP-era code or not implemented.

**Plug-in opportunities:**

| Surface | Opportunity | Effort |
|---|---|---|
| /ask Vet | "Send for signature" action after positive vet result — generateSigningToken + short code link | Multi-file + schema |
| /operator Contracts drawer | "Send for signature" button on contract rows — wire to createSigningClient | Multi-file |
| /c/[slug] route | resolveShortCode is available — signing review pages could be contract-specific | Already exists but hardcoded |

- **MEDIUM:** /operator "Send for signature" in Contracts drawer — the primitive has everything, the surface has the contract context
- **LOW:** /ask "send for signing" — /ask is no-auth, sending for signature implies operator action. Doesn't fit the public surface.
- **LOW:** /c/[slug] enhancement — would need signing flow redesign

---

## 6. @milo/contract-negotiation

**What it does:** Two-state negotiation model (draft → final), rounds with diff tracking, review links, agreement export.

**Where it's wired today:**
- `src/app/api/cron/demo-cleanup/route.ts` — reference only (cleanup cascade)
- No active consumer. Zero methods called from app code.

**Where it's NOT wired:**
- Everywhere. Like contract-signing, completely unwired.

**Plug-in opportunities:**

| Surface | Opportunity | Effort |
|---|---|---|
| /ask Vet | Part 2: redline link generation after vet flags issues. generateLink() for buyer review of AI-proposed changes. | Multi-file (Coder-1 will wire in Part 2) |
| /operator Disputes drawer | Negotiation rounds context on active disputes — "this entity has 3 open negotiation rounds" | Multi-file |

- **MEDIUM:** Vet Part 2 redline links (PLANNED, Coder-1)
- **LOW:** Disputes drawer negotiation context — disputes and negotiations are loosely related at best

---

## 7. @milo/onboarding

**What it does:** DAG-capable step engine with flows, runs, steps. Progress tracking, stalled step detection.

**Where it's wired today:**
- `src/lib/onboarding-client.ts` wrapper — createOnboardingClient, startRun, getStatus, getStatusByRun, findStalled
- `src/app/api/milo/route.ts` — getOnboardingStatusByCounterparty (Milo chat context)
- `src/app/api/ppcrm/route.ts` — startOnboarding
- `src/lib/publisher-onboarding.ts` — startPublisherOnboarding starts a run
- `src/app/api/cron/demo-cleanup/route.ts` — cleanup

**Where it's NOT wired:**
- `/ask` — no onboarding context. When a Buyer pill user asks about their setup progress, there's no data.
- `EventBus` — exported but never used. Event-driven step completion exists in the primitive but isn't consumed.
- `completeStep()` — never called from app code. Steps are created but never marked complete programmatically.

**Plug-in opportunities:**

| Surface | Opportunity | Effort |
|---|---|---|
| /ask Buyer pill | Pull onboarding run status for the entity being discussed — "your setup is 4/8 steps complete" | Multi-file |
| /operator | Stalled step alert pill — findStalled() powers an alert for operators showing who's stuck | Multi-file + new predicate |

- **MEDIUM:** /ask Buyer onboarding context (deferred until Buyer pill sees real usage)
- **MEDIUM:** Stalled step alerts — findStalled() already exists, would surface stuck publishers/buyers
- **LOW:** EventBus integration — no consumer pattern exists yet

---

## 8. @milo/feedback

**What it does:** Polymorphic feedback collection — context types, signals (up/down), metadata, tenant-scoped.

**Where it's wired today:**
- `src/app/api/feedback/record/route.ts` — raw Supabase insert to feedback_signals (does NOT use createFeedbackClient)
- Hero page (`src/app/page.tsx`) — email_capture signal via fetch to /api/feedback/record (D189)
- /proof page (`src/app/proof/page.tsx`) — reaction signal via fetch (D189)

**Where it's NOT wired:**
- `/ask` rating — uses `chat_logs.user_rating` column, NOT @milo/feedback
- `/api/feedback/record` itself — uses raw Supabase, not the SDK's createFeedbackClient

**@milo/feedback duplicate question:**

/ask thumbs (D192) writes to `chat_logs.user_rating` + `chat_logs.admin_flag`. @milo/feedback writes to `feedback_signals` table. These are **two separate systems doing related but different things:**

| Aspect | chat_logs.user_rating | @milo/feedback |
|---|---|---|
| Table | chat_logs | feedback_signals |
| Scope | Per-chat-message | Per-context (analysis, email, reaction) |
| Metadata | None (just up/down + flag) | JSONB metadata field |
| Tenant | Implicit (chat context) | Explicit tenant_id |
| Admin action | admin_flag triggers /review attention | No admin action model |
| Query pattern | Part of chat_logs queries | Standalone feedback queries |

**Recommendation: KEEP-AS-IS.**

Rationale: chat_logs.user_rating is tightly coupled to the chat message it rates — it's part of the chat log record, not a standalone signal. Migrating to @milo/feedback would mean either (a) duplicating the chat reference into feedback_signals.context_id or (b) JOINing feedback_signals to chat_logs on every /review query. Neither improves the UX. The admin_flag → /review pipeline is already built on chat_logs.

The real gap is that `/api/feedback/record` itself doesn't use createFeedbackClient — it's raw Supabase. But the route works fine, and switching to the SDK would be polish, not function.

**Plug-in opportunities:**

| Surface | Opportunity | Effort |
|---|---|---|
| /api/feedback/record | Replace raw Supabase insert with createFeedbackClient.recordSignal() | Single-file (polish) |
| /operator | Feedback dashboard pill — aggregate signals by context_type, show trends | Multi-file + new pill |

- **LOW:** SDK migration in feedback route — works fine as-is
- **LOW:** Feedback dashboard pill — insufficient signal volume to justify

---

## TOP 5 PLUG-IN OPPORTUNITIES

Ranked by operator impact × effort × dependencies:

### 1. /ask → @milo/ai-client migration (HIGH impact, multi-file)

**What:** Replace `new Anthropic()` in `/api/ask/route.ts` with callClaudeStream/callClaudeExtended. Replace classifier's raw SDK call with callClaude(tier='classify').

**Why #1:** /ask is the most-used public surface. It currently bypasses ALL primitive safeguards: no MODEL_QUIRKS (Pattern 15 repeat risk), no tier policy, no prompt caching (wasted tokens), no upstream error logging. A single Anthropic API deprecation breaks /ask while /operator stays fine.

**Effort:** Multi-file — route.ts refactor + prompt format adaptation.
**Dependencies:** None. callClaudeStream already exists in vendor.
**Lane:** Coder-1 (Extraction — package consumer wiring)
**Methods to wire:** `callClaudeStream()`, `callClaude(tier='classify')`, `TIER`

### 2. /ask → @milo/blacklist migration (HIGH impact, single-file)

**What:** Replace raw `from('blacklist').select()` in `/api/ask/route.ts` with `screenEntity()` from blacklist-client wrapper.

**Why #2:** Raw query does exact match only. Fuzzy matching in the primitive catches misspelled company names ("Acme Corp" vs "ACME Corporation"). Missing a blacklist hit on /ask — the public-facing surface — is a higher-impact failure than missing it in /operator.

**Effort:** Single-file — replace ~15 lines in route.ts with one screenEntity() call.
**Dependencies:** None.
**Lane:** Coder-1 or Coder-2
**Methods to wire:** `screenEntity()` (via existing wrapper), `BLACKLIST_SEVERITIES` for severity-aware tells

### 3. /ask Vet → @milo/contract-analysis (IN-FLIGHT)

**What:** Wire createPpcAnalysisEngine() behind Vet pill responses so Vet can run real 77-rule analysis.

**Why #3:** Already in-flight (Coder-1). Transforms Vet from "AI opinion about contracts" to "Milo's actual analysis engine output."

**Effort:** Multi-file. In progress.
**Dependencies:** None — engine already vendored.
**Lane:** Coder-1 (active)

### 4. /operator Contracts → detectJurisdiction() badge (MEDIUM impact, single-file)

**What:** Call `detectJurisdiction()` on contract text in the Contracts pill and show a jurisdiction badge (e.g., "CA", "NY", "International") on each contract row.

**Why #4:** Jurisdiction is a key factor operators check. The primitive already extracts it. Wiring a badge is ~20 lines in the content renderer.

**Effort:** Single-file — ContractClauseList.tsx or PillDrawer contentRenderer.
**Dependencies:** Contract text must be available in drawer context (it is via analysis_records).
**Lane:** Coder-1 or Coder-2
**Methods to wire:** `detectJurisdiction()`

### 5. /operator → stalled onboarding alerts (MEDIUM impact, multi-file)

**What:** New alert predicate using `findStalled()` from @milo/onboarding. Surfaces publishers/buyers stuck at an onboarding step for >48h.

**Why #5:** Operators currently have no visibility into stuck onboarding. findStalled() already exists and returns the exact data needed. Would appear as a new alert-based pill or as items in existing Alerts pill.

**Effort:** Multi-file — new predicate in operator-pills.ts + pill wiring.
**Dependencies:** Onboarding runs must exist (they do — publisher-onboarding.ts creates them).
**Lane:** Coder-1 (new predicate) + Coder-4 (pill config)
**Methods to wire:** `findStalled()`, `getOnboardingStatusByCounterparty()`

---

## Recommended pickup order (after current Vet wire-up completes)

1. **#1 /ask → ai-client** — Coder-1, immediate. Highest risk reduction.
2. **#2 /ask → blacklist** — Coder-1 or Coder-2, same session. Single-file, fast.
3. **#4 Jurisdiction badge** — Coder-2, follow-up. Small polish.
4. **#5 Stalled onboarding alerts** — Coder-1 + Coder-4, next ops cycle.
5. **#3 is already in-flight** — lands with current Vet work.

**Primitives with no good plug-in target right now:**
- **@milo/contract-signing** — No active signing flow in the product. The primitive is complete but the signing UX is unbuilt. Defer until signing feature is prioritized.
- **@milo/contract-negotiation** — Same situation. Part 2 redline links are planned but not imminent.
- **@milo/feedback** — SDK migration in the API route is polish, not function. KEEP-AS-IS on the /ask duplicate question.

---

Report at: https://raw.githubusercontent.com/moscaleflow/conductor-mark-public/main/reports/2026-04-24-coder-4-primitive-plugin-gap-audit.md
