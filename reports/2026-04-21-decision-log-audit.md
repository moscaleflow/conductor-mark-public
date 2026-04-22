# DECISION-LOG Numbering Integrity Audit — 2026-04-21

> Read-only audit. No DECISION-LOG changes. Recommendations only.

---

## Current state

81 decisions logged. File-order sequence:

```
1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19 20 21 22 23 24 25 26 27 28 29 30
40 31 32 33 34 35 36 37 38 39 41 42 43 44 45 46 47 48 49 50 51 52 53 54 55 56 57
58 59 60 61 62 63 64 65 66 67 68 69 71 72 73 77 78 79 80 81
```

---

## Anomaly catalog

### A1. OUT-OF-ORDER — D40 appears between D30 and D31

**Numbers:** 30, **40**, 31
**Lines:** D30 at L436, D40 at L446, D31 at L469

**Context:**
- D30: "Anthropic-only model policy enforced across all consumers" (original)
- D40: "@milo/crm schema — best-of-both design" (was originally D30, collided, renumbered to 40)
- D31: "Counterparty status includes 'blacklisted'"

**How it happened:** D40 was originally written as D30 by Coder-4 in `69ff886` (2026-04-19). Collided with the existing D30 (Anthropic policy). Coder-4 renumbered to D40 in `9ad1832` (2026-04-20) but left it in the same file position between D30 and D31 rather than moving it after D39.

**Recommendation:** **Move D40 after D39** (line 563). This is a file-order fix only — no renumbering needed. D40's content is logically contemporaneous with D31-D39 (all are 2026-04-19 schema/migration decisions). Moving it preserves numerical sort order without changing any decision number or cross-reference.

---

### A2. GAP — D70 missing

**Numbers:** 69, ~~70~~, 71
**Lines:** D69 at L957, D71 at L972

**Context:**
- D69: "Milo v1 Funnel Vision locked" (Coder-3 D61, commit d363507)
- D71: "v1 funnel pivot — cinematic Milo-led hero" (Coder-3 D64, commit 38f4b19)

**How it happened:** D70 was logged by Coder-1 for "@milo/feedback v0.1.0 shipped" — referenced in SESSION-HANDOFF-2026-04-21.md and MILESTONE-LOG.md. But the D70 entry never landed in conductor-mark's DECISION-LOG.md. It was likely logged in milo-engine's local decision log and never synced to conductor-mark by Coder-4 ops.

**Evidence:** SESSION-HANDOFF line 44: "61 | 1 | @milo/feedback primitive build | COMPLETE | b2534af/f37a87c/ff1f03d, Decision 70." MILESTONE-LOG: "@milo/feedback v0.1.0 primitive shipped (Decision 70, 17 tests)."

**Recommendation:** **Backfill D70** with a stub entry sourced from SESSION-HANDOFF and milo-engine commits. Content: "@milo/feedback v0.1.0 implemented — polymorphic feedback collection, 17 tests, commits b2534af/f37a87c/ff1f03d." Coder-4 ops work.

**Resolution (D95):** D70 backfilled by Coder-3 between D69 and D71. Evidence sourced from SESSION-HANDOFF directive ledger + CURRENT-STATE primitive table.

---

### A3. GAP — D74, D75, D76 missing

**Numbers:** 73, ~~74~~, ~~75~~, ~~76~~, 77
**Lines:** D73 at L1004, D77 at L1023

**Context:**
- D73: "Phase 2 start gate — wait for clean hero state" (Strategy-Claude via Coder-4, commit b1d87ee)
- D77: "Demo tenant TTL — 72 hours" (Coder-2 D88, commit c29b730)

**How it happened:** Coder-3 D87 originally planned to use D74-D76 for the MOP redirect auto-lock decisions. But between commit 1 (spec amendments) and commit 2 (decision locks), D77 landed from Coder-2. Coder-3 correctly jumped to D78-D80 to avoid collision (guard #9). D74-D76 were never claimed by anyone else.

**Recommendation:** **Leave as-is.** Gaps are harmless. Decision numbers are identifiers, not indices. Backfilling would require renumbering D77+ which creates cross-reference churn across reports, SESSION-HANDOFF, and MILESTONE-LOG for zero functional benefit. The gap documents the collision-avoidance that happened — that's actually useful audit trail.

---

### A4. HISTORICAL COLLISIONS — D30/D40, D42/D43/D44/D45 (RESOLVED)

**Numbers:** D30 (2x), D40 (2x), D42 (2x), D43 (2x), D44 (2x)

**Context:** All documented in-file with `**Renumbered:**` notes:
- D30 collision: CRM schema vs Anthropic policy. CRM schema → D40. Commit `9ad1832`.
- D40 collision: CRM schema (newly renumbered) vs contract-analysis v0.1.0. Analysis → D42. Commit `31e913f`.
- D42 collision: contract-signing v0.1.0 → D43. Commit `bf8c78d`.
- D43 collision: contract-negotiation schema → D44. Commit `bf8c78d`.
- D44/D45 collision: MOP writes unfrozen → D45. Commit `bf8c78d`.

**How it happened:** Multi-repo decision logging (milo-engine + conductor-mark) without a centralized numbering lock. Coder-4 ops performed bulk sync in `69ff886` and `31e913f`, resolving collisions via renumbering.

**Recommendation:** **No action.** All collisions are resolved. Renumbering notes are preserved as audit trail. Drift guard #9 was written specifically to prevent this class of error going forward.

---

### A5. HISTORICAL COLLISION — D73/D81 (RESOLVED)

**Numbers:** D73 (2x before fix)

**Context:**
- D73 original: "Phase 2 start gate" (Strategy-Claude, commit b1d87ee)
- D73 duplicate: "@milo/feedback context_id widened to TEXT" (Coder-1, originally claimed as D73)

**How it happened:** Coder-1 Directive #81 fixed a feedback API 500 error and wrote a decision entry claiming D73 without reading conductor-mark's DECISION-LOG first (guard #9 violation). D73 was already claimed for "Phase 2 start gate" (b1d87ee). Coder-1 Directive #90 corrected this by adding a canonical D81 entry in commit `7e1fe6a`. The commit was authored by Coder-1 (via Mark's git identity, `Mark <mo@scaleflow.co>`), not by Strategy-Claude or Coder-3. Commit message: "fix(numbering): renumber feedback schema decision D73 → D81 (collision with Phase 2 gate)."

**Correction (D95):** D92 audit originally described this as "Coder-1 D90 self-corrected." Verified: `7e1fe6a` is indeed Coder-1 D90 work — the commit message explicitly says "Coder-1 D90." However, Strategy-Claude killed D90 mid-session before paste-send. The commit landed regardless, meaning Coder-1 executed the fix before the kill signal arrived. The fix is valid and correctly resolves the collision.

**Recommendation:** **No action.** Resolved. D81 entry is correctly placed at end of file (after D80).

---

## Summary

| # | Type | Numbers | Severity | Recommendation |
|---|------|---------|----------|----------------|
| A1 | Out-of-order | D40 between D30/D31 | Low | Move D40 after D39 in file |
| A2 | Gap | D70 missing | Medium | **RESOLVED (D95)** — backfilled between D69/D71 |
| A3 | Gap | D74-D76 missing | None | Leave as-is (collision-avoidance artifact) |
| A4 | Historical collision | D30/40/42/43/44 | None | Already resolved, audit trail preserved |
| A5 | Historical collision | D73 duplicate | None | Already resolved by Coder-1 D90 |

**Actionable items:** 1 remaining (A1 move). A2 backfill completed by Coder-3 D95. A1 is Coder-4 ops work.

**Next claimable decision number:** D82.

**Cross-reference integrity:** All decision numbers referenced in SESSION-HANDOFF, MILESTONE-LOG, MASTER-PROMPT, and reports/ match their DECISION-LOG entries except D70 (referenced but missing from log — A2 above).

---

## Git authorship for each anomaly

| Anomaly | Commit | Author | Directive |
|---------|--------|--------|-----------|
| A1 (D40 placement) | 9ad1832 | Coder-4 Ops | ops sync |
| A2 (D70 gap) | n/a — entry never written to conductor-mark | Coder-1 Extraction | D61 |
| A3 (D74-76 gap) | 420dd35 (D78-80 claimed instead) | Coder-3 Research | D87 |
| A4 (D30/40 collision) | 69ff886 → 9ad1832 | Coder-4 Ops | ops sync |
| A4 (D42-45 cascade) | 31e913f → bf8c78d | Coder-4 Ops | ops sync |
| A5 (D73 dup) | b1d87ee (original) + 7e1fe6a (fix) | Strategy-Claude + Coder-1 | D73 gate + D90 fix |

---

## Authorship Trace (D107 follow-up)

> D92 audit flagged D77 as "added by another Coder since D73" without identifying the author.
> This trace resolves the mystery and documents provenance for every decision in the D69–D81 window.
> Source: `git log --format` on conductor-mark DECISION-LOG.md. All commits use Mark's git identity (`Mark <mo@scaleflow.co>`) — Coder attribution comes from commit message text.

| Decision | SHA | Date | Coder | Directive | Notes |
|----------|-----|------|-------|-----------|-------|
| D69 | d363507 | 2026-04-20 17:15 | Coder-3 Research | D61 | — |
| D70 | 73bdc8b | 2026-04-21 14:30 | Coder-3 Research | D95 | Backfilled. Originally Coder-1 work (D61), never synced to conductor-mark. |
| D71 | 38f4b19 | 2026-04-20 19:43 | Coder-3 Research | D64 | — |
| D72 | b4af4ce | 2026-04-21 12:15 | Coder-3 Research | D66 | Bundled with milestone + drift guards 13/14/15. |
| D73 | b1d87ee | 2026-04-21 14:12 | Coder-4 Ops | D86 | Phase 2 start gate. Verbatim text from Strategy-Claude. No Coder attribution in subject line. |
| D74 | — | — | — | — | Never claimed. Gap artifact from Coder-3 D87 collision avoidance. |
| D75 | — | — | — | — | Never claimed. Same as D74. |
| D76 | — | — | — | — | Never claimed. Same as D74. |
| **D77** | **c29b730** | **2026-04-21 14:19** | **Coder-2 Migration** | **D88** | **MYSTERY RESOLVED. Commit message: "Decision 77: Demo tenant TTL locked at 72h — spec + decision log (Coder-2 D88)".** |
| D78 | 420dd35 | 2026-04-21 14:21 | Coder-3 Research | D87 | Batch commit with D79 and D80. |
| D79 | 420dd35 | 2026-04-21 14:21 | Coder-3 Research | D87 | Same commit as D78. |
| D80 | 420dd35 | 2026-04-21 14:21 | Coder-3 Research | D87 | Same commit as D78. |
| D81 | 7e1fe6a | 2026-04-21 14:22 | Coder-1 Extraction | D90 | Renumbered from D73 collision. Kill signal arrived after commit landed. |

### D77 resolution

D77 was authored by **Coder-2 (Migration lane)** executing **Directive #88**. Commit `c29b730` landed at 2026-04-21 14:19:58, seven minutes after D73 (b1d87ee at 14:12:32) and two minutes before D78-D80 (420dd35 at 14:21:01). The commit message explicitly identifies the source: `"Decision 77: Demo tenant TTL locked at 72h — spec + decision log (Coder-2 D88)"`.

D92 audit (0119fd2) described D77 as "added by another Coder since D73" because `git log` was not consulted — only file diff between the D92 audit's known-good baseline and current state. D91 confirmed exactly one D77 entry exists, ruling out duplicate-append. This is straightforward directive execution, not drift.

### Anomalies surfaced

None beyond what D92 already cataloged. The D69–D81 window is clean:
- All decisions have identified authors via commit messages
- No orphaned entries (every decision traces to a directive)
- Chronological commit order matches logical decision order (D73 → D77 → D78-D80 → D81)
- The 7-minute gap between D73 and D77 confirms Coder-2 and Coder-4 were operating in parallel without collision
