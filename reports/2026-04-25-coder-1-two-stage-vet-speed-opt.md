# Two-Stage Vet Speed Optimization — Coder-1 Report

**Date:** 2026-04-25
**Commits:** 24d31b3 (initial), bd25503 (code-fence fix merged), d9829c3 (entity-name-only fix), e106312 (max_uses tuning), d1f23ec (parse/save separation)

---

## What Changed

Replaced the single Sonnet call for multi-entity vet with a two-stage pipeline:

1. **Stage 1 — Haiku extraction (~1-2s):** `callClaudeExtended` with `tier: 'classify'` extracts entity names and classifies them as primary (full vet) or secondary (pass-through). Skipped for clean single-entity input via regex pre-check.

2. **Per-entity cache check:** Each primary entity checked against `lookupExistingVet` + `lookupCrmEntity` in parallel. Fully cached results return in <1s.

3. **Stage 2 — Parallel Sonnet research:** Uncached entities get independent `callClaudeStream` calls with `web_search` (max_uses: 2), fired via `Promise.allSettled`. Wall clock = slowest single entity, not sum.

---

## Latency Results

| Scenario | Before (single Sonnet call) | After (two-stage parallel) | Improvement |
|---|---|---|---|
| 4 entities, all cached | 25-35s (no multi-entity cache!) | **0.8s** | ~35x faster |
| 4 entities, all fresh | 25-35s | **~23s** | ~1.3x (parallel wins) |
| 1 entity, cached | ~1s | ~1s | Same |
| 1 entity, fresh | ~10-15s | ~10-12s | Same (fast path, no Haiku) |

**Why not 10s for multi-entity fresh:** Each web_search round-trip adds ~9s to Sonnet. With `max_uses: 2`, individual entities take ~19-23s. Parallelism means wall clock = max(individual) rather than sum(individual). The old approach was sequential-within-one-call (same Sonnet instance did 5-8 searches for all entities), so the theoretical maximum was similar.

**The real win is per-entity caching.** The old approach had ZERO multi-entity caching — every paste re-researched all entities from scratch. Now, repeated entities across pastes return instantly from cache.

---

## Entity-Count Quality Verification

Tested on Mark's scam chat paste: `"Teams chat paste - Sarah Johnson (National Media Solutions)..."`

| | Sonnet Baseline | Haiku Stage 1 | Match? |
|---|---|---|---|
| **Primary: National Media Solutions** | found | found | YES |
| **Primary: QuickConnect Digital** | found | found | YES |
| **Primary: FIRMFUEL** | found | found | YES |
| **Primary: ReachMax Networks** | found | found | YES |
| **Secondary: Sarah Johnson** | found | found | YES |
| **Secondary: Dave Torres** | found | found | YES |
| **Secondary: Tom Bradley** | found | found | YES |

**Result: Exact match.** Haiku extracts the same entities as Sonnet. No entities missed. Stage 1 stays on `tier: 'classify'` (Haiku) — no need to upgrade to Sonnet.

---

## Test Results

All 5 Playwright tests passing on production:

| Test | Result | Notes |
|---|---|---|
| 1. Multi-entity scam chat | PASS (4 cards) | Cards render, no raw JSON |
| 2. Single entity | PASS (1 card) | Fast path (regex, skip Haiku) |
| 3. Cache hit | PASS | Second vet returns cached card |
| 4. Contract drop (SSE) | PASS | Content-Type: text/event-stream |
| 5. Sales pill (SSE) | PASS | Content-Type: text/event-stream |

Screenshot: `/Users/markymark/Desktop/d-vet-non-streaming-fix.png`

---

## Bugs Found and Fixed

1. **Haiku wraps JSON in code fences** despite prompt instructions. Fixed: strip ```json/``` before parsing.
2. **Sonnet ignored per-entity name** when full chat paste was included as context. Fixed: user message now sends only `"Vet this entity: <name>"` without the original paste.
3. **DB save failure dropped parsed entity.** Parse and save were in the same try/catch. Fixed: separated so a save failure still includes the parsed data in the response.

---

## Architecture

```
User paste → [regex pre-check]
  ├─ Single clean entity → skip Haiku → cache check → Sonnet research
  └─ Multi-entity paste → Haiku extraction → per-entity cache check
       ├─ Cached → instant return
       └─ Uncached → parallel Sonnet calls (Promise.allSettled)
            → Parse JSON per entity → saveVetResult → assemble response
```

New file: `src/lib/ask/prompts/vet-extraction.ts` — Haiku extraction prompt + `buildPerEntityPrompt()` for per-entity Sonnet calls.

Modified file: `src/app/api/ask/route.ts` — replaced non-streaming vet block (lines 325-509) with two-stage pipeline.

---

## Stage 1 Tier Decision

**Haiku stays.** Entity-count quality matches Sonnet exactly on the test paste. Haiku extraction runs in ~1-2s vs Sonnet extraction which would add ~5-8s. The cost savings (Haiku is ~20x cheaper per token than Sonnet) make the choice clear given equal quality on this task.
