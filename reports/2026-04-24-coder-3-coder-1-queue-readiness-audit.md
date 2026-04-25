# Pre-Build Readiness Audit for Coder-1 Queue

**Coder-3 Research | 2026-04-24**
**Scope:** Pure audit, read-only. Every assumed dependency verified against actual code before Coder-1 burns directive cycles.
**Motivation:** D200 hotfix proved assumptions can be wrong (engine.ts referenced files that didn't exist). This audit catches all such issues across 7 queued directives.

---

## Directive 1: Vet Part 1 Finish (post-D200 hotfix)

### Dependencies Verified

| Dependency | Status | Notes |
|---|---|---|
| `src/components/contract/SeverityBadge.tsx` | PRESENT | 34 lines. Exports `SeverityBadge` (component) + `compareSeverity` (function). Imports `RiskLevel` from `@milo/contract-analysis`. |
| `src/components/contract/IssueCard.tsx` | PRESENT | 220 lines. Exports `IssueCard`. Uses `verticalConfig.analysis.perIssueFields` to drive display — Coder-1 should verify that config key exists in `vertical.config.ts`. |
| `src/components/contract/IssueSidebar.tsx` | PRESENT | 89 lines. Exports `IssueSidebar`. Imports from SeverityBadge + contract-analysis. |
| `src/lib/ppc/engine.ts` | PRESENT | 42 lines. Exports `createPpcAnalysisEngine`. Imports from `./index` and `./system-prompt` (extensionless, fine for Next.js). Line 1: imports `createAnalysisEngine, BASE_RULES` from `@milo/contract-analysis`. |
| `src/app/api/ask/route.ts` | PRESENT | Large file. **Type-only** import from `@milo/contract-analysis` (line 10: `import type { AnalysisResult, ContractIssue, RiskLevel }`). Does NOT call runtime methods from the package — has its own inline analysis logic. |
| `@milo/contract-analysis` — `BASE_RULES` | PRESENT | Named export from index.ts line 18, sourced from `./rules/base-registry.js`. |
| `@milo/contract-analysis` — `VerticalRulePack` | PRESENT | Type export from index.ts line 60. |
| `@milo/contract-analysis` — `runAnalysis` | **ABSENT as standalone** | Not a top-level export. It is internal to `engine.ts`. The public API is `createAnalysisEngine(config)` → `engine.analyze(text, opts)`. |
| `@milo/contract-analysis` — `getAnalysisRecord` | **ABSENT** | No such method exists anywhere. The correct method is `engine.getAnalysis(id)` on the engine instance, or `getAnalysis()` from internal `client.ts`. |
| `chat_logs.analysis_id` column | PRESENT | Migration at `supabase/migrations/20260424500001_analysis_id_column.sql`. |

### Verdict

- **READY-TO-BUILD:** SeverityBadge, IssueCard, IssueSidebar, engine.ts, route.ts, BASE_RULES, VerticalRulePack, analysis_id column
- **BLOCKERS:**
  - `runAnalysis` — not a standalone import. Coder-1 must use `createAnalysisEngine(config)` then call `engine.analyze(text, opts)`. Name correction only, no code missing.
  - `getAnalysisRecord` — does not exist. Correct method: `engine.getAnalysis(id)` returns `Promise<StoredAnalysis | null>`. Name correction only.
- **SCHEMA AMENDMENTS NEEDED:** None
- **ESTIMATED PREP TIME:** 0 — name corrections are awareness items, not missing code

---

## Directive 2: /proof Visual Cleanup

### Dependencies Verified

| Dependency | Status | Notes |
|---|---|---|
| `src/app/proof/page.tsx` | PRESENT | 662 lines. `'use client'` component. Loads from `/public/marketing/real-contracts.json`. |
| Multicolored bar component | PRESENT | `SeverityBar` defined inline at lines 123-151. 6px height, flex-based proportional widths with severity colors. NOT the shared `StatBar.tsx` component. |
| 4-stat row | PRESENT | Lines 556-569. Flex container with 4 inline `Stat` calls: Contracts analyzed / Issues found / Buyer-Publisher split / CRITICAL count. |
| `Stat` component | PRESENT | Inline at lines 651-661. Simple value + label div. NOT the shared `StatBar.tsx`. |
| `RiskBadge` | PRESENT | Inline at lines 81-101. Different from `components/contract/SeverityBadge.tsx` — separate implementation. |
| `RoleBadge` | PRESENT | Inline at lines 103-121. buyer/publisher label. |

### Verdict

- **READY-TO-BUILD:** All components present, all inline
- **BLOCKERS:** None
- **SCHEMA AMENDMENTS NEEDED:** None
- **ESTIMATED PREP TIME:** 0

---

## Directive 3: /contracts Gap Fixes (4 sub-parts)

### Dependencies Verified

| Dependency | Status | Notes |
|---|---|---|
| `src/lib/tools/publish-contract-for-buyer-review.ts` | PRESENT | 170 lines. Tool ID `2A`. Generates 6-char hex `short_slug`, builds URLs. |
| URL generation in publish tool | **BUG** | Line 131 generates `${baseUrl}/contracts/review/${row.id}` but the actual page route is at `/contract-review/[id]`. **Published links will 404.** |
| `src/app/contract-review/[id]/page.tsx` | PRESENT | 1,603 lines. Route resolves to `/contract-review/:id`. Uses `useParams()` for id. |
| `src/app/contracts/page.tsx` (listing page) | **ABSENT** | No `/contracts/` directory exists under `src/app/`. Must be created from scratch. |
| Ghost analysis pipeline | **ABSENT** | Zero cleanup/pipeline logic exists. All "ghost"/"orphan" matches in codebase are prompt text and regex patterns, not pipeline code. If directive assumes a pipeline exists, it must be built. |
| `@milo/blacklist` — `screenEntity` | **ABSENT** | No such method. Correct method is `check(client, input)` where `input: ScreenInput = { company_name (required), contact_names?, email?, ip_address?, linkedin_url? }`. Also `screenAgainst(entries, input)` for pure matching. |
| `@milo/blacklist` integration with milo-for-ppc | **PARTIAL** | milo-for-ppc does NOT import `@milo/blacklist` anywhere. Route.ts has its own inline `blacklistPreCheck()` at line 136 that queries the `blacklist` table directly — but the package queries `blacklist_entries` table. **Table name mismatch.** |

### Verdict

- **READY-TO-BUILD:** publish tool (exists), contract-review page (exists)
- **BLOCKERS:**
  1. **URL mismatch (BUG):** Publish tool generates `/contracts/review/:id` but route is `/contract-review/:id`. Fix: change line 131 in publish tool to match actual route. Single-line fix.
  2. **`/contracts` listing page absent:** Must be created. Scope: new page with contract list, filters, status indicators.
  3. **Ghost analysis pipeline absent:** No cleanup code exists. Must be built from scratch if the directive requires it. Scope: depends on directive — could be a cron/edge function or a one-off script.
  4. **`screenEntity` → `check()`:** Name correction. But bigger issue: milo-for-ppc uses inline blacklist code against `blacklist` table while `@milo/blacklist` package uses `blacklist_entries` table. Integration requires either (a) migrating inline code to use the package, or (b) aligning table names.
- **SCHEMA AMENDMENTS NEEDED:** Blacklist table alignment may need Mark decision — which table is authoritative?
- **ESTIMATED PREP TIME:** 30min for URL fix + blacklist name correction. `/contracts` page and ghost pipeline are full build items, not prep.

---

## Directive 4: Voice Refinement HIGH Proposals (D106)

### Dependencies Verified

| Dependency | Status | Notes |
|---|---|---|
| `src/lib/ask/prompts/vet.ts` | PRESENT | 60 lines. TOOLS, SIGNAL LIBRARY, OUTPUT FORMAT, BLACKLIST PRE-CHECK, CONTRACT VET MODE, STYLE. |
| `src/lib/ask/prompts/sales.ts` | PRESENT | 40 lines. TOOLS, OUTREACH FRAMEWORK, OUTPUT FORMAT, RE-ENGAGEMENT RULES, BUYER PSYCHOLOGY, STYLE. |
| `src/lib/ask/prompts/publisher.ts` | PRESENT | 40 lines. TOOLS, RECRUITMENT FRAMEWORK, OUTPUT FORMAT, PUBLISHER EVALUATION SIGNALS, TRAFFIC SOURCE INTELLIGENCE, STYLE. |
| `src/lib/ask/prompts/buyer.ts` | PRESENT | 55 lines. TOOLS, CALL PREP FRAMEWORK, DISPUTE HANDLING, BLACKLIST DECISIONS, PERFORMANCE ANALYSIS, BILLING TYPE AWARENESS, STYLE. |

### D106 HIGH Proposal Target Text Verification

| Proposal | Target Text | Still Present? | Verdict |
|---|---|---|---|
| HIGH-01: Normalize experience claims | Publisher L36: `"Talk like someone who's recruited 500 publishers, because I have"`. Buyer L51: `"Talk like someone who's managed 200 buyer accounts, because I have"` | YES — both contradictory claims unchanged | Valid target |
| HIGH-02: Anti-hedging | Vet L56: `"'I have concerns' is weak."` Sales/Publisher/Buyer: zero anti-hedging | YES — gap confirmed, only Vet has it | Valid target |
| HIGH-03: Out-of-scope deflection | All 4 pills: zero deflection instructions for off-topic queries | YES — absent from all pills | Valid target |
| HIGH-04: Posture anchors | Vet L5: `"Your default posture is skepticism."` Other 3 pills: no posture anchor | YES — only Vet has anchor | Valid target |
| HIGH-05: TLP voice registers | Sales: zero matches for "po tita", "where you at", "penguin", "FAB" | YES — absent from all pills | Valid target |

### Verdict

- **READY-TO-BUILD:** All 4 prompt files present. All 5 HIGH targets verified — text has not drifted since D106 audit.
- **BLOCKERS:** None
- **SCHEMA AMENDMENTS NEEDED:** None
- **ESTIMATED PREP TIME:** 0

---

## Directive 5: Vet Part 2 (Redline Links, Dual URL)

### Dependencies Verified

| Dependency | Status | Notes |
|---|---|---|
| `@milo/contract-negotiation` package | PRESENT | `~/Documents/GitHub/milo-engine/packages/contract-negotiation/`. Files: client.ts, export.ts, index.ts, links.ts, negotiations.ts, rounds.ts, state-machine.ts, types.ts. |
| `createNegotiation` | **PARTIAL** | Not a direct public export. Accessed via `createNegotiationClient().create()`. |
| `createRound` | **NAME MISMATCH** | Actual method is `sendRound()` (rounds.ts:14). Also `submitCounterpartyRound()` at line 97. |
| `createLink` | **NAME MISMATCH** | Actual method is `generateLink()` (links.ts:26). Exported directly from index.ts. |
| `@milo/contract-signing` package | PRESENT | Files: audit.ts, certificate.ts, client.ts, documents.ts, email.ts, index.ts, short-url.ts, tax-routing.ts, templates.ts, tokens.ts, types.ts, webhooks.ts. |
| `tokens.ts` | PRESENT | Exports `generateSigningToken()`, `generateShortCode()`, `isTokenExpired()`, `computeExpiresAt()`. |
| `short-url.ts` | PRESENT | Exports `resolveShortCode()`. Resolves against `signing_documents` table only. |
| Email send capability | PRESENT | `ResendEmailProvider` class with `sendSigningConfirmation()`. Uses Resend SDK. **Signing-only templates** — negotiation notification emails would need a new method. |
| `/api/s/[code]` route | **ABSENT** | No route exists at `/api/s/[code]/route.ts` or `/s/[code]/page.tsx`. Must be built. |
| `negotiation_links` table | PRESENT | Migration at `packages/contract-negotiation/migrations/00001_initial_schema.sql` (lines 44-74). 12 columns including `short_code`, `token`, `expires_at`. RLS with tenant isolation. Migrated to shared Supabase (tappyckcteqgryjniwjg). |
| `negotiation_links.short_code` column | PRESENT | Exists in migration. Index on `short_code WHERE NOT NULL`. |
| Short code generation | PRESENT (dual) | `contract-negotiation/links.ts:9` — `generateShortCode()` via `crypto.getRandomValues`, 8-char. `contract-signing/tokens.ts:7` — `generateShortCode()` via `randomUUID().substring(0, 8)`. Two independent implementations. |
| `/contract/[slug]` route | **ABSENT** | Does not exist. Must be created. |
| Pretty slug generation | **ABSENT** | No slug generation beyond the 8-char random codes. Human-readable slugs must be built. |

### Verdict

- **READY-TO-BUILD:** contract-negotiation package (with correct method names), contract-signing package, negotiation_links table, short code generation
- **BLOCKERS:**
  1. **`/api/s/[code]` or `/s/[code]` route absent:** Must be built. Resolves short codes to negotiation links or signing documents.
  2. **`/contract/[slug]` route absent:** Must be created for pretty-URL contract display.
  3. **Method name corrections:** `sendRound` not `createRound`, `generateLink` not `createLink`, use `createNegotiationClient().create()` not bare `createNegotiation`.
  4. **Pretty slug generation absent:** Only random 8-char codes exist. Human-readable slugs need implementation.
- **SCHEMA AMENDMENTS NEEDED:** None — `negotiation_links` already has `short_code` column
- **ESTIMATED PREP TIME:** Method names are awareness items. `/s/[code]` + `/contract/[slug]` routes are build items (~2-3 hours).

---

## Directive 6: Vet Part 3 (Send/Extend Buttons)

### Dependencies Verified

| Dependency | Status | Notes |
|---|---|---|
| `/s/[code]` page | **ABSENT** | Same as D5. Must be built first (D5 dependency). |
| Resend email infrastructure | PRESENT (partial) | `ResendEmailProvider` exists with `sendSigningConfirmation()`. For negotiation round notifications (e.g., "You have a redline to review"), a new method and email template are needed. Resend SDK is wired — just needs new template. |
| Extend expiration API | **ABSENT** | Zero methods for extending link expiration in `@milo/contract-negotiation`. `generateLink()` sets `expires_at` at creation via `computeExpiresAt(days)`, but no update method exists. The `negotiation_links` table has `expires_at` column — could be updated directly, but no package API for it. Must be built. |

### Verdict

- **READY-TO-BUILD:** Resend SDK integration (infra exists)
- **BLOCKERS:**
  1. **`/s/[code]` page absent:** Same blocker as D5. D5 must ship first or in parallel.
  2. **Extend expiration API absent:** Must be built in `@milo/contract-negotiation`. New method on the client: `extendLinkExpiration(linkId, additionalDays)`.
  3. **Negotiation email template absent:** `sendSigningConfirmation()` is signing-only. Need `sendNegotiationNotification()` or similar. Resend infra is wired — template + method needed.
- **SCHEMA AMENDMENTS NEEDED:** None
- **ESTIMATED PREP TIME:** ~2-3 hours (extend API + email method + template)

---

## Directive 7: Surface 18 Implementation (D112-locked)

### Dependencies Verified

| Dependency | Status | Notes |
|---|---|---|
| `@milo/ai-client` — `callClaudeVision` | PRESENT | Exported from index.ts:4. Signature: `(prompt, { imageBase64, mediaType, tier, system, maxTokens, temperature })` → string. |
| `@milo/ai-client` — `callClaude` (text) | PRESENT | For non-vision classification (document classifier). |
| `@milo/contract-analysis` — `extractText` | PRESENT | Exported from index.ts:6. Signature: `(file: Buffer, filename: string)` → `Promise<ExtractionResult>`. Handles PDF (pdf-parse) and DOCX (mammoth). |
| `@milo/crm` — `createLead` | PRESENT | Exported from index.ts:15. Signature: `(client, input: LeadInsert, performedBy?)` → Lead. |
| `@milo/blacklist` — `check` | PRESENT | Correct method name is `check(client, input)`. NOT `screenEntity`. `ScreenInput: { company_name (required), contact_names?, email?, ip_address?, linkedin_url? }`. |
| `@milo/feedback` — `recordFeedback` | PRESENT | Exported from index.ts:5. For routing accuracy signals. |
| File drop handler (`ask/page.tsx`) | PRESENT | `handleFileContent` at line 416. Bifurcates on `.pdf/.docx` regex. Contract: ArrayBuffer → base64, max 5MB. Other: readAsText, max 100KB. Auto-submit hardcoded to `selectedPill === 'vet'` only. |
| SSE streaming pattern | PRESENT | Uses `fetch` + `getReader()` (NOT EventSource). Manual SSE parsing. Event types: `classified`, `blacklist_hit`, `analysis`, `status`, `delta`, `done`, `error`. |
| `contractFile` request field | PRESENT | `{ name, data }` in JSON body. D112 says replace with `fileUpload: { name, data, mime }`. |
| `ASK_CLASSIFIER_AUTO_SUBMIT_THRESHOLD` env var | ABSENT (expected) | Zero matches in codebase. Needs to be added per D112 spec. |
| Classifier prompt location | PARTIAL | Current classifier (`CLASSIFIER_PROMPT`) is inline in `api/ask/route.ts` at line 28. `classifyInput()` at line 164 uses `claude-haiku-4-5-20251001` with 4 few-shot examples. No classifier file in `/src/lib/ask/prompts/`. D7 should extract to `prompts/classifier.ts`. |
| Image upload path on client | ABSENT (expected) | No `readAsDataURL` path for images. Only text and ArrayBuffer. Must be added per D112 spec. |
| New SSE event types (`confirm_switch`, `classify_confirm`) | ABSENT (expected) | Must be added per D112 spec. |

### Verdict

- **READY-TO-BUILD:** All 6 @milo/* primitive methods exist with correct signatures. SSE pattern established. File drop handler exists (needs extension).
- **BLOCKERS:**
  1. **`screenEntity` → `check()`:** Name correction across all D7 references. Not a code blocker.
  2. **Blacklist integration gap:** milo-for-ppc uses inline `blacklistPreCheck()` against `blacklist` table. Package uses `blacklist_entries` table. If D7 wires the package, table alignment is needed.
  3. **Env var addition:** `ASK_CLASSIFIER_AUTO_SUBMIT_THRESHOLD` needs to be added to `.env` + `.env.example`.
- **SCHEMA AMENDMENTS NEEDED:** None (D107 spec already covers the 2 new columns on chat_logs)
- **ESTIMATED PREP TIME:** 15min (env var + classifier extraction are part of the build, not prep)

---

## Final Summary

### Total Blockers by Category

| Category | Count | Items |
|---|---|---|
| **Absent routes (must build)** | 3 | `/s/[code]` route (D5+D6), `/contract/[slug]` route (D5), `/contracts` listing page (D3) |
| **Absent APIs (must build)** | 2 | Extend expiration method (D6), negotiation email template (D6) |
| **Bug fix needed** | 1 | URL mismatch in publish tool — `/contracts/review/:id` should be `/contract-review/:id` (D3) |
| **Name corrections (awareness)** | 5 | `runAnalysis` → `engine.analyze()` (D1), `getAnalysisRecord` → `engine.getAnalysis()` (D1), `createRound` → `sendRound()` (D5), `createLink` → `generateLink()` (D5), `screenEntity` → `check()` (D3+D7) |
| **Integration issues** | 1 | Blacklist table mismatch: inline code uses `blacklist` table, package uses `blacklist_entries` (D3+D7) |
| **Expected absent (add during build)** | 3 | `ASK_CLASSIFIER_AUTO_SUBMIT_THRESHOLD` env var (D7), image upload path (D7), classifier prompt file extraction (D7) |
| **Absent pipelines** | 1 | Ghost analysis cleanup (D3) — if directive requires it |
| **Pretty slug generation** | 1 | No human-readable slug logic exists (D5) |

**TOTAL HARD BLOCKERS: 7** (3 absent routes + 2 absent APIs + 1 bug + 1 integration issue)
**TOTAL AWARENESS ITEMS: 8** (5 name corrections + 3 expected absent)

### Blockers Grouped by Fix Type

**Stub file / new route (Coder-1 builds these):**
- `/s/[code]` or `/api/s/[code]` — short-URL resolver route (blocks D5 + D6)
- `/contract/[slug]` — pretty-URL contract display (blocks D5)
- `/contracts/page.tsx` — contracts listing page (blocks D3)

**Primitive refactor (Coder-1 in milo-engine):**
- `@milo/contract-negotiation` — add `extendLinkExpiration()` method (blocks D6)
- `@milo/contract-signing` — add `sendNegotiationNotification()` email method (blocks D6)

**Bug fix (Coder-1 single-line):**
- `publish-contract-for-buyer-review.ts` line 131 — change `/contracts/review/${row.id}` to `/contract-review/${row.id}`

**Mark decision needed:**
- Blacklist table alignment: `blacklist` (inline code) vs `blacklist_entries` (package). Which is authoritative? Should inline code migrate to use `@milo/blacklist` package, or should package be updated to query `blacklist` table? Affects D3 + D7.

### Recommended Directive Sequencing

Based on blocker cascade analysis:

| Order | Directive | Rationale |
|---|---|---|
| **1** | **D4: Voice refinement** | Zero blockers. All target text verified. Pure prompt edits, no dependencies. Ship immediately. |
| **2** | **D2: /proof visual cleanup** | Zero blockers. All components inline. Pure UI, no SDK dependencies. Independent of everything else. |
| **3** | **D1: Vet Part 1 finish** | Two name corrections only (awareness items, not missing code). Can ship as soon as D200 hotfix lands. |
| **4** | **D3: /contracts gap fixes** | 4 sub-parts. URL bug is a one-line fix. `/contracts` listing page is new work. Ghost pipeline scope unclear. Blacklist table decision from Mark needed. Start after Mark resolves blacklist table question. |
| **5** | **D7: Surface 18** | Most dependencies PRESENT. Biggest build. Blacklist table decision affects this too. Classifier extraction and env var are part of the build, not prep. Can start in parallel with D3 if blacklist question is resolved. |
| **6** | **D5: Vet Part 2** | Needs `/s/[code]` route + `/contract/[slug]` route built. Method name corrections. Not blocked by earlier directives. Can start after D1. |
| **7** | **D6: Vet Part 3** | Depends on D5 shipping first (`/s/[code]` route). Also needs extend-expiration API + email template built in milo-engine. Last in sequence. |

### Critical Path

```
D4 (voice) ──────────────────────> ship
D2 (/proof) ─────────────────────> ship
D1 (Vet P1) ── D200 lands ──────> ship
                                     ↓
Mark: blacklist table decision ───> D3 (/contracts) ──> ship
                                  ↘ D7 (Surface 18) ──> ship
D5 (Vet P2) ── build /s/[code] ─> ship
                                     ↓
                              D6 (Vet P3) ──> ship
```

**Single Mark decision gates D3 + D7:** Which blacklist table is authoritative — `blacklist` (inline) or `blacklist_entries` (package)?

---

Cross-references: Decision 107 (Surface 18 spec), Decision 111 (open questions), Decision 112 (3 HIGH decisions locked), Decision 200 (engine.ts hotfix).
