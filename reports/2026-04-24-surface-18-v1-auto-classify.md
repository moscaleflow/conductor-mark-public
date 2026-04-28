# Surface 18 v1 — Auto-Classify + Auto-Fire on File Drop/Paste

**Date:** 2026-04-24
**Agent:** Coder-1 (2 sub-agents via Pattern 23 fan-out)
**Status:** COMMITTED (pending deploy + visual verification)

## What shipped

File drop or paste on `/ask` now triggers automatic classification. HIGH-confidence results auto-fire in the correct pill. Cross-pill routes show inline confirmation. LOW-confidence pre-fills input with suggested prompt.

### Decision 38 sequence

1. SCHEMA.md amendment committed (`0256963`)
2. Migration applied via `supabase db push` (`5083954`)
3. Implementation committed (`8d6b7b2`)

### Sub-agent fan-out (Pattern 23)

| Agent | Scope | Files | Duration |
|-------|-------|-------|----------|
| A — Backend classifier | classify-file endpoint, classifier prompt, types | 3 files | ~1.3 min |
| B — Frontend classify + auto-fire | page.tsx rewrite, ClassifierConfirm component | 2 files | ~5 min |
| Integration | route.ts classifier_decision logging | 1 file (manual) | — |

### Three classification paths

| File type | Method | AI call? | Confidence | Latency |
|-----------|--------|----------|------------|---------|
| PDF/DOCX | Extension shortcut | No | 0.95 fixed | ~0ms |
| Image (LinkedIn/scam) | Haiku vision (`callClaudeVision`, tier: classify) | Yes | 0.6-0.9 | ~1-3s |
| Other | Immediate fallback | No | 0.3 fixed | ~0ms |

### Client flow

1. File dropped or pasted → `handleFileContent` reads file
2. `runClassifier()` called → shows "Classifying..." status
3. `POST /api/ask/classify-file` returns `ClassifierResult`
4. Confidence >= 0.85:
   - Pill matches or none selected → auto-fire immediately
   - Pill differs → show `ClassifierConfirm` inline ("Switch to Sales?")
     - "Yes, switch" → switch pill + auto-fire
     - "Stay" → clear classifier, keep file attached, operator proceeds manually
5. Confidence < 0.85:
   - Pre-fill input with suggested prompt, set pill, operator types/edits + hits send

### Instrumentation

Three nullable columns on `chat_logs`:
- `classifier_decision` JSONB — full classifier response
- `operator_override` BOOLEAN — true if operator stayed on current pill
- `final_destination` TEXT — pill actually used

Evidence query: `SELECT classifier_decision, final_destination FROM chat_logs WHERE classifier_decision IS NOT NULL LIMIT 5`

### Tool-reuse audit

| Primitive | Reused? | Notes |
|-----------|---------|-------|
| `@milo/ai-client` callClaudeVision | Yes | tier: 'classify' (Haiku) |
| `@milo/ai-client` TIER | Yes | TIER.classify for model name |
| Vet Part 1.5 cards | Yes | Contract drops still render IssueListExpandable |
| Existing pill switching | Yes | `setSelectedPill()` from classifier |
| Existing /api/ask streaming | Yes | Classifier routes into same pipeline |
| **New primitives created** | **Zero** | |

## Files changed

| File | Action | Lines |
|------|--------|-------|
| `SCHEMA.md` | Modified | +37 |
| `supabase/migrations/20260424700001_classifier_columns.sql` | Created | +4 |
| `src/app/api/ask/classify-file/route.ts` | Created | +91 |
| `src/lib/ask/classifier-prompt.ts` | Created | +29 |
| `src/lib/ask/classifier-types.ts` | Created | +15 |
| `src/components/ask/ClassifierConfirm.tsx` | Created | +102 |
| `src/app/ask/page.tsx` | Modified | +128/-38 |
| `src/app/api/ask/route.ts` | Modified | +4/-2 |

## Commits

- `0256963` schema: Surface 18 classifier instrumentation — 3 columns on chat_logs
- `5083954` migration: add classifier_decision, operator_override, final_destination to chat_logs
- `8d6b7b2` feat: Surface 18 v1 — auto-classify + auto-fire on file drop/paste

## Build

`npm run build` — clean, no errors, no warnings. `/api/ask/classify-file` endpoint registered.

## What's NOT in v1

- Audio file routing (defer)
- Spreadsheet routing (defer)
- Override/undo after confirmed pill switch (defer)
- Classifier accuracy training loop (defer)
- Surface 18 in /operator surfaces (defer)
- Visual verification screenshots (pending deploy)
- Token cost measurement (pending live test)

## Env var

`ASK_CLASSIFIER_AUTO_SUBMIT_THRESHOLD` — default `0.85`. Set lower to auto-fire more aggressively, higher to require more manual confirmation.
