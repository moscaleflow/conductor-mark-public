# Vet Research Quality Fix — Coder-1 Report

**Date:** 2026-04-25
**Commits:** 18c94ab (research quality fix), 401adde (missing team-roster + EntityVetCard deps), a0d4c93 (maxTokens 1024→2048)

---

## Problem

After the two-stage vet optimization (Decision 144), per-entity Sonnet research produced "Not found" for all facts on entities that Google finds immediately. Test case: "Matthew Barner / TopClass Leads LLC" — topclassleads.com is Google's first result, but Sonnet returned verdict "unclear" with every fact as "Not found."

**Root cause:** The entity-name fix (d9829c3) stripped ALL context from per-entity Sonnet calls to prevent entity confusion. But this overcorrected — a person's name alone without company affiliation is unsearchable. When Sonnet receives only "Vet this entity: Matthew Barner", it has no idea to search for "TopClass Leads."

---

## Changes

### 1. Associated entity context (route.ts)

Stage 2 user message now includes other primary entities from the same paste:

- Before: `"Vet this entity: Matthew Barner"`
- After: `"Vet this entity: Matthew Barner (associated with: TopClass Leads LLC)"`

`buildPerEntityPrompt` also receives the association list and injects an ASSOCIATED ENTITIES block into the system prompt.

### 2. max_uses 2→3 (route.ts)

`web_search` max_uses increased from 2 to 3 per entity. Allows an additional search round-trip for entity disambiguation. Worst-case adds ~9s per entity.

### 3. Mandatory 3-search minimum (vet-extraction.ts)

System prompt now explicitly requires 3 distinct search queries before concluding "Not found":
1. Entity name alone
2. Entity name + associated person/company
3. Entity name + industry keywords

Includes explicit instruction: "An entity that Google finds on the first result should never come back as 'Not found' from you."

### 4. Haiku extraction: person+company association rule (vet-extraction.ts)

EXTRACTION_PROMPT now instructs Haiku to include BOTH person and company as separate primary entries when they're clearly associated (e.g., "Matthew Barner / TopClass Leads LLC" → both as primary).

---

## Build & Deploy

- Local build: PASS (clean)
- First deploy (18c94ab): FAILED — `classifySecondaryNames` referenced `full_name` on `TeamRosterEntry` but team-roster.ts wasn't committed. Fixed with 401adde.
- Second deploy (401adde): PASS — Ready
- Third deploy (a0d4c93): PASS — Ready (maxTokens fix)

---

## Test Results

Playwright suite (5 tests): All PASS (59.0s)

### Production test: "Matthew Barner / TopClass Leads LLC"

| | Before (max_uses=2, no context) | After (max_uses=3, associated entities) |
|---|---|---|
| **Verdict** | "unclear" — all facts "Not found" | "likely_real" + "unclear" (2 cards) |
| **Facts found** | 0 | 9 per card (18 total) |
| **Flags** | 0 | 8-9 per card |
| **Key findings** | None | topclassleads.com, LinkedIn, ZoomInfo, LLC records, employment history |
| **Latency** | ~10s (gave up fast) | 27.3s (3 web searches per entity) |

Screenshot: `/Users/markymark/Desktop/d-vet-research-quality.png`

---

## Files Modified

| File | Change |
|---|---|
| `src/app/api/ask/route.ts` | Associated entity context in Stage 2 user msg + max_uses 2→3 |
| `src/lib/ask/prompts/vet-extraction.ts` | associatedEntities param, ASSOCIATED ENTITIES block, 3-search minimum, person+company extraction rule |
| `src/lib/team-roster.ts` | Added `full_name?` to TeamRosterEntry (dependency for classifySecondaryNames) |
| `src/components/ask/EntityVetCard.tsx` | Assign dropdown click-outside/Escape dismiss (other agent's work, committed for deploy) |
