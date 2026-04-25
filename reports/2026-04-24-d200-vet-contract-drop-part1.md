# D200 — Vet Contract Drop Part 1

**Date:** 2026-04-24
**Agent:** Coder-1
**Status:** SHIPPED

## What shipped

Contract file drops (PDF/DOCX) on `/ask` Vet pill. Drop a contract → get structured analysis + Milo voice translation.

### Pipeline

1. Frontend detects PDF/DOCX drop, reads as base64, sends as `contractFile` in request body
2. Backend extracts text: `pdf-parse@1.1.1` for PDF, `mammoth` for DOCX
3. Direct Anthropic SDK call analyzes contract (Sonnet 4.6, structured JSON output)
4. Analysis result streams via SSE `analysis` event
5. Analysis context injected into Claude prompt for Milo voice translation
6. `analysis_id` persisted on `chat_logs`

### Frontend components

- `ContractAnalysisCard` with `SeverityBadge`, expandable issue list
- Binary file handling (ArrayBuffer → base64, 5MB limit for contracts)
- Auto-submit on contract drop in Vet mode

## Bugs fixed during ship

| Bug | Root cause | Fix |
|-----|-----------|-----|
| Build failure: `./index.js` not found | Turbopack doesn't resolve `.js` extensions on TS imports | Removed `.js` from all relative imports in `ppc/engine.ts`, `ppc/index.ts`, `vertical.config.ts` |
| `text-extraction.js` pulls pdfjs-dist/tesseract.js | Vendored barrel re-exports heavy deps | Stubbed `text-extraction.js`, excluded from barrel |
| `pdf-parse@2.4.5` → `DOMMatrix is not defined` | v2 bundles browser-dependent pdfjs-dist | Pinned to `pdf-parse@1.1.1` (Node.js compatible) |
| Engine 529 Overloaded | `modelTier: 'judgment'` → Opus 4.7, low capacity | Switched to `structured` tier (Sonnet) |
| Engine 15s timeout | `@milo/ai-client` default `withRetry` timeout | Patched vendored engine: 100s timeout, 1 retry |
| Vendored engine hangs on Vercel | Dynamic `import('@milo/ai-client')` module resolution in serverless | Bypassed engine — direct `sdk.messages.create()` in route |
| JSON truncation at 17552 chars | 4096 max_tokens too small for verbose analysis | Truncate input to 12K chars, max 8 issues, concise field limits, 6144 max_tokens |

## Decisions

- **Direct SDK over vendored engine** (Decision 104 amendment): The vendored `@milo/contract-analysis` engine's dynamic import chain (`engine.js → @milo/ai-client → @anthropic-ai/sdk`) has module resolution issues in Vercel's serverless bundler. Direct SDK call in the route avoids all this. The engine can be wired back in for the dedicated `/contracts` page where the full pipeline (with Supabase persistence, rule packs, scoring modifiers) is worth the complexity.
- **12K char truncation**: First ~6 pages of a typical MSA covers the critical sections (payment, termination, liability, compliance). Sufficient for inline /ask analysis. Full-document analysis belongs on the dedicated contract review page.

## Evidence

### API test (PDF)
```
HTTP 200 in 85.7s
Document: CrossBlade_Buyer_MSA.pdf | Role: buyer | Risk: HIGH | Score: 28/100
8 issues: 3 CRITICAL, 4 HIGH, 1 MEDIUM
```

### API test (DOCX)
```
HTTP 200 in 87.2s
Document: Remix_Publisher_MSA.docx | Role: publisher | Risk: HIGH | Score: 28/100
8 issues: 2 CRITICAL, 3 HIGH, 3 MEDIUM (est.)
```

### SQL evidence
```sql
SELECT id, pill, analysis_id, created_at FROM chat_logs
WHERE id = 'eb7ea96a-ab34-4997-9b6d-1ab7e0854f8b';
-- pill: vet, analysis_id: 57e39093-3c7b-43ea-b5ea-b38107727b61
```

## Commits

- `f87ec3c` feat: D200 Vet contract drop Part 1
- `8afd088` fix: pdf-parse API (PDFParse class)
- `5917733` fix: pin pdf-parse@1.1.1 (DOMMatrix)
- `8d1d9f8` fix: structured tier instead of judgment
- `69d24de` fix: engine timeout 90s
- `9f190ae` fix: reduce maxTokens, single retry
- `884951e` fix: bypass vendored engine, direct SDK
- `78f0c40` fix: 12K truncation, concise format
- `fac39fa` fix: clean error output

## What's NOT in Part 1

- Redline links (Part 2)
- Send/negotiate buttons (Part 3)
- Version routing
- Full-document analysis (dedicated `/contracts` page)
- Visual verification screenshots (Playwright drag/drop doesn't work for binary files in headless mode — tested via curl/API instead)
