# /ask End-to-End Operator Workflow Audit — What's Missing for Monday Morning

**Coder-3 Research | 2026-04-24**
**Scope:** Pure audit. No code. Walk 7 daily-use operator scenarios against shipped production code. Identify gaps before Tiffani hits it Monday.
**Shipped today:** D116 (Vet Part 1), Vet Part 1.5 cards, D117 (CRM context), D118 (ai-client + blacklist wireup), D120 (paste handler), image vision support.
**Not shipped yet:** Surface 18 v1 (schema deployed, application code not wired).

---

## Scenario A: Vet a New Entity by Name Only

**Operator action:** Types "Vet Granite Media" with no pill selected.

### Code path

1. No pill selected → `effectivePill = 'auto'` (route.ts L315)
2. Auto-classifier (Haiku, tier `classify`) runs `classifyInput()` → maps "Vet Granite Media" to `vet` with high confidence
3. `classified` SSE event emitted → client updates pill indicator to Vet
4. Blacklist pre-check fires (only runs for `pill === 'vet'`, route.ts L278) → `screenEntity({ company_name: 'Granite Media' })` against `blacklist_entries` table via `@milo/blacklist`
5. CRM context: `findCrmContext()` extracts "Granite Media", searches `crm_counterparties` by ILIKE, injects `[CRM CONTEXT: ...]` if found
6. Vet system prompt loaded with web_search tool (max 5 uses)
7. Claude streams response: entity name + type → RED FLAGS → GREEN FLAGS → CONFIDENCE → verdict → suggested blacklist note if negative
8. Response logged to `chat_logs` with pill, input_text, model, tokens, latency, context_metadata

### What works

- Auto-classify correctly routes "Vet X" to Vet pill
- Blacklist pre-check surfaces hits with `[BLACKLIST HIT]` tag → Vet prompt says "I've seen this one before"
- CRM context tells Milo if Granite Media is already in the system
- web_search fires for entity research (LinkedIn, LLC registrations, BBB)
- Response voice is strong: "Your default posture is skepticism" + anti-hedging + locked output format
- Rating thumbs appear after response completes
- Chat logged for /review QA dashboard

### What's missing or broken

| Gap | Severity | Detail | Fix | Effort |
|---|---|---|---|---|
| **No conversation memory** | HIGH | If Tiffani follows up "what about their payment history?", Claude sees a blank context — zero memory of prior turn. Each submission is a single-shot API call. | Send conversation history in `messages` array | M |
| **web_fetch promised but missing** | MEDIUM | Vet prompt says "web_fetch: Grab LinkedIn pages, company websites" but only web_search is in tools array. Model may attempt web_fetch and fail silently. | Add web_fetch to tools array or remove from prompts | S |
| **Blacklist only checks company_name** | MEDIUM | `screenEntity({ company_name })` — doesn't pass email, linkedin_url, contact_names even though @milo/blacklist supports all signals. "Vet john@granitemedia.com" misses email-based blacklist hits. | Pass all extracted signals to ScreenInput | S |
| **No "Searching the web..." indicator** | LOW | TODO at route.ts L371. User sees no feedback during web searches (5-10 seconds). | Restore streamEvent exposure from callClaudeStream | M |

---

## Scenario B: Vet by Dropping a Contract PDF

**Operator action:** Drops CrossBlade_Buyer_MSA.pdf on /ask while on Vet pill.

### Code path

1. `handleFileContent()` detects `.pdf` → `isContract = true`
2. readAsArrayBuffer → base64, max 5MB
3. `selectedPill === 'vet'` → `pendingAutoSubmitRef.current = true`
4. useEffect auto-fires `doSubmit('vet', 'Vet this.', attachedFile)`
5. Route.ts: `pill === 'vet' && contractFile?.data` → extract text via pdf-parse, truncate to 12K chars
6. Role detection: filename lacks "publisher" → defaults to `'buyer'`
7. Separate `callClaude` (Sonnet, structured tier) with analysis system prompt → returns AnalysisResult JSON
8. `analysis` SSE event → client stores on message, renders IssueListExpandable BEFORE text
9. `formatAnalysisContext()` prepends `[CONTRACT ANALYSIS]` with CRITICAL + HIGH issues only
10. Main `callClaudeStream` with vet prompt's CONTRACT VET MODE → 3-4 line verdict streams AFTER cards

### What works

- Drag-and-drop UX with full-screen overlay ("Drop it anywhere")
- Auto-submit on Vet pill — zero typing required
- PDF text extraction (pdf-parse) and DOCX (mammoth)
- Per-issue cards with SeverityBadge, title, plain_summary, problem_explanation, suggested_revision, negotiation_leverage
- Accept / Reject / Edit / Explain Further buttons per issue
- InlineRedlineEditor with "Ask Milo to refine" AI-powered redline refinement
- Action summary bar showing accept/reject/edit counts
- Milo verdict streams below cards — correct visual hierarchy
- analysis_id persisted to chat_logs

### What's missing or broken

| Gap | Severity | Detail | Fix | Effort |
|---|---|---|---|---|
| **Per-issue decisions NOT persisted** | HIGH | `issueStates` is `useState<Record<number, IssueState>>({})` in IssueListExpandable.tsx L37. Accept/reject/edit states are React-only. Page refresh resets ALL decisions to 'pending'. Tiffani accepts 8 of 11 issues, refreshes, all gone. | Save states to Supabase keyed by analysis_id + issue_id | M |
| **No file picker button** | HIGH | No `<input type="file">` or paperclip icon anywhere. Only drag-and-drop and clipboard paste. Mobile users cannot attach files at all — drag-and-drop is non-functional on phones. | Add attach button with file input | S |
| **Role detection is filename-based** | MEDIUM | `const role = /publisher/i.test(contractFile.name) ? 'publisher' : 'buyer'` (route.ts L322). If filename is "Contract_v3.pdf", defaults to buyer. Wrong role = wrong rules applied. | Infer role from extracted text content (party names, terminology) | S |
| **MEDIUM/LOW issues invisible in verdict context** | MEDIUM | `formatAnalysisContext()` only includes CRITICAL + HIGH issue titles in KEY ISSUES list. If contract has 0 critical/high but 6 medium issues, Milo's verdict has no issue names to reference — just counts. | Include MEDIUM issue titles (not details) in context | S |
| **Contract analysis only fires for Vet pill** | MEDIUM | `if (pill === 'vet' && contractFile?.data)` (route.ts L315). Dropping a PDF on any other pill ignores the contract entirely — no extraction, no analysis, no cards. | Surface 18 classifier (D112) handles this — file-type routing to vet regardless of current pill | M |
| **Auto-submit race condition** | LOW | useEffect at L298-303 has `[attachedFile]` dependency but reads `selectedPill` from stale closure. Drop file while pill is null → auto-submit check fails silently, `pendingAutoSubmitRef` stays true but never fires. | Add `selectedPill` to dependency array or use ref | S |

---

## Scenario C: Sales Advice on a Buyer Interaction

**Operator action:** Types "Buyer ghosted me after 4 days, what do I say?" with no pill selected.

### Code path

1. Auto-classify → "drafting/writing actions" → `sales` (first priority rule in classifier)
2. No blacklist check (only fires for vet)
3. CRM context: no entity name extracted from this query (no proper-cased multi-word phrase)
4. Sales system prompt: Hormozi outreach framework, re-engagement rules, buyer psychology
5. web_search available but unlikely to fire (no entity to search)
6. Claude drafts a follow-up message in Sales voice

### What works

- Classifier correctly routes "what do I say" to Sales
- Sales prompt has specific re-engagement rules ("3-day rule", "value-first callback")
- Response produces a ready-to-use drafted message
- Under-150-word cold outreach target keeps messages concise

### What's missing or broken

| Gap | Severity | Detail | Fix | Effort |
|---|---|---|---|---|
| **Drafted outreach stays in chat only** | MEDIUM | Milo drafts a message but it lives in the chat bubble. No "Copy to clipboard" button, no "Send via email" action, no CRM activity logging. Tiffani manually copies from chat to Teams/email. | Add copy button on assistant messages. CRM activity log is a stretch goal. | S (copy), L (CRM) |
| **No cross-pill context** | MEDIUM | If Tiffani vetted this buyer 5 minutes ago and now switches to Sales, Sales pill has zero knowledge of what Vet found. "Draft outreach to that buyer" requires re-explaining who the buyer is. | Conversation history in messages array (same as Scenario A fix) | M |
| **CRM context fails on unstructured queries** | LOW | "Buyer ghosted me" has no entity name → `extractEntityNames()` returns nothing → no CRM lookup. If Tiffani says "MediaBridge ghosted me", CRM context would fire. The extraction relies on proper-cased multi-word phrases. | Improve entity extraction to handle "buyer [Name] ghosted" patterns | M |
| **No TLP voice register awareness** | LOW | D106 HIGH-05 flagged: Sales pill has no "Tiffani style" or "Cam style" voice registers. If operator asks "make it sound like Cam", Milo ignores the style request. | Add voice register section per D106 HIGH-05 proposal | S |

---

## Scenario D: Publisher Recruitment on a Vertical

**Operator action:** Types "Looking for ACA publishers" with no pill selected.

### Code path

1. Auto-classify → "publisher topics" → `publisher`
2. No blacklist check (vet only)
3. CRM context: "ACA" is not a proper-cased multi-word entity name → no CRM lookup
4. Publisher system prompt: recruitment framework, publisher evaluation signals, traffic source intelligence
5. web_search fires to find ACA-focused publisher networks

### What works

- Classifier correctly routes to Publisher pill
- Publisher prompt has strong recruitment framework with red/yellow/green evaluation signals
- Traffic source intelligence section (SEO, PPC, social, radio/TV, warm transfer) helps evaluate publisher types
- web_search can find real ACA publishers in the industry

### What's missing or broken

| Gap | Severity | Detail | Fix | Effort |
|---|---|---|---|---|
| **No access to publisher network data** | MEDIUM | Publisher pill has zero access to TLP's actual publisher roster, performance data, call volumes, or conversion rates. It can only web-search externally and advise generically. | Inject publisher performance data from pipeline/performance tables as context | L |
| **No publisher list or directory** | MEDIUM | If Tiffani asks "which publishers are already running ACA for us?", Milo cannot answer. The CRM has counterparties but the /ask route doesn't search by vertical or campaign type. | Extend CRM context to support vertical-based publisher search | M |
| **web_fetch not available** | LOW | Prompt says "web_fetch: Pull publisher websites, LinkedIn pages, affiliate network profiles" but tool not configured. | Same fix as Scenario A | S |

---

## Scenario E: Buyer Dispute Defense

**Operator action:** Types "Buyer disputing 15 calls, how do I handle?" with Buyer pill selected.

### Code path

1. Pill explicitly set → no auto-classify
2. No blacklist check (vet only)
3. CRM context: no entity name in query → no CRM lookup
4. Buyer system prompt: dispute handling framework, call prep, performance analysis, billing type awareness
5. web_search available

### What works

- Buyer prompt has detailed dispute handling framework with escalation tiers
- Billing type awareness (CPA/RTB/CPL/CPQL) ensures billing-type-specific advice
- "Never side with the buyer automatically" voice instruction keeps responses TLP-first
- Blacklist decision framework ("Tiffani's note pattern") for when to blacklist

### What's missing or broken

| Gap | Severity | Detail | Fix | Effort |
|---|---|---|---|---|
| **No access to actual dispute/call data** | MEDIUM | Buyer pill cannot look up specific disputed calls, listen to recordings, check ConvoQC scores, or pull billing records. Advice is generic framework only. | Inject call data from TrackDrive/ConvoQC as context (requires integration) | L |
| **No dispute-specific context injection** | MEDIUM | CRM context finds counterparties but doesn't surface dispute history, open tickets, or prior resolutions. "How do I handle" gets generic advice, not "last time MediaBridge disputed calls, here's what worked." | Extend CRM context to include dispute/activity history | M |
| **Blacklist check not running** | LOW | Buyer might be discussing a known bad actor but blacklist pre-check only runs on Vet pill. | Extend blacklist check to Buyer pill (and optionally all pills) | S |

---

## Scenario F: Drop a LinkedIn Screenshot (Surface 18 v1)

**Operator action:** Pastes a LinkedIn profile screenshot on /ask.

### Code path (current state)

1. `handleFileContent()` detects `file.type.startsWith('image/')` → `isImage = true`
2. readAsArrayBuffer → base64, stored in `attachedFile.imageData` + `attachedFile.mimeType`
3. Image preview shows 40x40 thumbnail in attachment chip
4. If `selectedPill === 'vet'`, auto-submits with "Vet this."
5. Route.ts: `imageFile` present → constructs vision content blocks (image + text)
6. Claude sees the screenshot and responds in the active pill's voice

### What works

- Image paste handler (D120) captures clipboard images
- Image drag-and-drop works
- 10MB size limit for images
- Vision content blocks correctly constructed for Claude API
- Image preview with click-to-enlarge

### What's missing or broken (Surface 18 v1 NOT shipped)

| Gap | Severity | Detail | Fix | Effort |
|---|---|---|---|---|
| **No file-type classifier** | HIGH | No Surface 18 classifier exists in shipped code. Dropping a LinkedIn screenshot doesn't auto-route to Sales/Publisher — it stays on whatever pill the operator selected. No entity extraction, no CRM lead creation, no blacklist check on the screenshot entity. | Implement Surface 18 classifier per D112 spec | L |
| **No cross-pill confirmation** | HIGH | D112 specified "I see a LinkedIn screenshot. Switch to Sales?" flow. Not implemented. | Part of Surface 18 v1 build | L |
| **Auto-submit fires for images in Vet mode** | MEDIUM | `pendingAutoSubmitRef` is set for ANY file type when pill is Vet (page.tsx L262-264). Image paste on Vet pill auto-submits "Vet this." which makes no sense for a LinkedIn screenshot. | Guard auto-submit: only fire for contract MIME types, not images | S |
| **No CRM lead creation from screenshot** | MEDIUM | D112 spec says LinkedIn drops should `createLead()` with extracted data. Not implemented. | Part of Surface 18 v1 build | M |
| **No outreach draft generation** | MEDIUM | D112 spec says LinkedIn drops should produce outreach in destination pill's voice. Currently, Claude just describes what it sees in the screenshot. | Part of Surface 18 v1 build | M |

---

## Scenario G: Drop a Scam-Chat Screenshot (Surface 18 v1)

**Operator action:** Pastes a Telegram scam-alert group chat screenshot.

### Code path (current state)

Same as Scenario F — image goes to vision content blocks, Claude responds in active pill's voice.

### What works

- Claude can see and describe the screenshot via vision
- If operator is on Vet pill, the response will have the skeptical Vet voice

### What's missing or broken (Surface 18 v1 NOT shipped)

| Gap | Severity | Detail | Fix | Effort |
|---|---|---|---|---|
| **No multi-entity extraction** | HIGH | D112 spec says scam-chat screenshots should extract entity names, determine sentiment per entity, check each against blacklist. None of this exists. Claude describes the screenshot but doesn't produce structured entity disposition. | Implement Surface 18 Route C per D107 spec | L |
| **No blacklist integration for image entities** | HIGH | Even if operator manually names entities from the screenshot, blacklist pre-check only fires for Vet pill text queries, not for entities extracted from images. | Part of Surface 18 v1 — vision classifier + per-entity blacklist check | M |
| **No "Add to blacklist?" action** | MEDIUM | D107 spec says negative-sentiment entities should get "Add to blacklist?" with one-click confirm. No action button UI exists. | New ScamChatEntityList component per D107 spec | M |

---

## Cross-Cutting Gaps (All Scenarios)

| Gap | Severity | Scenarios Affected | Detail |
|---|---|---|---|
| **No conversation memory** | HIGH | A, B, C, D, E | Each API call is single-shot. Follow-up questions have zero context. This is the #1 operator-blocker. |
| **No chat history persistence** | HIGH | All | Messages in React state only. Page refresh destroys everything. No localStorage, no server sessions. |
| **Single-line input** | MEDIUM | A, C, D, E | `<input type="text">` prevents multi-line entry. Operators can't paste chat transcripts, email threads, or multi-paragraph context. Helper text says "paste the whole conversation" but the input field is single-line. |
| **No /ask navigation link** | MEDIUM | All | /ask is unreachable from the main product UI. No Topbar link, no sidebar link. Operator needs to know the URL or bookmark it. |
| **No auth on /ask** | LOW | All | Fully public — anyone with the URL can use it. Rate limiting (100/hr per IP) is the only gate. Intentional for demo purposes but risky for production team use with real CRM data. |
| **explain-further + refine-redline bypass @milo/ai-client** | LOW | B | Raw Anthropic SDK with hardcoded model. No tier management, no retry, no spend tracking. |

---

## Top 5 HIGH Operator-Blockers (Ranked)

| Rank | Blocker | Scenarios | Impact | Fix | Effort |
|---|---|---|---|---|---|
| **1** | **No conversation memory** | A, B, C, D, E | Every follow-up is blind. "Tell me more" gets a blank-context response. Tiffani must re-explain everything each message. Kills the "intelligence layer" value prop. | Send `messages` history array to API. Claude sees prior turns. | M |
| **2** | **Per-issue decisions not persisted** | B | Accept/reject/edit states lost on refresh. Tiffani works through 11 issues, refreshes, all reset to pending. Contract review workflow is unusable beyond a single session. | Save issue states to Supabase keyed by `analysis_id + issue_id`. Load on re-render. | M |
| **3** | **No chat history persistence** | All | Refresh = empty page. No localStorage, no server sessions. If Tiffani accidentally refreshes mid-conversation, entire context is gone. | localStorage for chat history + optional server-side via chat_logs session grouping. | M |
| **4** | **No file picker button** | B, F, G | Only drag-and-drop and paste. Mobile users cannot attach files. Tiffani on her phone cannot drop a contract. | Add `<input type="file">` with paperclip icon trigger. | S |
| **5** | **Surface 18 classifier not shipped** | F, G | LinkedIn/scam-chat screenshots get no special handling — no entity extraction, no CRM lead creation, no blacklist integration. Image drops are just "describe what you see." | Implement D112-locked Surface 18 classifier. | L |

---

## Top 5 MEDIUM Friction Items (Ranked)

| Rank | Item | Scenarios | Impact | Fix | Effort |
|---|---|---|---|---|---|
| **1** | **Single-line input** | A, C, D, E | Cannot paste multi-line content. Chat transcripts, email threads, dispute details all flatten. | Replace `<input type="text">` with `<textarea>` + Shift+Enter for newlines. | S |
| **2** | **web_fetch tool not configured** | A, D | All 4 prompts promise web_fetch ("Grab LinkedIn pages"). Only web_search exists. Model may attempt a tool that doesn't exist. | Add web_fetch to tools array or remove from system prompts. | S |
| **3** | **No /ask navigation link** | All | /ask is an orphaned page. No way to reach it from /operator dashboard. | Add /ask link to Topbar navigation. | S |
| **4** | **Contract analysis locked to Vet pill** | B | Dropping a contract on Sales/Publisher/Buyer does nothing — no extraction, no analysis. Operator must know to select Vet first. | Surface 18 handles this (file-type routing). Interim: show toast "Switch to Vet for contract analysis." | S |
| **5** | **Drafted outreach not copyable** | C | Sales drafts live in chat bubbles. No "Copy" button. Tiffani manually selects and copies. | Add copy-to-clipboard button on assistant messages. | S |

---

## Recommended Directive Sequence

Based on operator impact and effort:

| Priority | Directive | Effort | Closes |
|---|---|---|---|
| **1** | **Conversation memory** — send message history to API | M | HIGH-1 (conversation memory), all follow-up scenarios |
| **2** | **Chat persistence + issue state persistence** — localStorage + Supabase issue states | M | HIGH-2 (issue decisions), HIGH-3 (chat history) |
| **3** | **Input + UX polish** — textarea, file picker button, copy button, /ask nav link | S | HIGH-4 (file picker), MEDIUM-1 (textarea), MEDIUM-3 (nav), MEDIUM-5 (copy) |
| **4** | **Tool alignment** — web_fetch config, blacklist signal expansion | S | MEDIUM-2 (web_fetch), Scenario A blacklist gap |
| **5** | **Surface 18 v1** — file classifier, cross-pill routing, entity extraction | L | HIGH-5 (classifier), Scenarios F + G |
| **6** | **Operational data access** — publisher performance, dispute history, call data injection | L | Scenario D + E medium gaps |

Directives 1-3 are pre-Monday critical. Directive 4 is same-day polish. Directives 5-6 are post-launch.

---

## Architecture Notes for Directive Authors

### Conversation memory implementation path

The cheapest path: page.tsx already stores `messages` in state. Modify `doSubmit` to send the last N messages (capped at ~4000 tokens) as a `conversationHistory` array in the request body. Route.ts passes them as `messages` parameter to `callClaudeStream`. No database changes needed — it's client-side context window management.

### Issue state persistence path

IssueListExpandable already has `issueStates` and `setIssueState(issueId, newState)`. Add a `useEffect` that debounce-saves states to Supabase: `UPDATE contract_analyses SET issue_states = $1 WHERE id = $analysisId`. On mount, load from the same row. The `contract_analyses` table already has the `id` column. Add a `issue_states JSONB` column.

### Chat persistence path

Save messages to localStorage keyed by a session ID (generate on first message, store in URL hash). On page load, check localStorage for an active session. Optionally group chat_logs rows by session_id for server-side persistence.

---

Cross-references: D116 (Vet Part 1), D117 (CRM context), D118 (ai-client wireup), D120 (paste handler), D106 (voice consistency), D107 (Surface 18 spec), D112 (Surface 18 locked decisions), D113/D114 (readiness audit).
