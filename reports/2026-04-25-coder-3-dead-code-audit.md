# Dead Code Audit — milo-for-ppc Cleanup Candidates

**Coder-3 Research | 2026-04-25**
**Scope:** Pure audit. No code changes. No deletions. Identify dead code across milo-for-ppc for Coder-4 cleanup.
**Excluded:** src/app/ask/, src/app/api/ask/, src/lib/ask/, src/components/ask/, src/lib/ppc/, @milo/* vendored packages.

---

## Summary Table

| Category | Files | LOC | SAFE | MEDIUM | RISKY |
|---|---|---|---|---|---|
| Pre-fork legacy components | 4 | 2,020 | 4 | 0 | 0 |
| Unreferenced components | 4 | 1,561 | 4 | 0 | 0 |
| Orphaned utilities | 2 | 494 | 1 | 1 | 0 |
| Orphaned API routes | 3 | 140 | 2 | 1 | 0 |
| Orphaned docs | 2 | 402 | 2 | 0 | 0 |
| Dead Anthropic SDK refs | 0 | 0 | — | — | — |
| Old contract-checker code | 0 (all still live) | 0 | — | — | — |
| Stale TODOs | 0 (all 5 are active) | 0 | — | — | — |
| Scripts with hardcoded credentials | 6 | ~1,354 | 0 | 5 | 1 |
| One-time migration scripts (safe) | 4 | ~790 | 4 | 0 | 0 |
| Stale test files | 1 | 169 | 0 | 1 | 0 |
| Test artifact bloat | 68 files | ~3.1 MB | 68 | 0 | 0 |
| External API routes (unclear callers) | 3 | 344 | 0 | 0 | 3 |
| **Totals** | **97 files** | **~7,274 LOC + 3.1 MB** | **85** | **8** | **4** |

**SAFE deletion candidates: 85 files, ~5,407 LOC + 3.1 MB of screenshot artifacts.**
**MEDIUM (needs manual verification): 8 files, ~1,523 LOC.**
**RISKY (needs Mark decision): 4 files, ~344 LOC + 1 credential script.**

---

## Category 1: Pre-Fork Legacy Components (SAFE DELETE)

### 1. `src/components/OnboardingTour.tsx` — 699 lines
- **Why dead:** Zero imports anywhere. `OperatorTour.tsx` line 8 explicitly states "that file is the legacy tour, flipped via /api/walkthrough." OperatorTour is the active replacement.
- **Risk:** SAFE — OperatorTour comment confirms superseded. The `/api/walkthrough` route stays (Topbar uses its GET for username resolution).
- **Action:** DELETE

### 2. `src/components/CallDetailModal.tsx` — 522 lines
- **Why dead:** Zero imports. Was a standalone modal for viewing individual call records from milo-ops. JenView has its own inline `CallDetailList` that replaced this. Only mention: a color-token comment in NotificationModal.
- **Risk:** SAFE
- **Action:** DELETE

### 3. `src/components/CommandCenter.tsx` — 1,378 lines
- **Why dead:** Zero imports. This was the original "/" route main view. Replaced by the hero funnel page (contract analysis landing) around D45/D99. All sub-components it imported (ChatInterface, StatBar, etc.) are still used by other live views.
- **Risk:** SAFE — largest dead file in the repo.
- **Action:** DELETE

### 4. `src/app/api/calls/[id]/route.ts` — 32 lines
- **Why dead:** Only consumer was `CallDetailModal.tsx` (see #2). Transitively dead.
- **Risk:** SAFE
- **Action:** DELETE (with CallDetailModal)

---

## Category 2: Unreferenced Components (SAFE DELETE)

### 5. `src/components/contract/IssueSidebar.tsx` — 89 lines
- **Why dead:** Zero imports. Contract issue display uses `IssueListExpandable`. This was a sidebar-layout alternative never wired in.
- **Risk:** SAFE
- **Action:** DELETE

### 6. `src/components/contract/JurisdictionSelector.tsx` — 47 lines
- **Why dead:** Zero imports. Contract analysis uses a different jurisdiction flow. Built but never integrated.
- **Risk:** SAFE
- **Action:** DELETE

### 7. `src/components/contract/RoleSelector.tsx` — 47 lines
- **Why dead:** Zero imports. Role selection handled differently in the analysis flow. Standalone pill-style selector never wired in.
- **Risk:** SAFE
- **Action:** DELETE

### 8. `src/lib/operator-pills 2.ts` — 240 lines
- **Why dead:** macOS Finder duplicate of `operator-pills.ts`. Space in filename makes it un-importable. Diff confirms it's a strict subset (missing `PillWithRole`, `ROLE_LABELS`, `ALL_PILLS`, `PILL_DESCRIPTIONS` exports added to the real file).
- **Risk:** SAFE
- **Action:** DELETE

---

## Category 3: Orphaned Utilities

### 9. `src/lib/reviewer-agent.ts` — 254 lines
- **Why dead:** Zero imports from any file under `src/`. Exports `reviewDecision`, `ReviewerDecision`, `ReviewerContext`, `ReviewerVerdict` — none imported by any API route, component, or tool. Referenced only by two offline scripts: `scripts/replay-reviewer-validation.ts` and `scripts/replay-reviewer-soft-launch.ts`.
- **Risk:** MEDIUM — two replay scripts depend on it. If those scripts are still used for offline validation, deleting breaks them.
- **Action:** NEEDS-OWNER-DECISION — ask Mark whether replay-reviewer scripts are still in use. If not, delete all three.

### 10. `docs/REVIEWER_AGENT_PROMPT.md` — 305 lines + `docs/REVIEWER_ESCALATION_RULES.md` — 97 lines
- **Why dead:** Only referenced from `reviewer-agent.ts` via `fs.readFile`. If reviewer-agent is dead, these are dead.
- **Risk:** SAFE (orphaned with reviewer-agent)
- **Action:** DELETE (together with reviewer-agent if Mark confirms)

---

## Category 4: Orphaned API Routes

### 11. `src/app/api/refresh/on-login/route.ts` — 67 lines
- **Why dead:** Comment says "Called on dashboard mount" but zero references in any frontend file, middleware, AuthProvider, page, or other API route. Runs dispute detection as a warm-cache mechanism, but nothing invokes it.
- **Risk:** SAFE
- **Action:** DELETE

### 12. `src/app/api/captures/route.ts` — 47 lines + `src/app/api/captures/[id]/route.ts` — 41 lines
- **Why dead:** Zero frontend references. Note: `/api/capture` (singular, 859 lines) is actively used. The plural `/api/captures` was a read-only query endpoint never wired to any UI.
- **Risk:** MEDIUM — could theoretically be called by an external client or Postman workflow.
- **Action:** DELETE (after confirming no external consumers)

---

## Category 5: Dead Anthropic SDK References

**Result: CLEAN. Zero findings.**

Grep for `new Anthropic`, `import Anthropic`, `from '@anthropic-ai/sdk'` across all of `src/` (excluding `/ask/`) returned zero hits. The D118 migration to `@milo/ai-client` is complete for all non-ask paths. The `analyze-bg` route correctly uses `import { callClaude } from '@milo/ai-client'`.

---

## Category 6: Old Contract-Checker Era Code

**Result: All still live. Zero deletion candidates.**

| File | Status | Why kept |
|---|---|---|
| `src/lib/contract-analysis-prompt.ts` (495 lines) | LIVE | Imported by analyze-bg route. Active production prompt. |
| `src/app/api/contract/analyze-bg/route.ts` (379 lines) | LIVE | Called by contract-review page + milo-tools. Has one stale header comment ("calls Anthropic directly" — should say "@milo/ai-client"). |
| `src/app/api/contract/analyze-proxy/route.ts` (97 lines) | LIVE | Called by ConversationModal.tsx. Forwards to MOP for extraction. |
| `src/lib/review/persona-library-v1.json` (1,841 lines) | LIVE | Imported by review/page.tsx. The "v1" is a version label, not deprecation. |
| `src/app/api/demo/analyze/route.ts` (~97 lines) | MEDIUM | Demo-only path with duplicated `parseAnalysisJSON`. See MEDIUM section. |

**Stale comment fix needed:** `analyze-bg/route.ts` line 5 says "calls Anthropic directly" but line 17 shows `import { callClaude } from '@milo/ai-client'`. Comment is outdated post-D118.

---

## Category 7: Stale TODOs

**Result: All 5 TODOs found are ACTIVE. Zero stale.**

| File | TODO | Status |
|---|---|---|
| `src/lib/tools/setup-new-publisher.ts` | MOP sync endpoint | ACTIVE — MOP frozen, no cross-project endpoint built |
| `src/lib/tools/setup-new-buyer.ts` | MOP sync endpoint | ACTIVE — same as above |
| `src/lib/tools/analyze-contract.ts` | v1.1: fetch document_url, parse PDF/DOCX | ACTIVE — no document_url fetcher exists |
| `src/lib/tools/generate-redlined-version.ts` | v1.1: render redlinedText into actual DOCX | ACTIVE — no DOCX rendering library exists |
| `src/lib/tools/publish-contract-for-buyer-review.ts` | v1.1: send actual notification email | ACTIVE — no transactional email provider integration |

No shipped directives (D116, D117, D118, D120, Vet Part 1.5, Surface 18) left behind orphan TODOs in these paths.

---

## Category 8: Scripts with Hardcoded Credentials (SECURITY FLAG)

**5 scripts contain hardcoded Supabase service_role JWTs** for the frozen tmzsmdkfqvjjjwkukysg project. **1 script contains a hardcoded Supabase personal access token** (`sbp_34a8...`) granting Management API access.

### 13. `scripts/seed-demo.ts` — ~234 lines
- Hardcoded service_role JWT (line 11). Targets frozen project. Demo data already cleaned up by migration.
- **Risk:** MEDIUM
- **Action:** DELETE

### 14. `scripts/anomaly-scan.ts` — ~370 lines
- Hardcoded service_role JWT (line 15). One-time scan against old data.
- **Risk:** MEDIUM
- **Action:** ARCHIVE to `scripts/_archive/`

### 15. `scripts/build-buyer-profiles.ts` — ~190 lines
- Hardcoded service_role JWT (line 14).
- **Risk:** MEDIUM
- **Action:** NEEDS-OWNER-DECISION — still used against production data? If superseded by entity_knowledge, archive.

### 16. `scripts/build-publisher-profiles.ts` — ~240 lines
- Hardcoded service_role JWT (line 18).
- **Risk:** MEDIUM
- **Action:** NEEDS-OWNER-DECISION (same as #15)

### 17. `scripts/update-milo-knowledge.ts` — ~240 lines
- Hardcoded service_role JWT (line 15).
- **Risk:** MEDIUM
- **Action:** NEEDS-OWNER-DECISION — if entity_knowledge replaced this pipeline, archive.

### 18. `scripts/apply-migrations.ts` — 69 lines
- **Hardcoded Supabase personal access token** (`sbp_34a8...`, line 31). Grants Management API access. Higher severity than service_role JWTs.
- **Risk:** RISKY
- **Action:** DELETE — migrations were applied 2026-04-09. Token should be rotated regardless.

**Recommendation:** Rotate the `sbp_34a8...` personal access token immediately. Archive or delete all 6 scripts. If the repo ever becomes public or is forked, these credentials are exposed.

---

## Category 9: One-Time Migration Scripts (SAFE ARCHIVE)

### 19. `scripts/migrate-milo-ops-contacts.ts` — ~280 lines
- D94 one-time migration of contacts from milo-ops to shared Supabase. Source project frozen.
- **Action:** ARCHIVE to `scripts/_archive/`

### 20. `scripts/backfill-snapshot-entity-names.ts` — ~120 lines
- One-shot backfill for ~2,200 rows. Header says "The cron has since been fixed."
- **Action:** ARCHIVE

### 21. `scripts/create-kent-profile.ts` — ~250 lines
- One-time user setup. Kent's profile exists. New team members use `/welcome/[slug]` flow.
- **Action:** ARCHIVE

### 22. `scripts/e2e_mediarite.mjs` — ~140 lines
- E2E test targeting old `milo-ops.netlify.app`. Stale URL.
- **Action:** ARCHIVE

---

## Category 10: Stale Test Files

### 23. `tests/smoke.spec.ts` — 169 lines
- Hardcodes `BASE = 'https://milo-ops.netlify.app'` — project now deployed on Vercel (`tlp.justmilo.app`). All 13 tests would fail. First assertion ("command center loads" on `/`) is semantically wrong (now hero page, not CommandCenter).
- **Risk:** MEDIUM — valid test structure, wrong URL.
- **Action:** NEEDS-OWNER-DECISION — update BASE to Vercel URL and fix test #1, or delete if superseded by `d180-smoke.spec.ts`.

### 24. `tests/business-hours-test.ts` — 50 lines
- Informal verification script (not a test framework file). Tests `isBusinessHours()` which is still live. Uses `America/Los_Angeles` in verification but comment says "Denver" — possible bug.
- **Risk:** SAFE
- **Action:** KEEP-WITH-COMMENT or archive

---

## Category 11: Test Artifact Bloat

### 25. `tests/screenshots/` — 68 files, ~3.1 MB
- 46 PNGs + 22 JSON API snapshots + 1 markdown summary + 2 TXT failure logs. Point-in-time artifacts from D146, D146b, D151, D155, D175, D177. Not consumed by any code. Inflate repo with binary data.
- **Risk:** SAFE
- **Action:** DELETE entire directory (git history preserves if needed)

---

## Category 12: External API Routes (RISKY — unclear callers)

### 26. `src/app/api/live-stats/route.ts` — 93 lines + `src/app/api/live-stats/pulse/route.ts` — 51 lines
- Zero frontend references. Comments state these are "MOP-independent" endpoints for "PPCRM sub-bots" authenticated via `x-api-key` header.
- **Risk:** RISKY — cannot determine from this codebase alone whether PPCRM sub-bots still call these. PPCRM project is frozen.
- **Action:** NEEDS-OWNER-DECISION — Mark confirms whether PPCRM bots still call these.

### 27. `src/app/api/accuracy/route.ts` — 200 lines
- Zero frontend references. Standalone accuracy report endpoint for reconciliation data. Could be called via Postman or external tools.
- **Risk:** RISKY
- **Action:** NEEDS-OWNER-DECISION

### 28. `src/app/api/demo/analyze/route.ts` — ~97 lines
- Demo-only analysis path. Contains duplicated `parseAnalysisJSON` function copy-pasted from analyze-bg.
- **Risk:** MEDIUM
- **Action:** NEEDS-OWNER-DECISION — if demo flow still used for sales/onboarding, keep. Otherwise delete.

---

## Category 13: Security-Adjacent Findings

### 29. `src/lib/auth-server.ts` line 133 — legacy query-param auth fallback
- Comment: "allow legacy `?userId=&isAdmin=` query parameters as a fallback when no Supabase session cookie is present (e.g. server-to-server calls, cron jobs, or older code paths)."
- **Risk:** MEDIUM — `?isAdmin=true` query param grants admin access. If only cron jobs use it, those should migrate to service-role auth.
- **Action:** NEEDS-OWNER-DECISION — audit which callers use `?userId=&isAdmin=` query param path.

---

## Pipeline Board: NOT Dead Code

Both `/pipeline` (1,448 lines) and `/pipeline-board` (1,251 lines) are actively imported and serve distinct UX purposes:
- `/pipeline` — work list with table/board toggle, search, task management. Linked from CommandCenter sidebar (NOTE: CommandCenter itself is dead, but the pipeline page has other entry points via direct URL and other navlinks).
- `/pipeline-board` — strategic board with rich DnD, next-action AI hints, outreach compose signals. Linked from SupportPanel and dashboard-v2-types.

**Not dead code.** Shared pipeline data logic is duplicated but both pages are live. Column config extraction is a refactoring task, not a cleanup task.

---

## SAFE Deletion Summary — Ready for Coder-4

| # | File | Lines | Category |
|---|---|---|---|
| 1 | `src/components/CommandCenter.tsx` | 1,378 | Orphaned main view |
| 2 | `src/components/OnboardingTour.tsx` | 699 | Superseded by OperatorTour |
| 3 | `src/components/CallDetailModal.tsx` | 522 | Replaced by JenView inline |
| 4 | `src/components/contract/IssueSidebar.tsx` | 89 | Never wired in |
| 5 | `src/components/contract/JurisdictionSelector.tsx` | 47 | Never wired in |
| 6 | `src/components/contract/RoleSelector.tsx` | 47 | Never wired in |
| 7 | `src/lib/operator-pills 2.ts` | 240 | macOS Finder duplicate |
| 8 | `src/app/api/calls/[id]/route.ts` | 32 | Consumer is dead (#3) |
| 9 | `src/app/api/refresh/on-login/route.ts` | 67 | Zero callers |
| 10 | `docs/REVIEWER_AGENT_PROMPT.md` | 305 | Orphaned with reviewer-agent |
| 11 | `docs/REVIEWER_ESCALATION_RULES.md` | 97 | Orphaned with reviewer-agent |
| 12 | `scripts/seed-demo.ts` | ~234 | Done + hardcoded credential |
| 13 | `scripts/apply-migrations.ts` | 69 | Done + hardcoded PAT |
| 14 | `scripts/migrate-milo-ops-contacts.ts` | ~280 | One-time done → archive |
| 15 | `scripts/backfill-snapshot-entity-names.ts` | ~120 | One-time done → archive |
| 16 | `scripts/create-kent-profile.ts` | ~250 | One-time done → archive |
| 17 | `scripts/e2e_mediarite.mjs` | ~140 | Stale URL → archive |
| 18 | `scripts/anomaly-scan.ts` | ~370 | Done + credential → archive |
| 19 | `tests/screenshots/` (68 files) | ~3.1 MB | Binary artifact bloat |
| — | **TOTAL SAFE** | **~4,986 LOC + 3.1 MB** | |

**Recommended Coder-4 cleanup sequence:**
1. Delete SAFE files #1-11 (src/ dead code) — single commit, run `npm run build` to verify
2. Delete/archive SAFE scripts #12-18 — separate commit (different repo area)
3. Delete tests/screenshots/ — separate commit
4. Rotate `sbp_34a8...` personal access token (credential hygiene)

---

## NEEDS-OWNER-DECISION Summary

| # | File | Lines | Question for Mark |
|---|---|---|---|
| A | `src/lib/reviewer-agent.ts` + 2 replay scripts | ~254 | Are replay-reviewer scripts still in use? |
| B | `src/app/api/captures/route.ts` + `[id]/route.ts` | 88 | Any external consumers of `/api/captures`? |
| C | `src/app/api/live-stats/route.ts` + `pulse/route.ts` | 144 | Do PPCRM sub-bots still call these? |
| D | `src/app/api/accuracy/route.ts` | 200 | Used in manual audits via Postman? |
| E | `src/app/api/demo/analyze/route.ts` | ~97 | Is the demo flow still used for sales? |
| F | `scripts/build-buyer-profiles.ts` | ~190 | Superseded by entity_knowledge? |
| G | `scripts/build-publisher-profiles.ts` | ~240 | Superseded by entity_knowledge? |
| H | `scripts/update-milo-knowledge.ts` | ~240 | Superseded by entity_knowledge? |
| I | `tests/smoke.spec.ts` | 169 | Update to Vercel URL or delete (d180-smoke supersedes)? |
| J | `src/lib/auth-server.ts` line 133 | N/A | Who uses `?userId=&isAdmin=` query-param auth fallback? |

---

Cross-references: D118 (ai-client migration — confirmed complete), D45/D99 (hero page replacing CommandCenter), D94 (contact migration), D146/D155/D175/D177 (screenshot artifact directives).
