# D101 Report: /ask Wired to @milo/ai-client + @milo/blacklist

**Agent:** Coder-2 | **Date:** 2026-04-24 | **Status:** Code complete, build clean, awaiting commit approval

---

## Summary

/ask route eliminated raw Anthropic SDK (`new Anthropic()`) and raw blacklist Supabase query. All three AI calls (classifier, contract analysis, streaming) now route through @milo/ai-client with tier-based model selection and MODEL_QUIRKS. Blacklist pre-check now uses `screenEntity()` from @milo/blacklist — fuzzy matching, aliases, and 343 entries instead of 0.

---

## Changes

### 1. `src/app/api/ask/route.ts` (-42 LOC net)

**Imports replaced:**
- `import Anthropic from '@anthropic-ai/sdk'` → `import { callClaude, callClaudeStream, TIER } from '@milo/ai-client'`
- Added `import { screenEntity } from '@/lib/blacklist-client'`
- Added `extractEntityNames` to crm-context import

**Classifier (Part A1):**
- `sdk.messages.create({ model: 'claude-haiku-4-5-20251001', ... })` → `callClaude('', { tier: 'classify', ... })`
- Few-shot messages preserved via `messages` parameter
- `classifyInput(message, sdk)` → `classifyInput(message)` (no SDK parameter)

**Contract analysis (Part A2):**
- `sdk.messages.create({ model: 'claude-sonnet-4-6', max_tokens: 6144, ... })` → `callClaude(prompt, { tier: 'structured', maxTokens: 6144, ... })`
- `callClaude` returns text directly — eliminated `Anthropic.TextBlock` filter

**Main streaming (Part A3):**
- `sdk.messages.stream({ ... })` + manual `on('text')` + `on('streamEvent')` + `finalMessage()` → `callClaudeStream(prompt, opts, onChunk)`
- `cacheSystemPrompt: true` replaces manual `cache_control` array construction
- Model resolved from `TIER.structured` instead of `finalMessage.model`

**Blacklist (Part B):**
- Entire `blacklistPreCheck` body replaced
- Was: load 500 rows from legacy `blacklist` table, substring + partial word match
- Now: extract entity names via `extractEntityNames()`, call `screenEntity({ company_name })` per candidate
- Gains: fuzzy matching (Levenshtein < 3), alias matching, contact name matching, email domain matching

### 2. `src/lib/ask/crm-context.ts` (+1 word)

- `extractEntityNames` exported (was private). Shared between CRM context and blacklist screening.

---

## What /ask gains

| Capability | Before | After |
|---|---|---|
| Model routing | Hardcoded model strings | TIER table (classify→Haiku, structured→Sonnet) |
| MODEL_QUIRKS | None — raw SDK | Active (auto-strips deprecated params per model) |
| Prompt caching | Manual `cache_control` array | `cacheSystemPrompt: true` |
| Blacklist table | Legacy `blacklist` (0 rows) | `blacklist_entries` (343 rows, tenant='tlp') |
| Blacklist matching | Exact substring + partial word | Exact + normalized + fuzzy (Levenshtein) + aliases + contacts + email domains |
| Error handling | Manual try/catch | ai-client withRetry (classifier + analysis) |

## Minor regression

"Searching the web..." status message during web_search tool use is lost. `callClaudeStream` doesn't expose stream events beyond text chunks. The web search still works — only the loading indicator is missing. UX polish, not functionality.

---

## Tool-Reuse Audit

| Method | Package | Usage |
|---|---|---|
| `callClaude` | @milo/ai-client | Classifier (tier='classify') + contract analysis (tier='structured') |
| `callClaudeStream` | @milo/ai-client | Main streaming response (tier='structured') |
| `TIER` | @milo/ai-client | Model name for chat_logs + done event |
| `screenEntity` | @/lib/blacklist-client (wraps @milo/blacklist) | Blacklist pre-check with fuzzy matching |
| `extractEntityNames` | @/lib/ask/crm-context | Entity extraction for blacklist screening |

**New SDK methods added: 0**
**Primitives modified: 0**

---

## Build status

Build passes clean. No TypeScript errors.

---

## LOC Breakdown

| Component | Insertions | Deletions | Net |
|---|---|---|---|
| route.ts | 58 | 100 | -42 |
| crm-context.ts | 1 | 1 | 0 |
| **Total** | **59** | **101** | **-42** |
