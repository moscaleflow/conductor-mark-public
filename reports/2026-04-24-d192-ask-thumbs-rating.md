# D192 — /ask Thumbs Rating: Silent Quality Signal

**Date:** 2026-04-24  
**Agent:** Coder-1  
**Scope:** Schema migration + PATCH endpoint + frontend rating UI

---

## Summary

Added silent thumbs up/down rating to every completed /ask response. SVG stroke-only thumbs render after stream finishes. Click fills with color, shows "Thanks.", fires PATCH to update the chat_logs row. Thumbs-down sets `admin_flag=true` for review queue filtering. One-shot rating — no toggle, no undo, no popup.

---

## Commits

### milo-for-ppc

| Commit | Description |
|---|---|
| `4fef3c7` | feat: D192 /ask thumbs rating — silent quality signal on every response |

---

## Schema

Migration `20260424200001_ask_rating_columns.sql` adds to `chat_logs`:

| Column | Type | Default | Purpose |
|---|---|---|---|
| `user_rating` | `TEXT CHECK ('up','down')` | NULL | Rating value |
| `rating_at` | `TIMESTAMPTZ` | NULL | When rated |
| `admin_flag` | `BOOLEAN` | `false` | `true` on thumbs-down for review queue |

Applied via `supabase db push --linked`.

---

## Architecture

### PATCH /api/ask/rating

New endpoint at `src/app/api/ask/rating/route.ts`:
- Accepts `{ chatLogId: string, rating: 'up' | 'down' }`
- Updates `user_rating` and `rating_at` on the chat_logs row
- Sets `admin_flag = true` when `rating === 'down'`
- Returns `{ ok: true }`

### /api/ask route change

Changed chat_logs insert from fire-and-forget (`void`) to awaited with `.select('id').single()`. The `done` SSE event now includes `chatLogId` so the frontend can rate the response.

### Frontend: RatingThumbs component

Inline component in `src/app/ask/page.tsx`:
- Renders after `msg.completed && msg.chatLogId` (stream finished + log ID received)
- Two SVG thumbs (14px, stroke-only, `rgba(255,255,255,0.25)`)
- Hover: up → `rgba(74,222,128,0.7)`, down → `rgba(239,68,68,0.7)` via CSS classes
- Clicked: chosen thumb fills with hover color, other thumb fades to 0.15 opacity
- "Thanks." label fades in (200ms animation)
- One-shot: rating state locks after click, button disabled
- No popup, no modal, no comment field

### No emoji in chrome

Zero emoji. All icons are SVG only. Confirmed via visual inspection of all 4 screenshots.

---

## Evidence

### SQL verification

```json
[
  { "id": "adf80495-...", "pill": "sales", "user_rating": "up", "admin_flag": false },
  { "id": "271a8964-...", "pill": "vet", "user_rating": "down", "admin_flag": true }
]
```

### curl PATCH

```
curl -X PATCH https://tlp.justmilo.app/api/ask/rating \
  -H "Content-Type: application/json" \
  -d '{"chatLogId":"4c1a733f-...","rating":"up"}'
→ {"ok":true}
```

---

## Verification (Pattern 16)

### Playwright tests: 5/5 pass (1.4m)

| Test | Result |
|---|---|
| Submit and see thumbs — desktop | PASS |
| Click thumbs-up — desktop | PASS |
| Click thumbs-down — desktop | PASS |
| Thumbs — mobile | PASS |
| Verify screenshots | PASS |

### Screenshots

- `d192-01-ask-thumbs-unrated.png` — muted thumbs visible after completed vet response
- `d192-02-ask-thumbs-up-clicked.png` — green filled thumb, "Thanks." visible
- `d192-03-ask-thumbs-down-clicked.png` — red filled thumb, "Thanks." visible
- `d192-04-ask-thumbs-mobile.png` — thumbs reachable on 375px viewport

**Stitched:** `/Users/markymark/Desktop/d192-ask-thumbs-rating.png` (354KB)

---

## Decision Log

Decision 93 added to DECISION-LOG.md: "/ask thumbs rating on responses — silent quality signal, admin_flag on thumbs-down for review queue"

---

## Deploy URL

https://tlp.justmilo.app/ask
