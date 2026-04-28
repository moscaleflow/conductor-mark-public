# D193 — /ask Three Surgical Fixes

**Date:** 2026-04-24  
**Agent:** Coder-1  
**Scope:** Rate limits, empty-response safeguard, TTFT + tool progress UX

---

## Summary

Three targeted fixes to `/ask`:

1. **Rate limits raised + Milo voice** — 30→100/hr per IP, 100→1000/day, new 500/hr global limit via `__global__` key in `ask_rate_limits`. Error messages rewritten in Milo's voice.
2. **Empty-response safeguard** — If model returns <20 chars of text, `admin_flag=true` is set on the `chat_logs` row for review queue filtering.
3. **Cache latency breakdown** — TTFT tracked from first text delta, streamed via `ttftMs` in the `done` SSE event. New `status` SSE event type emits "Searching the web..." during `server_tool_use` content blocks. Frontend shows `Xs total · Ys TTFT · N tok · cache hit`.

---

## Commits

### milo-for-ppc

| Commit | Description |
|---|---|
| `5f62d1d` | fix: D193 /ask three fixes — rate limit 100/hr + Milo voice, empty-response safeguard, TTFT + tool progress |

---

## FIX 1 — Rate Limits + Milo Voice

### Thresholds

| Limit | Old | New |
|---|---|---|
| Per-IP hourly | 30 | 100 |
| Per-IP daily | 100 | 1,000 |
| Global hourly | — | 500 |

### Global limit mechanics

New `__global__` row in `ask_rate_limits` tracks cross-IP hourly usage. Checked after per-IP checks, incremented on every allowed request.

### Milo-voice error copy

| Trigger | Message |
|---|---|
| Hourly limit hit | "You're going fast. Slow down for a few minutes and I'll be ready again." |
| Daily limit hit | "You've hit the daily ceiling. Come back tomorrow and I'll be fresh." |
| Global limit hit | "I'm getting a lot of traffic right now. Give me a few minutes." |

### ask_rate_limits table state

```
ip_hash          | hourly_count | daily_count
b40dd548d63e7742 | 3            | 3
__global__       | 2            | 2
```

---

## FIX 2 — Empty-Response Safeguard

```typescript
const isEmptyResponse = fullText.trim().length < 20;
// ... 
admin_flag: isEmptyResponse,
```

Sets `admin_flag=true` on the `chat_logs` insert when the model returns fewer than 20 characters of streamed text. Works alongside D192's thumbs-down flagging — both feed the same review queue.

### Evidence — admin_flag rows

```json
[
  {
    "id": "271a8964-...",
    "pill": "vet",
    "input_text": "Tell me about MediaBridge",
    "admin_flag": true,
    "user_rating": "down",
    "created_at": "2026-04-24T20:42:54Z"
  },
  {
    "id": "15a6c551-...",
    "pill": "vet",
    "input_text": "Tell me about MediaBridge",
    "admin_flag": true,
    "user_rating": "down",
    "created_at": "2026-04-24T20:35:09Z"
  }
]
```

These rows were flagged via thumbs-down (D192). No organic empty-response cases have occurred yet — the safeguard fires on `fullText.trim().length < 20` at insert time.

---

## FIX 3 — TTFT + Tool Progress

### Server changes

- `firstTokenMs` tracked on first `text` event: `firstTokenMs = Date.now() - startMs`
- `streamEvent` listener on `content_block_start` detects `server_tool_use` → emits `{ type: 'status', text: 'Searching the web...' }` SSE event
- `done` event includes `ttftMs: firstTokenMs`

### Frontend changes

- New state: `lastTtft`, `statusText`
- `status` SSE event → sets purple italic status text below streaming placeholder
- `delta` event → clears status text (tool search finished, text streaming)
- Latency bar: `"X.Xs total · Y.Ys TTFT · N tok · cache hit"`
- Both values reset in `handleReset`

### Key technical detail

Anthropic SDK `MessageStream` emits `streamEvent` (not `event`). The `server_tool_use` block type appears in `content_block_start` events when web_search is executing.

---

## Visual Verification

Stitched screenshot: `/Users/markymark/Desktop/d193-ask-three-fixes.png` (1280×2440)

| Panel | Content |
|---|---|
| d193-01 | "Searching the web..." status in purple italic during tool execution |
| d193-02 | "22.4s total · 5.5s TTFT · 1309 tok · cache hit" latency breakdown |
| d193-03 | Normal buyer response working post rate-limit fix |

### Playwright coverage

Test file: `tests/d193-ask-three-fixes.spec.ts` — 3/3 passing

| Test | Verifies |
|---|---|
| Vet query with TTFT + tool progress | Status text "Searching the web", TTFT in latency display |
| Normal response after rate limit fix | Buyer pill query completes with thumbs |
| Verify screenshots | All 3 PNGs exist on Desktop |

---

## Files Changed

| File | Change |
|---|---|
| `src/app/api/ask/route.ts` | Rate limits, global limit, Milo voice, empty-response safeguard, TTFT tracking, status SSE event |
| `src/app/ask/page.tsx` | Status text display, TTFT in latency bar, state management |
| `tests/d193-ask-three-fixes.spec.ts` | Playwright visual verification |

---

## Decision

Logged as **Decision 94** in DECISION-LOG.md.
