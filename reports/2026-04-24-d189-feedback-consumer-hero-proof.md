# D189 — @milo/feedback Consumer Integration: Hero Email Capture + /proof Reactions

**Date:** 2026-04-24  
**Agent:** Coder-1  
**Scope:** 2 consumer integrations + 2 new context types

---

## Summary

Wired @milo/feedback as a signal layer on two existing surfaces. Hero email capture now fires an `email_capture` feedback signal alongside the CRM lead. /proof page replaces FeedbackThumbs (thumbs icons) with muted "Useful" / "Not quite" text buttons using `reaction` context type. No schema changes — both write to existing `feedback_signals` table.

---

## Commits

### milo-for-ppc

| Commit | Description |
|---|---|
| `61fdf33` | feat: D189 @milo/feedback consumer — hero email capture signal + /proof reaction buttons |

---

## Architecture

### 1. New VALID_CONTEXT_TYPES

Added to `/api/feedback/record`:
- `email_capture` — hero funnel email submissions
- `reaction` — /proof page "Useful" / "Not quite" buttons

### 2. Hero email capture feedback signal

Both `handleCapture` (PPC role) and `handleOtherCapture` (other industry) now fire a `void fetch` to `/api/feedback/record` after the CRM lead is created:

```typescript
void fetch('/api/feedback/record', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    contextType: 'email_capture',
    contextId: `hero-${Date.now()}`,
    signal: 'up',
    submittedBy: email,
    metadata: { vertical: 'ppc', source: 'hero', role },
  }),
})
```

- Fire-and-forget (`void`) — does not block the capture flow
- `submittedBy` is the email itself (not 'anonymous')
- Metadata captures vertical, source, and role for segmentation

### 3. /proof page reaction buttons

Replaced `FeedbackThumbs` component (lucide-react thumbs up/down) with inline text buttons:

- **"Useful"** → signal: `up`, hover: green border glow
- **"Not quite"** → signal: `down`, hover: red border glow
- After vote: shows "Noted." in muted gray
- Muted styling: 11px, #6b7280 text, #1e1e2e border, 4px 10px padding
- Matches /proof's dark aesthetic (no icons, no color until hover)

API call fires from the `handleVote` callback in `RealContractsPage`:
```typescript
void fetch('/api/feedback/record', {
  method: 'POST',
  body: JSON.stringify({
    contextType: 'reaction',
    contextId,
    signal,
    metadata: { source: 'proof-page', vertical: 'ppc', reaction: signal === 'up' ? 'useful' : 'not-quite' },
  }),
})
```

### 4. FeedbackThumbs import removed from /proof

/proof no longer imports `FeedbackThumbs` from `@/components/FeedbackThumbs`. The component remains available for other pages (hero demo results still use it).

---

## Files changed

| File | Change |
|---|---|
| `src/app/api/feedback/record/route.ts` | Added `email_capture`, `reaction` to VALID_CONTEXT_TYPES |
| `src/app/page.tsx` | Added feedback signal fire to handleCapture + handleOtherCapture |
| `src/app/proof/page.tsx` | Replaced FeedbackThumbs with "Useful" / "Not quite" text buttons |

---

## No schema changes

Both new context types write to the existing `feedback_signals` table (Supabase project: tappyckcteqgryjniwjg). The `context_type` column is TEXT, not an enum — no migration needed.

---

## Verification

### Build: PASS
```
npm run build — clean, no warnings
```

### Visual verification: pending
Screenshots to be captured after Vercel deploy settles.

---

## Decision Log

Decision 91 added to DECISION-LOG.md: "@milo/feedback consumer integration — hero email capture + /proof reactions"

---

## Deploy URL

https://tlp.justmilo.app/proof
