# Pre-Build Readiness Audit for Coder-1 Queue

**Coder-3 Research | 2026-04-24**
**Scope:** Pure audit, read-only. Every assumed dependency verified against actual code before Coder-1 burns directive cycles.
**Motivation:** D200 hotfix proved assumptions can be wrong (engine.ts referenced files that didn't exist). This audit catches all such issues across 7 queued directives.

**UPDATE (Decision 114):** All 7 hard blockers resolved. Mark decided `blacklist_entries` is authoritative (Decision 35 primitive table, 343 entries, tenant='tlp'). Legacy `blacklist` table deprecated — migration/drop after Coder-2 D101 wireup. Each blocker mapped to the directive that owns its resolution. Coder-1 cleared to fire down queue.

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
- **BLOCKERS:** None (0 hard blockers)
- **AWARENESS (build-time):**
  - `runAnalysis` — not a standalone import. Use `createAnalysisEngine(config)` then `engine.analyze(text, opts)`.
  - `getAnalysisRecord` — does not exist. Use `engine.getAnalysis(id)` → `Promise<StoredAnalysis | null>`.
- **SCHEMA AMENDMENTS NEEDED:** None
- **ESTIMATED PREP TIME:** 0

### Coder-1 Prep Checklist
- [ ] Confirm D200 hotfix landed (engine.ts imports resolved)
- [ ] Use `createAnalysisEngine(config)` factory pattern, not `runAnalysis` standalone
- [ ] Use `engine.getAnalysis(id)`, not `getAnalysisRecord`
- [ ] Verify `verticalConfig.analysis.perIssueFields` exists in `vertical.config.ts` before relying on IssueCard field display

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
- **BLOCKERS:** None (0 hard blockers)
- **SCHEMA AMENDMENTS NEEDED:** None
- **ESTIMATED PREP TIME:** 0

### Coder-1 Prep Checklist
- [ ] All UI is inline in `proof/page.tsx` — no external component dependencies
- [ ] `SeverityBar` (line 123), `Stat` (line 651), `RiskBadge` (line 81), `RoleBadge` (line 103) are all inline — modify in-place
- [ ] Note: proof page has its OWN `RiskBadge`, separate from `components/contract/SeverityBadge.tsx`

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
- **BLOCKERS:** None remaining (all 4 resolved — see below)
- **RESOLVED BLOCKERS (Decision 114):**
  1. ~~URL mismatch (BUG)~~ **RESOLVED** — fix publish tool line 131: `/contracts/review/${row.id}` → `/contract-review/${row.id}`. In-scope for D3 gap fix Part A. Single-line fix.
  2. ~~`/contracts` listing page absent~~ **RESOLVED** — will be created as D3 gap fix Part B. Already in directive scope.
  3. ~~Ghost analysis pipeline absent~~ **RESOLVED** — pipeline will be scoped and built as part of D3. If directive determines it's unnecessary, it's a no-op.
  4. ~~Blacklist table mismatch~~ **RESOLVED (Decision 114)** — `blacklist_entries` is authoritative (Decision 35 primitive, 343 entries, tenant='tlp'). Legacy `blacklist` table deprecated. All new code uses `@milo/blacklist` SDK with `check(client, input)` method against `blacklist_entries`. Inline `blacklistPreCheck()` in route.ts to be migrated to use package.
- **AWARENESS (build-time):**
  - `screenEntity` does not exist — correct method is `check(client, input)` where `input: ScreenInput = { company_name (required), contact_names?, email?, ip_address?, linkedin_url? }`
  - Also available: `screenAgainst(entries, input)` for pure matching without DB call
- **SCHEMA AMENDMENTS NEEDED:** None
- **ESTIMATED PREP TIME:** 0 — all items are in-scope for the D3 directive

### Coder-1 Prep Checklist
- [ ] Fix URL in `publish-contract-for-buyer-review.ts` line 131 first (single-line, unblocks all links)
- [ ] Use `@milo/blacklist.check(client, input)` for all blacklist operations — NOT inline `blacklistPreCheck()`
- [ ] `check()` queries `blacklist_entries` table (not `blacklist`) — this is authoritative per Decision 114
- [ ] `ScreenInput` requires `company_name` — all other fields optional
- [ ] `/contracts/page.tsx` is new — no existing code to modify, create from scratch

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
- **BLOCKERS:** None (0 hard blockers)
- **SCHEMA AMENDMENTS NEEDED:** None
- **ESTIMATED PREP TIME:** 0

### Coder-1 Prep Checklist
- [ ] Read D106 report for exact refinement text: `reports/2026-04-24-coder-3-pill-voice-consistency-audit.md`
- [ ] HIGH-01: Normalize Publisher L36 + Buyer L51 experience claims (remove contradictory first-person numbers)
- [ ] HIGH-02: Add anti-hedging to Sales, Publisher, Buyer (Vet L56 is the template)
- [ ] HIGH-03: Add out-of-scope deflection to all 4 pills
- [ ] HIGH-04: Add posture anchors to Sales, Publisher, Buyer (Vet L5 is the template)
- [ ] HIGH-05: Add TLP voice registers to Sales (po tita, penguin culture, FAB team voice)

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
- **BLOCKERS:** None remaining (all resolved — see below)
- **RESOLVED BLOCKERS (Decision 114):**
  1. ~~`/api/s/[code]` or `/s/[code]` route absent~~ **RESOLVED** — will be created as part of D5 (Vet Part 2) build scope. Short-URL resolver route is a core deliverable of this directive.
  2. ~~`/contract/[slug]` route absent~~ **RESOLVED** — will be created as part of D5 build scope. Pretty-URL contract display page is a core deliverable.
  3. ~~Pretty slug generation absent~~ **RESOLVED** — will be built as part of D5. Only random 8-char codes exist currently; human-readable slug logic is new work in-scope.
- **AWARENESS (build-time):**
  - `sendRound()` not `createRound()` (rounds.ts:14)
  - `generateLink()` not `createLink()` (links.ts:26, exported from index.ts)
  - Use `createNegotiationClient().create()` factory pattern, not bare `createNegotiation()` import
  - Two independent `generateShortCode()` implementations exist: `contract-negotiation/links.ts:9` (crypto.getRandomValues) and `contract-signing/tokens.ts:7` (randomUUID). Be aware of dual implementation.
- **SCHEMA AMENDMENTS NEEDED:** None — `negotiation_links` already has `short_code` column
- **ESTIMATED PREP TIME:** 0 — all items are in-scope for the D5 directive

### Coder-1 Prep Checklist
- [ ] Use `createNegotiationClient()` factory, then `.create()` for negotiations
- [ ] Use `sendRound()` for rounds (NOT `createRound`)
- [ ] Use `generateLink()` for links (NOT `createLink`)
- [ ] Build `/s/[code]` route — resolves short codes to negotiation links or signing documents
- [ ] Build `/contract/[slug]` route — pretty-URL contract display page
- [ ] Build pretty slug generation (currently only random 8-char codes exist)
- [ ] Note: `resolveShortCode()` in contract-signing resolves against `signing_documents` only — negotiation short codes may need separate resolver or unified approach

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
- **BLOCKERS:** None remaining (all resolved — see below)
- **RESOLVED BLOCKERS (Decision 114):**
  1. ~~`/s/[code]` page absent~~ **RESOLVED** — prerequisite built in D5 (Vet Part 2). D6 fires after D5 ships.
  2. ~~Extend expiration API absent~~ **RESOLVED** — will be built as part of D6 scope. New method in `@milo/contract-negotiation`: `extendLinkExpiration(linkId, additionalDays)`. Goes through Decision 38 process (SDK extension). Proposed signature: `extendLinkExpiration(client: NegotiationClient, linkId: string, additionalDays: number) → Promise<NegotiationLink>`. Updates `expires_at` column on `negotiation_links`.
  3. ~~Negotiation email template absent~~ **RESOLVED** — will be built as part of D6 scope. New method in `@milo/contract-signing`: `sendNegotiationNotification()` or similar. Resend SDK is wired — needs new template for "You have a redline to review" notifications. Proposed: reuse `ResendEmailProvider` class, add `sendNegotiationRoundNotification({ to, negotiationId, roundNumber, linkUrl })`.
- **SCHEMA AMENDMENTS NEEDED:** None
- **ESTIMATED PREP TIME:** 0 — extend API + email template are core D6 deliverables, not prep

### Coder-1 Prep Checklist
- [ ] Confirm D5 (Vet Part 2) shipped first — `/s/[code]` route must exist
- [ ] Build `extendLinkExpiration()` in `@milo/contract-negotiation` — updates `expires_at` on `negotiation_links` table
- [ ] Build `sendNegotiationRoundNotification()` in `@milo/contract-signing` — reuse `ResendEmailProvider` class + Resend SDK
- [ ] Existing `sendSigningConfirmation()` is signing-only — do NOT repurpose for negotiation notifications
- [ ] `computeExpiresAt(days)` in `tokens.ts` can be reused for expiration calculation

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
- **BLOCKERS:** None remaining (all resolved — see below)
- **RESOLVED BLOCKERS (Decision 114):**
  1. ~~Blacklist table mismatch~~ **RESOLVED (Decision 114)** — `blacklist_entries` is authoritative. Inline `blacklistPreCheck()` in route.ts must be migrated to use `@milo/blacklist.check(client, input)`. Part of D7 build scope.
  2. ~~`screenEntity` name~~ **RESOLVED** — awareness item only. Correct method: `check(client, input)`.
- **AWARENESS (build-time):**
  - Use `@milo/blacklist.check(client, input)` — NOT inline `blacklistPreCheck()` and NOT `screenEntity`
  - `ASK_CLASSIFIER_AUTO_SUBMIT_THRESHOLD` env var — add to `.env` + `.env.example` during build (default 0.85 per D112)
  - Image upload path (`readAsDataURL` for images) — create during build, extend existing `handleFileContent`
  - Classifier prompt file — extract from inline `CLASSIFIER_PROMPT` in route.ts to `src/lib/ask/prompts/classifier.ts`
  - New SSE events (`confirm_switch`, `classify_confirm`) — add during build
  - Replace `contractFile: { name, data }` with `fileUpload: { name, data, mime }` per D112
- **SCHEMA AMENDMENTS NEEDED:** 2 new columns on `chat_logs` per D107 spec (`operator_override BOOLEAN`, `final_destination TEXT`)
- **ESTIMATED PREP TIME:** 0 — all items are in-scope for the D7 directive

### Coder-1 Prep Checklist
- [ ] Read D112-locked spec: `reports/2026-04-24-coder-3-surface-18-minimal-spec.md` (definitive)
- [ ] Migrate inline `blacklistPreCheck()` → `@milo/blacklist.check(client, input)` targeting `blacklist_entries` table
- [ ] `ScreenInput` requires `company_name` — all other fields optional
- [ ] Add `ASK_CLASSIFIER_AUTO_SUBMIT_THRESHOLD=0.85` to `.env` + `.env.example`
- [ ] Extract classifier from inline `CLASSIFIER_PROMPT` (route.ts L28) to `src/lib/ask/prompts/classifier.ts`
- [ ] Replace `contractFile` field with `fileUpload: { name, data, mime }` — backward compat shim for stale payloads
- [ ] Extend `handleFileContent` (page.tsx L416) with `readAsDataURL` for image MIME types
- [ ] Add SSE events: `classifying`, `classified`, `classify_confirm`, `confirm_switch`
- [ ] Run `ALTER TABLE chat_logs ADD COLUMN IF NOT EXISTS operator_override BOOLEAN DEFAULT FALSE; ALTER TABLE chat_logs ADD COLUMN IF NOT EXISTS final_destination TEXT;`

---

## Final Summary

### Blocker Resolution Status (Decision 114)

All 7 hard blockers from D113 are now RESOLVED. Mark decided `blacklist_entries` is authoritative (last open decision). Every blocker is mapped to the directive that owns its build.

| # | Original Blocker | Resolution | Owner Directive |
|---|---|---|---|
| 1 | `/s/[code]` route absent | Built as core deliverable | D5 (Vet Part 2) |
| 2 | `/contract/[slug]` route absent | Built as core deliverable | D5 (Vet Part 2) |
| 3 | `/contracts` listing page absent | Built as core deliverable | D3 (/contracts gap fixes Part B) |
| 4 | Extend expiration API absent | New `extendLinkExpiration()` in `@milo/contract-negotiation` | D6 (Vet Part 3) |
| 5 | Negotiation email template absent | New `sendNegotiationRoundNotification()` in `@milo/contract-signing` | D6 (Vet Part 3) |
| 6 | URL bug in publish tool | Single-line fix: `/contracts/review/` → `/contract-review/` | D3 (/contracts gap fixes Part A) |
| 7 | Blacklist table mismatch | `blacklist_entries` authoritative (Decision 114). Migrate inline code to `@milo/blacklist` SDK. | D3 + D7 |

### Awareness Items (build-time, not blockers)

| Item | Correct Usage | Affected Directives |
|---|---|---|
| `runAnalysis` standalone | Use `createAnalysisEngine(config)` → `engine.analyze(text, opts)` | D1 |
| `getAnalysisRecord` | Use `engine.getAnalysis(id)` → `Promise<StoredAnalysis \| null>` | D1 |
| `createRound` | Use `sendRound()` (rounds.ts:14) | D5 |
| `createLink` | Use `generateLink()` (links.ts:26) | D5 |
| `screenEntity` | Use `check(client, input)` — `ScreenInput.company_name` required | D3, D7 |
| `ASK_CLASSIFIER_AUTO_SUBMIT_THRESHOLD` | Add to `.env` + `.env.example`, default `0.85` | D7 |
| Image upload path | Extend `handleFileContent` with `readAsDataURL` for image MIMEs | D7 |
| Classifier prompt extraction | Extract inline `CLASSIFIER_PROMPT` to `src/lib/ask/prompts/classifier.ts` | D7 |

### Recommended Directive Sequencing (CONFIRMED — Decision 114)

Order unchanged from D113. Blacklist decision now resolved — D3 and D7 no longer gated.

| Order | Directive | Status | Rationale |
|---|---|---|---|
| **1** | **D4: Voice refinement** | CLEAR | Zero blockers. All target text verified. Pure prompt edits. Ship immediately. |
| **2** | **D2: /proof visual cleanup** | CLEAR | Zero blockers. All components inline. Pure UI. Ship immediately. |
| **3** | **D1: Vet Part 1 finish** | CLEAR | Awareness items only. Ships as soon as D200 hotfix lands. |
| **4** | **D3: /contracts gap fixes** | CLEAR | Blacklist decision resolved. URL fix + listing page + blacklist migration all in-scope. |
| **5** | **D7: Surface 18** | CLEAR | All primitives present. Biggest build. Blacklist migration shared with D3 approach. |
| **6** | **D5: Vet Part 2** | CLEAR | `/s/[code]` + `/contract/[slug]` routes built in-scope. Method name corrections documented. |
| **7** | **D6: Vet Part 3** | CLEAR (after D5) | Depends on D5 shipping. Extend API + email template built in-scope. |

### Critical Path (Updated)

```
D4 (voice) ───────────────────────> ship
D2 (/proof) ──────────────────────> ship
D1 (Vet P1) ── D200 lands ───────> ship
D3 (/contracts) ── blacklist migration ──> ship
D7 (Surface 18) ── biggest build ─> ship
D5 (Vet P2) ── /s/[code] + /contract/[slug] ──> ship
                                                    ↓
                                          D6 (Vet P3) ──> ship
```

**No remaining gates.** D4, D2, D1 can fire in parallel. D3 and D7 can start immediately (blacklist decision resolved). D5 can start anytime. Only D6 has a hard prerequisite (D5 must ship first).

**Parallelization opportunities:**
- D4 + D2 + D1 — fully independent, can ship in parallel
- D3 + D5 — independent, can run in parallel
- D7 — can start after D3's blacklist migration pattern is established (shared approach)

---

Cross-references: Decision 107 (Surface 18 spec), Decision 111 (open questions), Decision 112 (3 HIGH decisions locked), Decision 113 (original audit), Decision 114 (this resolution), Decision 200 (engine.ts hotfix).
