# Report: Surface 18 v1 — Pattern 15 Verification + Fix

**Agent:** Coder-2 | **Date:** 2026-04-24 | **Status:** VERIFIED + FIX SHIPPED (8856e0d)

---

## Summary

Audited Surface 18 v1 classify-file endpoint (commit 8d6b7b2) against D101 @milo/ai-client wireup pattern and Pattern 15 model-quirks protection. Found one violation: prompt caching not enabled, and the vision primitive didn't support it. Fixed both.

---

## Audit Results

| Check | Result |
|-------|--------|
| No `new Anthropic(` or `import Anthropic` in classify-file | PASS — zero matches |
| Uses `callClaudeVision` from `@milo/ai-client` | PASS |
| `tier: 'classify'` (Haiku) | PASS |
| MODEL_QUIRKS applied via ai-client `resolveModel` | PASS |
| `cacheSystemPrompt: true` for system prompt | **FAIL** — not set |
| Error logging includes full upstream detail | Improved — was `err.message`, now full `err` |

---

## Files Audited

### `src/app/api/ask/classify-file/route.ts` (Surface 18 v1)
- Imports `callClaudeVision` from `@milo/ai-client` (line 2)
- Uses `tier: 'classify'` (line 48) — routes to Haiku
- PDF/DOCX fast-path: deterministic classification, no API call (lines 28-38)
- Image path: calls `callClaudeVision` with vision content blocks (lines 47-54)
- JSON parse fallback: returns safe defaults on parse failure (lines 69-76)
- **Missing:** `cacheSystemPrompt: true` — system prompt (~29 lines) sent uncached every call

### `src/lib/ask/classifier-prompt.ts`
- Clean. Exports `FILE_CLASSIFIER_PROMPT` constant. No SDK usage.

### `src/lib/ask/classifier-types.ts`
- Clean. Type definitions only. `ClassifierResult` + `ClassifierDestination`.

### `vendor/ai-client/src/vision.ts` (primitive gap)
- Line 33: `...(options.system ? { system: options.system } : {})` — passes system as raw string
- Does NOT check `options.cacheSystemPrompt` — flag ignored even if set
- `client.ts` implements this via `buildSystemParam()` — vision.ts was missing parity

---

## Fix Applied (8856e0d)

### `vendor/ai-client/src/vision.ts` (+5 LOC)
Added `cacheSystemPrompt` support matching `client.ts` pattern:
```typescript
const systemParam = options.system
  ? options.cacheSystemPrompt
    ? { system: [{ type: 'text' as const, text: options.system, cache_control: { type: 'ephemeral' as const } }] }
    : { system: options.system }
  : {};
```

### `src/app/api/ask/classify-file/route.ts` (+1 LOC, 1 LOC improved)
- Added `cacheSystemPrompt: true` to `callClaudeVision` options
- Changed error log from `err.message` to full `err` object

### Build
`npm run build` — clean, no warnings.

---

## Architecture Notes

- `vision.ts` does not use `buildParams()` from `client.ts`, so MODEL_QUIRKS is not applied. For `classify` tier (Haiku), no quirks exist today, so no functional impact. If Haiku ever gets quirks, vision.ts would need the same `buildParams` refactor. Noted but not fixed — that's scope creep beyond this verification.

---

## Commit

`8856e0d` — `fix: Surface 18 classify-file — enable cacheSystemPrompt + fix vision.ts to support it`
