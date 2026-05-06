# Operator UX Deep Audit -- /ask Pills + Operator Surfaces

**Date:** 2026-05-06
**Auditor:** Coder-3 (Research)
**Scope:** All pill prompts, StructuredResponseCard, EntityCard flow, chat persistence, PillDrawer, FlaggedCallsTab, API endpoints

---

## CRITICAL -- Bugs that will crash or produce wrong behavior in production

### C-1. Blacklist endpoint URL does not exist
**File:** `src/components/ask/StructuredResponseCard.tsx` line 368
**What:** `handleBlacklist()` calls `fetch('/api/blacklist/add', ...)` but no route exists at `/api/blacklist/add/route.ts`. The only blacklist POST endpoint is at `/api/blacklist` (root). The fetch will return 404, the catch block silently resets `blacklistState` to `idle`, and the operator sees the button reset with no feedback. The entity is never actually blacklisted.
**Impact:** Operators clicking "Add to Blacklist" on a likely_scam entity will think it worked (button resets) but the entity is NOT blacklisted. Fraudulent entities remain unblocked.
**Fix:** Change `/api/blacklist/add` to `/api/blacklist` on line 368. Also, the body shape differs -- `/api/blacklist` POST expects `company_name` and `reason` (required), while the component sends `entity_name` and `reason`. Map `entity_name` to `company_name`.

### C-2. Structured responses lost on page refresh
**File:** `src/app/api/ask/history/route.ts`
**What:** The history endpoint restores only `response_text` (raw text) and `vetResults` from `vet_results` table. It does NOT restore `structuredResponse`, `opportunityMatches`, `analysis`, or `vetSecondaryNames`. When a user refreshes the page mid-session, all formatted card responses from sales/buyer/publisher pills revert to either empty content or raw JSON text.
**Impact:** After any page refresh, operator sees raw JSON or empty responses where they previously had formatted outreach drafts, research cards, or entity lists. Entire session work product becomes unreadable.
**Fix:** Either (a) persist `structuredResponse` JSON in a column on `chat_logs` (or a joined table) and restore it in the history endpoint, or (b) re-parse `response_text` through `extractStructuredResponse()` during history hydration on the client side. Option (b) is simpler and requires no DB migration.

### C-3. Pipeline stage silently dropped on CRM add
**File:** `src/components/ask/StructuredResponseCard.tsx` line 271-277 vs `src/app/api/ask/entity-add/route.ts`
**What:** `EntityCard.addToCrm(stage)` sends `pipeline_stage` in the request body (e.g., `'vetted'` or `'unverified'`), but `/api/ask/entity-add` destructures only `entity_name`, `entity_type`, `website`, `source`. The `pipeline_stage` value is silently ignored. All entities enter the CRM with no pipeline stage regardless of vet verdict.
**Impact:** Entities marked as "vetted" (likely_real) and "unverified" (unclear) both enter the CRM in the same state. The pipeline stage distinction that the auto-CRM-add flow was designed to preserve is lost.
**Fix:** Read `pipeline_stage` from the request body in `entity-add/route.ts` and pass it through to `findOrCreateCounterparty()`, or store it as a CRM field during creation.

---

## HIGH -- UX issues that will confuse operators

### H-1. No error feedback on blacklist failure
**File:** `src/components/ask/StructuredResponseCard.tsx` lines 364-382
**What:** The `handleBlacklist` catch block only resets `blacklistState` to `'idle'` -- no error message, no toast, no visual indicator. Since C-1 means this ALWAYS fails (404), the operator sees the button flicker and return to "Add to Blacklist" with no explanation.
**Fix:** Add error state handling similar to the vet error pattern. Show "Blacklist failed - Retry" text.

### H-2. Entity discovery section detection: dead code + false positive risk
**File:** `src/components/ask/StructuredResponseCard.tsx` lines 16-19, 1005-1031
**What:** The `isEntityDiscoverySection()` function (checks label for "TARGET LIST") is defined but NEVER called. Entity card rendering is triggered solely by `parseDiscoveredEntities()` finding `### N.` patterns in ANY section content, regardless of section label. This means:
1. A research section that happens to use `### 1.` numbered headings will incorrectly render as entity cards with Vet/Draft buttons
2. The `collapsed: false` rule in pill prompts for target list sections is irrelevant since the section label isn't checked
**Fix:** Gate entity card rendering on BOTH `parseDiscoveredEntities()` returning results AND `isEntityDiscoverySection(section.label)` returning true. This prevents false positives from numbered markdown in non-discovery sections.

### H-3. Truncation applied to collapsible sections even when expanded
**File:** `src/components/ask/StructuredResponseCard.tsx` line 1071
**What:** When a collapsible section is expanded, the content is still passed through `truncateToMaxWords(section.content)` which caps at 200 words. The user clicks "Expand" expecting full content, but the detail section is truncated to 200 words with "..." appended. The collapsed preview (2-line clamp) and the expanded view are both truncated.
**Impact:** Operators cannot read full detail sections in research responses. Information cut off at 200 words with no indication that more exists.
**Fix:** Only apply `truncateToMaxWords()` to the collapsed preview (line 1000), not to the expanded content rendering at line 1071. When expanded, render `section.content` directly.

### H-4. History restores vet cards with `content: ''`, hiding any prose summary
**File:** `src/app/api/ask/history/route.ts` line 162
**What:** When a history row has vet results, the assistant message content is set to empty string: `content: hasVetCards ? '' : (row.response_text || '')`. But the original response may have contained prose text ALONGSIDE vet cards (e.g., the vet prompt's "CONTRACT VET MODE" outputs prose + cards). This prose is permanently lost on history restore.
**Fix:** Always include `response_text` as content, let the UI decide what to show based on whether vetResults exist.

### H-5. CRM batch check uses exact name match -- misses common variations
**File:** `src/app/api/ask/entity-check/route.ts` line 37
**What:** The entity-check endpoint uses `.in('company', capped)` which is an exact match (case-sensitive PostgreSQL `IN`). The client-side `useEntityCrmStatus` passes entity names from Claude's discovery format which may differ in casing or suffixes from CRM records. Example: Claude returns "Granite Media LLC" but CRM has "Granite Media" -- the batch check returns `exists: false`, showing no "IN CRM" badge, even though the entity exists.
**Fix:** Use `ilike` with fuzzy matching, or normalize names on both sides before comparison. The `normalizeForDedup` function on the client already strips suffixes -- apply similar logic server-side.

### H-6. Draft outreach error state has no retry mechanism
**File:** `src/components/ask/StructuredResponseCard.tsx` line 634-636
**What:** When draft outreach fails, the UI shows "Draft failed. Click to retry." but this text is a plain `<p>` element, NOT a button. There is no click handler. The only way to retry is to click the "Draft Outreach" button again, which is still visible but shows "Drafts Ready" label from the previous draftState.
**Impact:** Misleading -- tells the operator to "click to retry" but clicking the text does nothing.
**Fix:** Either make the error text a button that calls `handleDraftOutreach()`, or update the "Draft Outreach" button text to show a retry state when `draftState === 'error'`.

### H-7. Resolve note input in FlaggedCallsTab: dispute action broken by ID prefix check
**File:** `src/components/operator/FlaggedCallsTab.tsx` lines 493-496
**What:** The `onKeyDown` handler for the resolution note input checks `resolveNoteId.startsWith('dispute-')` to determine whether to resolve or dispute. But `resolveNoteId` is set to `item.id` (a UUID), never to a string starting with `'dispute-'`. The Enter key handler will ALWAYS use `'resolve'` action, even if the operator intended to dispute.
**Impact:** Pressing Enter in the resolve note input always resolves, never disputes. The dispute button works correctly because it calls `handleAction` directly, but the keyboard shortcut is wrong.
**Fix:** Store the intended action alongside the resolveNoteId, or use a separate state variable for dispute note entry.

---

## MEDIUM -- Edge cases that should be handled

### M-1. All /api/* routes bypass auth middleware
**File:** `src/middleware.ts` line 37
**What:** The middleware explicitly skips auth for all `/api/*` routes. This means every API endpoint (entity-add, entity-vet-inline, draft-outreach, flagged-calls PATCH, blacklist POST, rating PATCH, etc.) is accessible without authentication. Anyone with the URL can add entities to CRM, trigger AI vets, resolve flagged calls, or add blacklist entries.
**Impact:** No authentication on mutation endpoints. Any external party can manipulate CRM, blacklist, and flagged call data.
**Note:** This may be intentional given the current deployment model (internal tool behind network controls), but should be documented as a known risk. If the tool is ever exposed more broadly, this is a critical security gap.

### M-2. Race condition: two entity cards can trigger vet simultaneously
**File:** `src/components/ask/StructuredResponseCard.tsx`
**What:** The `EntityCardList` renders multiple `EntityCard` components, each with independent `handleVet` state. If an operator clicks "Vet Entity" on two cards in rapid succession, both will fire concurrent `entity-vet-inline` requests. There is no global concurrency guard -- only per-card `vetState === 'loading'` check.
**Impact:** Two simultaneous Claude API calls with web_search tools. Not a crash, but doubles API cost and may hit rate limits. Both will also attempt `addToCrm()` on completion, potentially creating duplicate CRM entries if the entity-add dedup guard has a race window.
**Fix:** Add a global vetting semaphore in `EntityCardList` that allows only one vet at a time, queuing others.

### M-3. `addingRef` guard prevents retry after CRM add failure
**File:** `src/components/ask/StructuredResponseCard.tsx` lines 262-300
**What:** The `addingRef.current = true` is set at the start of `addToCrm()`. On success, it stays `true` (preventing re-add, correct). On failure, it is reset to `false`. However, the failure path also calls `setTimeout(() => setCrmState('idle'), 3000)` which allows retry after 3 seconds. The problem: `addingRef.current` is properly reset on error, but the `crmState` check at line 264 (`crmState === 'added' || crmState === 'adding'`) means during those 3 seconds of error state, a new vet completion could try to auto-add and be blocked because `crmState === 'error'` is not in the guard. Actually wait -- `crmState === 'error'` is NOT blocked by the guard. So auto-add from a second vet could trigger during the 3-second error window.
**This is actually OK** -- the guard works. Lower priority.

### M-4. Session gap heuristic can split a session incorrectly
**File:** `src/app/api/ask/history/route.ts` lines 109-119
**What:** The 30-minute gap logic walks backwards and finds the FIRST gap > 30 minutes. If the operator has two gaps in a day (e.g., lunch break at noon, coffee break at 3pm), only messages after the LAST gap are restored. Older session work is permanently hidden.
**Impact:** Operator loses access to earlier research in the same day. Not a crash, but the expected behavior (see all today's messages) differs from actual behavior (see only most recent session).
**Fix:** Consider showing all today's messages with session separators rather than hiding older sessions. Or allow a URL param to override the session filter.

### M-5. Entity names with special characters not escaped in API calls
**File:** `src/components/ask/StructuredResponseCard.tsx` lines 268-276
**What:** Entity names are sent directly in JSON body strings without sanitization. Names containing quotes, backslashes, or control characters could cause JSON parse errors on the server. While `JSON.stringify` handles this for the request body, the `entity.body.slice(0, 500)` in `handleDraftOutreach` (line 338) could include malformed markdown that confuses Claude's response.
**Impact:** Low likelihood but possible: entity names like `O'Brien & Associates "LLC"` could cause unexpected behavior in downstream prompts or CRM lookups.

### M-6. FlaggedCallsTab counts can drift from actual state
**File:** `src/components/operator/FlaggedCallsTab.tsx` lines 153-163
**What:** After a resolve/dispute action, the component manually decrements `counts.flagged` and increments the corresponding status count. If two operators are working simultaneously, or if the initial counts were wrong (e.g., network race), the displayed counts drift from the real data. The component never re-fetches counts after mutations.
**Fix:** Re-fetch from `/api/operator/flagged-calls` after each successful action, or at minimum sync counts with the actual filtered array lengths.

### M-7. `Buyer Target List` label detection: `isEntityDiscoverySection` is not used (repeat from H-2)
The `isEntityDiscoverySection` check at line 16-18 handles "Buyer Target List" correctly (matches on "TARGET LIST" substring), but since this function is never called, there's no label-based guard. All entity detection relies on content parsing. This means if Claude uses "Key Buyers" as a section label with `### 1. Company` formatting, it would still render as entity cards with Vet/Draft buttons -- inappropriate for a buyer performance section.

### M-8. History does not restore `structuredResponse` -- messages with structured cards show raw text
**File:** `src/app/ask/page.tsx` lines 435-447 vs `src/app/api/ask/history/route.ts` lines 144-179
**What:** The history restoration on the client maps `m.content` to `content` but never populates `structuredResponse`. The `response_text` stored in `chat_logs` for sales/buyer/publisher pills IS the raw JSON that Claude returned. On refresh, this JSON text gets rendered as plain text in the message bubble rather than being parsed into a `StructuredResponseCard`. (Closely related to C-2 but from the client perspective.)
**Fix:** In the client-side history restoration (page.tsx ~line 435), attempt `extractStructuredResponse(m.content)` for non-vet pills and populate `msg.structuredResponse` if valid.

---

## LOW -- Cleanup / consistency items

### L-1. Dead code: `isEntityDiscoverySection` function
**File:** `src/components/ask/StructuredResponseCard.tsx` lines 16-19
**What:** Function defined, exported to nobody, called by nobody. Should either be integrated into the rendering logic (see H-2) or removed.

### L-2. `onVetEntity` and `onDraftOutreach` props defined but inconsistently passed
**File:** `src/components/ask/StructuredResponseCard.tsx` lines 167-171
**What:** `StructuredResponseCardProps` includes `onVetEntity` but it is never used inside the component. The prop is passed from `page.tsx` but the card never renders a "Vet" button that calls it. The entity discovery cards have their own inline vet mechanism.
**Fix:** Remove `onVetEntity` from the props interface, or wire it to a UI element.

### L-3. `data_source` field not handled for outreach_draft or general types
**File:** `src/components/ask/StructuredResponseCard.tsx` lines 930-938, 968-974
**What:** The data source dot (green/gold/gray) is only rendered when `isResearchList` is true. The pill prompts tell Claude to include `data_source` on research_list sections only, so this is currently consistent. However, if a future pill type adds `data_source`, the UI won't show it.
**Impact:** None currently. Consistency item for future-proofing.

### L-4. `TYPE_LABELS` and `TYPE_COLORS` don't match vet rendering path
**File:** `src/components/ask/StructuredResponseCard.tsx` lines 141-157
**What:** `entity_vet` and `call_prep` have labels/colors defined in `TYPE_LABELS`/`TYPE_COLORS` but vet responses take a completely separate rendering path (EntityVetCard, not StructuredResponseCard). `call_prep` is defined in the type system but no pill prompt actually produces it. These are harmless but suggest dead schema values.

### L-5. `MAX_SECTION_WORDS` constant: 200 words is very short for expanded detail
**File:** `src/components/ask/StructuredResponseCard.tsx` line 74
**What:** The truncation guard caps at 200 words. Given that pill prompts also instruct Claude to keep detail sections under 200 words, this is a redundant guard. However, if Claude exceeds the limit (which happens), the truncation silently clips the response.
**Impact:** Minor -- the prompt enforcement usually works. But when it doesn't, the user loses content with no indication.

### L-6. `handleSnoozeWithDuration` calculates `snoozedUntil` but never sends it
**File:** `src/components/operator/PillDrawer.tsx` lines 412-413
**What:** The function computes `const snoozedUntil = new Date(Date.now() + hours * 60 * 60 * 1000).toISOString()` but this value is never included in the fetch body. The API receives the snooze request but not the duration/until time.
**Impact:** All snooze durations (1h, 4h, Tomorrow, Next week) produce the same server-side behavior. The duration selection UI gives the operator a false sense of control.
**Fix:** Include `snoozed_until` in the POST body and handle it server-side.

### L-7. `onMarkAsSent` callback: button prevents re-send but doesn't prevent re-submission on page restore
**File:** `src/components/ask/StructuredResponseCard.tsx` lines 714, 1136-1239
**What:** The "Mark as Sent" button uses local `useState(false)` for `markedSent`. On page refresh, the state resets and the operator can mark as sent again, creating duplicate action items and outreach records.
**Fix:** Check the `ai_action_log` on mount to determine if this chat log's outreach was already marked as sent.

### L-8. Inconsistent import: `supabaseAdmin` vs `getSupabaseAdmin()`
**File:** `src/app/api/blacklist/route.ts` line 2 uses `import { supabaseAdmin }` (direct import of singleton).
**Other routes** use `import { getSupabaseAdmin }` (factory function).
**Impact:** Both work, but the inconsistency could cause issues if the initialization pattern changes.

---

## Summary

| Severity | Count | Key Items |
|----------|-------|-----------|
| CRITICAL | 3 | Blacklist endpoint 404, structured responses lost on refresh, pipeline_stage silently dropped |
| HIGH | 7 | No blacklist error feedback, dead entity detection, truncation on expand, vet prose lost, CRM name mismatch, draft retry broken, dispute keyboard shortcut wrong |
| MEDIUM | 8 | No API auth, concurrent vet race, session gap heuristic, entity name escaping, count drift, target list false positives, client history hydration gap |
| LOW | 8 | Dead code, unused props, missing data_source, dead type values, word cap, snooze duration ignored, sent state not persisted, import inconsistency |

**Top 3 fixes for immediate operator impact:**
1. **C-1:** Fix blacklist endpoint URL (`/api/blacklist/add` -> `/api/blacklist`) + body field mapping -- operators are clicking "Blacklist" on scam entities and nothing happens
2. **C-2 / M-8:** Restore structured responses on page refresh -- operators lose all formatted research cards on any navigation
3. **H-3:** Remove truncation from expanded sections -- operators cannot read full detail content

---

Report at: https://raw.githubusercontent.com/moscaleflow/conductor-mark-public/main/reports/operator-ux-audit-2026-05-06.md
