# /ask Conversation Memory — Multi-Turn Context

**Date:** 2026-04-25
**Agent:** Coder-1
**Status:** COMMITTED + DEPLOYED + VERIFIED
**Source:** Coder-3 /ask end-to-end audit (HIGH-1 operator blocker)

## What shipped

Every /ask API call now includes conversation history from prior turns. Claude sees the full thread instead of blank context on each message. ~16K character cap prunes oldest messages first.

## Implementation

### page.tsx (+16 lines)

- Added `messagesRef` (useRef synced to messages state via useEffect)
- In `doSubmit`, builds `conversationHistory` from `messagesRef.current`:
  - Filters to completed user + assistant messages
  - Iterates newest-first, adding messages until ~16K char cap hit
  - Sends as optional `conversationHistory` array in POST body

### route.ts (+24/-10 lines)

- Added `conversationHistory` to body type (optional `Array<{role, content}>`)
- Filters history to valid roles (user/assistant)
- Builds `allMessages = [...historyMessages, currentUserTurn]`
- Current user turn can be text-only or image+text (vision blocks)
- Passes full array to `callClaudeStream` via `messages` parameter

## Visual verification — 4 scenarios

| Test | Scenario | Result | Evidence |
|------|----------|--------|----------|
| 1 | Multi-turn vet: "Vet Granite Media" → follow-up "payment history?" | Turn 2 references "Granite Media LLC", "multi-entity name confusion", "FIRMFUEL", "Irvine CA" from turn 1 | `d-convo-1-turn2-followup.png` |
| 2 | Cross-pill: Vet AppVista Digital → Sales "draft outreach" | Turn 2 references "AppVista Digital" and "5 out of 5" from vet, plus off-topic deflection | `d-convo-2-turn2-sales.png` |
| 3 | Long conversation: 6 turns of email drafting | Final "summarize" response references "TechLeads Corp", "asked for vertical four times" — full history retained | `d-convo-3-long-convo.png` |
| 4 | Image attachment: vet entity → paste screenshot → follow-up | Turn 3 references "Phoenix Call Solutions" from turn 1, "that screenshot" from turn 2 | `d-convo-4-image-history.png` |

Stitched screenshot: `/Users/markymark/Desktop/d-conversation-memory.png`

## Token cost analysis

| Test | Turn | Total tokens | Cache | Notes |
|------|------|-------------|-------|-------|
| 1 | Turn 2 (with history) | 1,660 | cache hit | History from 1 vet response (~1200 chars) |
| 2 | Turn 2 (with history) | 1,329 | cache hit | History from 1 vet response |
| 3 | Turn 6 (with history) | 758 | cache hit | 5 turns history, shorter response |
| 4 | Turn 3 (with history) | 920 | cache hit | 2 turns history including image reference |

Estimated overhead per turn: **200-800 additional input tokens** depending on conversation length. System prompt caching (`cache hit` confirmed in all 4 tests) offsets the cost — cached system prompt saves ~300-500 tokens per call.

## File scope verification

| File | Action | Lines |
|------|--------|-------|
| `src/app/ask/page.tsx` | Modified | +16 |
| `src/app/api/ask/route.ts` | Modified | +24/-10 |

Zero other files touched. No schema changes. No new dependencies. No new endpoints.

## Commit

- `274180c` feat: /ask conversation memory — send message history to API for multi-turn context

## Build

`npm run build` — clean, no errors, no warnings.

## What's NOT in this version

- Image blocks in history (prior images not re-sent as vision — only text descriptions carry)
- Persistent history across page refreshes (no DB storage)
- Per-pill history isolation (all history carries across pill switches)
- History compression for very long conversations (simple char cap only)
