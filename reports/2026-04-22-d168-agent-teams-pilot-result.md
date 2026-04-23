# D168 — Agent Teams Pilot Result

**Coder:** 4 (Ops)
**Directive:** D168
**Date:** 2026-04-22
**Status:** COMPLETE

---

## Summary

Agent Teams pilot ran two teammates ("verifier" and "auditor") in parallel against the D155 SyncFooter deployment. Both teammates launched, executed independently, and returned results without Mark relaying a single message. Wall time was ~2 minutes for both (parallel). The coordination model works. Two production findings surfaced: the footer spec lacks assertions (silent auth failure) and `/api/health-check` is an unauthenticated 1:7 amplification vector.

## Pass/Fail Criteria (from D166 spec)

| # | Criterion | Result | Notes |
|---|-----------|--------|-------|
| 1 | Teammate `cd`s to milo-for-ppc and operates | **PASS** | Verifier ran Playwright in ~/Documents/GitHub/milo-for-ppc without path issues |
| 2 | Teammate operates without Mark relaying messages | **PASS** | Both teammates received prompts at spawn, returned results at completion |
| 3 | Task dependency respected | **N/A** | Pilot used parallel (no-dependency) tasks; sequential dependency not tested |
| 4 | Screenshots appear on ~/Desktop/ | **PASS** | All 6 files landed (3 states × 2 viewports) |
| 5 | Mark did not relay a single message | **PASS** | Zero intervention required |
| 6 | Total wall time ≤ 45 minutes | **PASS** | ~2 min wall time (verifier 94s, auditor 123s, parallel) |

**Fail criteria checked:**

| Fail trigger | Observed? |
|---|---|
| Teammate cannot cd to cross-repo path | No |
| Teammate ignores CLAUDE.md rules | No |
| Lead crashes and takes teammates down | No |
| Mark intervenes more than once | No (zero interventions) |

**Overall: 5/5 testable criteria PASS. 0 fail triggers hit.**

## Finding 1: Footer Spec Missing Assertions

The Playwright spec at `tests/d155-footer.spec.ts` has no `expect()` calls. It captures screenshots but never asserts page content. In this pilot, all 6 screenshots showed the **MILO login page** — the session token at `/tmp/d146-session.json` was expired or rejected — but the spec reported 3/3 PASSED.

**Impact:** Silent false-pass. Visual verification requires Mark to manually open each screenshot. A login-wall guard (`expect(page.locator('.sync-footer')).toBeVisible()`) would catch this in CI.

**Action:** Add assertion before screenshot capture. Refresh session token and re-run.

## Finding 2: Health-Check Bandwidth Amplification

`/api/health-check` at `tlp.justmilo.app` is **publicly accessible without authentication**. Each call fans out to 7 external services.

| Metric | Value |
|---|---|
| Payload size | ~640 bytes |
| Response time (median) | 1.0s |
| Response time (worst) | 9.3s (ConvoQC Enrichment slow) |
| Amplification factor | 1:7 per request |
| Steady state (50 operators) | 600 inbound → 4,200 external calls/hour |
| Attack scenario (100 req/s) | **2.52M external calls/hour** |

**Additional finding:** `DEDUP_MS` (4 min) < `POLL_MS` (5 min), so the dedup window never fires during normal polling. It is dead code at steady state.

**Recommendations (priority order):**
1. Auth-gate the endpoint (operator session required)
2. Rate-limit: 12 req/min per IP minimum
3. Server-side response cache: 60–120s TTL
4. Per-service timeout cap: 3–5s (prevents connection pileup)
5. Raise DEDUP_MS ≥ POLL_MS or remove it

## Lessons Learned

1. **Parallel teammates are fast.** Two independent tasks completed in 2 minutes wall time vs. sequential Setup C which would require two separate pane dispatches and manual result collection.
2. **No message relay is the real win.** Setup C requires Mark to copy-paste results between panes. Agent Teams eliminates this entirely — the lead synthesizes both results.
3. **Sequential dependency not yet tested.** The D166 spec's criterion 3 (verifier waits for implementer) was not exercised. A future pilot should test a dependent chain.
4. **Teammate scope is clean.** Each teammate stayed in its assigned repo and task. No cross-contamination.
5. **Spec quality matters more with automation.** A human would notice login-page screenshots immediately. An automated pipeline with no assertions silently passes. Agent Teams amplifies both good and bad test design.

## Recommendation: Cutover vs. Setup C

**Recommend: Cutover to Agent Teams as primary, retain Setup C as fallback.**

Agent Teams proved faster (2 min vs. ~15 min Setup C), required zero intervention, and handled cross-repo access cleanly. The parallel execution model maps naturally to the 4-coder lane structure.

Setup C should remain as fallback for:
- Sessions requiring 4+ simultaneous coders (Agent Teams spawning limit unclear)
- Long-running tasks where teammate context window may compress
- Any session where Agent Teams exhibits instability

The cutover is low-risk because Setup C requires zero teardown — `tmux-coders.sh` is a script, not infrastructure.

## Files

| File | Action |
|---|---|
| `reports/2026-04-22-d168-agent-teams-pilot-result.md` | Created (this report) |

---

Report at: https://raw.githubusercontent.com/moscaleflow/conductor-mark-public/main/reports/2026-04-22-d168-agent-teams-pilot-result.md
