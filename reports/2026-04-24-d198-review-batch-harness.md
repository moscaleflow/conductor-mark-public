# D198 — Synthetic Harness on /review — Batch Runner

**Date:** 2026-04-24  
**Agent:** Coder-1  
**Scope:** Schema migration, persona library vendor, /review batch UI, batch-tag API, review API

---

## Summary

Built a batch runner UI on /review that fires persona library queries through /api/ask and tags results in chat_logs for review. 180 queries across 6 personas vendored as JSON. Rate-limit-aware with pause/resume. 200-query hard cap with confirmation modal.

---

## Commits

### milo-for-ppc

| Commit | Description |
|---|---|
| `e504558` | schema: D198 batch harness — propose 5 columns on chat_logs (SCHEMA.md amendment) |
| `eec6de1` | feat: D198 synthetic harness on /review — batch runner + persona library v1 |
| `8a939d7` | test: D198 Playwright — batch harness persona load, batch fire, adversarial warning |

---

## Schema

Migration `20260424300001_batch_harness_columns.sql`:

| Column | Type | Purpose |
|---|---|---|
| batch_id | TEXT | Groups queries from a single batch run (client-side UUID) |
| persona | TEXT | Source persona (vet, sales, publisher, buyer, contracts, adversarial) |
| query_id | TEXT | Query identifier (e.g., "vet-001") |
| expected_pill | TEXT | Expected pill routing from persona library |
| run_mode | TEXT | Batch config: auto_classify, force_expected, all_pills |

Partial index: `idx_chat_logs_batch_id ON chat_logs (batch_id) WHERE batch_id IS NOT NULL`

---

## Files Created

| File | Purpose |
|---|---|
| `SCHEMA.md` | chat_logs schema documentation + batch harness amendment |
| `src/lib/review/persona-library-v1.json` | Vendored persona library (180 queries, 6 personas, 141KB) |
| `src/app/review/page.tsx` | Batch runner UI |
| `src/app/api/ask/batch-tag/route.ts` | PATCH endpoint — tags chat_logs rows with batch metadata |
| `src/app/api/review/route.ts` | GET endpoint — fetches chat_logs with batch/flag/rating filters |
| `supabase/migrations/20260424300001_batch_harness_columns.sql` | Schema migration |
| `tests/d198-review-batch-harness.spec.ts` | Playwright tests |

---

## Persona Library

| Persona | Queries | Expected Pill |
|---|---|---|
| vet | 40 | vet |
| sales | 30 | sales |
| publisher | 30 | publisher |
| buyer | 30 | buyer |
| contracts | 30 | vet (no contracts pill) |
| adversarial | 20 | varies |
| **Total** | **180** | |

---

## Architecture

```
/review page  →  /api/ask (existing, same code path)
                     ↓
              chat_logs INSERT (normal)
                     ↓
              PATCH /api/ask/batch-tag (tags batch metadata)
                     ↓
              GET /api/review (filtered results)
```

Batch queries go through the exact same /api/ask endpoint as real users. No special treatment, no rate limit bypass. Metadata is patched onto the chat_logs row after the response completes using the chatLogId from the `done` SSE event.

---

## SQL Evidence

```json
[
  {"batch_id":"ec8a7572-...","persona":"vet","query_id":"vet-005","expected_pill":"vet","run_mode":"force_expected","pill":"vet","latency_ms":24239},
  {"batch_id":"ec8a7572-...","persona":"vet","query_id":"vet-004","expected_pill":"vet","run_mode":"force_expected","pill":"vet","latency_ms":21489},
  {"batch_id":"ec8a7572-...","persona":"vet","query_id":"vet-002","expected_pill":"vet","run_mode":"force_expected","pill":"vet","latency_ms":37915},
  {"batch_id":"ec8a7572-...","persona":"vet","query_id":"vet-001","expected_pill":"vet","run_mode":"force_expected","pill":"vet","latency_ms":35609},
  {"batch_id":"ec8a7572-...","persona":"vet","query_id":"vet-003","expected_pill":"vet","run_mode":"force_expected","pill":"vet","latency_ms":30203}
]
```

All 5 rows share batch_id, persona=vet, run_mode=force_expected. Pill routing matched expected_pill in all cases.

---

## Visual Verification

Stitched screenshot: `/Users/markymark/Desktop/d198-review-batch-harness.png` (1280x3260)

| Panel | Content |
|---|---|
| d198-01 | Empty state — persona selector, pill mode, limit, disabled Run Batch (0) |
| d198-04 | Batch complete — 5/5 queries, 62.6s total, 7k tokens, 0 errors, vet per-persona card |
| d198-05 | Results view — 5 expandable rows with pill badge, latency, token count |
| d198-06 | Adversarial warning — red-bordered "Includes 5 adversarial queries (prompt injection, out-of-scope). Confirm intent." |

---

## Playwright Coverage

4/4 passing (1.3m)

| Test | Verifies |
|---|---|
| /review loads with persona selector | Page renders, 6 persona buttons, library label "180 queries" |
| Batch fire 5 vet queries | Select vet, force expected pill, fire 5, batch complete summary, view results |
| Adversarial warning in confirm modal | Red warning text appears when adversarial selected |
| Verify screenshots | 6/6 exist on Desktop |

---

## Decision

Logged as **Decision 99** in DECISION-LOG.md.
