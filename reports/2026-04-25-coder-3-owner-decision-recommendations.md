# Owner-Decision Recommendations — 10 Dead Code Audit Edge Cases

**Coder-3 Research | 2026-04-25**
**Scope:** Pure analysis. No code changes. Propose DELETE/KEEP/REWRITE per item with rationale for Mark to scan and approve. Coder-4 acts on Mark's choices.
**Source:** Dead code audit (commit 5045eb7), NEEDS-OWNER-DECISION items A-J.

---

## Decision Matrix

| Item | File(s) | LOC | Recommend | Risk if kept | Risk if deleted | Next step |
|---|---|---|---|---|---|---|
| **A** | `reviewer-agent.ts` + 2 replay scripts | 666 | **DELETE code, KEEP docs** | Zero — unused infra | Zero — no callers | Coder-4 deletes 3 files |
| **B** | `/api/captures` + `/api/captures/[id]` | 88 | **DELETE** | Documentation drift (routes documented but dead) | Zero — data accessible via 4 other routes | Coder-4 deletes 2 files + updates docs |
| **C** | `/api/live-stats` + `/api/live-stats/pulse` | 144 | **DELETE** | Stale endpoint accumulation | Zero — underlying `getLiveStats()` stays in `daily-operations.ts` | Coder-4 deletes 2 files + env var |
| **D** | `/api/accuracy` | 200 | **KEEP** | Near zero — 200 lines, read-only, self-contained | Lose smoke test CI gate + non-trivial accuracy computation | No action needed |
| **E** | `/api/demo/analyze` | 163 | **KEEP** | Near zero — active production code | **Breaks homepage hero demo** | No action (NOT dead code) |
| **F** | `scripts/build-buyer-profiles.ts` | 167 | **DELETE** | Hardcoded service_role key in git | Zero — superseded by entity_knowledge | Coder-4 deletes + output JSON |
| **G** | `scripts/build-publisher-profiles.ts` | 235 | **DELETE** | Hardcoded service_role key in git | Zero — superseded by entity_knowledge | Coder-4 deletes + output JSON |
| **H** | `scripts/update-milo-knowledge.ts` | 177 | **DELETE** | Hardcoded service_role key in git | Zero — superseded by entity_knowledge | Coder-4 deletes + output markdown |
| **I** | `tests/smoke.spec.ts` | 169 | **REWRITE** | Stale tests against dead Netlify URL | Lose 8 page + 6 API + 1 upload test coverage | Coder-2 updates BASE URL |
| **J** | `auth-server.ts` L132-174 | 43 | **REWRITE (URGENT)** | **Unauthenticated privilege escalation** | N/A — must fix, not delete | Coder-1 security fix |

---

## Per-Item Deep Dive

### Item A: reviewer-agent.ts + 2 replay scripts — DELETE code, KEEP docs

**Files:**
- `src/lib/reviewer-agent.ts` (255 lines)
- `scripts/replay-reviewer-validation.ts` (203 lines)
- `scripts/replay-reviewer-soft-launch.ts` (208 lines)
- `docs/REVIEWER_AGENT_PROMPT.md` (306 lines) — KEEP
- `docs/REVIEWER_ESCALATION_RULES.md` (98 lines) — KEEP

**Current state:** LLM-powered proxy for Mark that reviews builder agent decisions during sprint fan-outs. `reviewDecision(ctx)` loads a system prompt from docs, fetches precedent from `reviewer_decisions` table, calls Claude via @milo/ai-client (judgment tier), parses verdict, logs to DB. The replay scripts run 8 hardcoded Sprint 2 decisions through `reviewDecisionDry()` and score match rates.

**Why flagged:** Zero callers in any route, tool, or component. Only the two replay scripts import it, and those are one-shot validation artifacts from 2026-04-10.

**Three options:**

| Option | What's lost | What's gained |
|---|---|---|
| **DELETE all 5 files** | Institutional knowledge in the 2 docs (21 TLP rules, dynamic verification protocol, escalation tree) | 666 lines removed, clean codebase |
| **DELETE code + KEEP docs** | Nothing — docs have standalone value as TLP code review reference | 666 lines of dead code removed, institutional knowledge preserved |
| **KEEP everything** | Nothing — but 666 lines of dead infrastructure stay in the build | Zero gain |

**Recommended: DELETE code, KEEP docs.** The docs encode 21 non-negotiable rules, a dynamic verification protocol, and an escalation decision tree that are useful even without the agent. If the reviewer-agent concept is revived, the docs are the seed. The `reviewer_decisions` migration (`20260410120000`) can be dropped in a future cleanup pass — the table is likely empty or near-empty.

---

### Item B: /api/captures (plural) routes — DELETE

**Files:**
- `src/app/api/captures/route.ts` (47 lines)
- `src/app/api/captures/[id]/route.ts` (41 lines)

**Current state:** GET-only read endpoints for `conversation_captures`. The list route returns a summary view (strips `raw_content`) with optional `contact_id`, `source`, `days` filters. The detail route returns full capture + related `action_items`.

**Why flagged:** Zero frontend callers. Created 2026-03-18 (D21), never imported by any component or page. The data is already accessible via 4 other paths: direct Supabase REST from `pipeline/page.tsx`, `/api/entities/[name]`, `/api/operator/executive/`, `/api/action-items/`.

**Documentation discrepancy:** CLAUDE.md documents POST/PATCH/DELETE for these routes, but only GET was ever implemented.

**Three options:**

| Option | What's lost | What's gained |
|---|---|---|
| **DELETE** | A clean read API for captures (never used) | 88 lines removed, documentation mismatch resolved |
| **KEEP** | Nothing — but orphaned routes stay | Zero — no consumer benefits |
| **REWRITE** | N/A — if needed, the 4 existing access paths suffice | N/A |

**Recommended: DELETE.** Also update CLAUDE.md, SYSTEMS.md, and docs/handoff.md to remove `/api/captures` references.

---

### Item C: /api/live-stats + pulse — DELETE

**Files:**
- `src/app/api/live-stats/route.ts` (93 lines)
- `src/app/api/live-stats/pulse/route.ts` (51 lines)

**Current state:** External API endpoints for "PPCRM sub-bots." Auth via `x-api-key` header against `LIVE_STATS_API_KEY` env var. Return today's/yesterday's call/revenue/payout/margin data from `getLiveStats()`. Created 2026-03-25, never modified.

**Why flagged:** Zero callers inside or outside the codebase. Built for PPCRM sub-bots that never integrated. No PPCRM repo exists in `~/Documents/GitHub/`. PPCRM project is frozen.

**Data available elsewhere:** `/api/stats` (called by CommandCenter dashboard), `getLiveStats()` direct import in briefing assembly, synthetic tester.

**Three options:**

| Option | What's lost | What's gained |
|---|---|---|
| **DELETE** | An external API nobody calls | 144 lines removed + `LIVE_STATS_API_KEY` env var cleaned |
| **KEEP** | Nothing — 144 lines of dead endpoints stay | Zero — PPCRM is frozen, bots never materialized |
| **REWRITE** | N/A | N/A |

**Recommended: DELETE.** Remove both routes + `LIVE_STATS_API_KEY` from `.env.example`.

---

### Item D: /api/accuracy — KEEP

**File:** `src/app/api/accuracy/route.ts` (200 lines)

**Current state:** Computes a weighted accuracy score (revenue 40%, payout 30%, call count 30%) comparing Milo's `call_records` against TrackDrive reconciliation data. Returns rich payload with per-day breakdowns, discrepancy explanations. Created 2026-03-12 (D9), never modified.

**Why flagged:** Zero frontend callers. Only consumer is `tests/smoke.spec.ts` which asserts `accuracy_score >= 90`.

**But:** The computation logic is non-trivial (weighted scoring across 3 dimensions, discrepancy explanation generation). The raw `reconciliation_runs` data is queried by 3 other routes but none compute the composite score. If a health dashboard is built later, this endpoint is ready-made.

**Three options:**

| Option | What's lost | What's gained |
|---|---|---|
| **DELETE** | Non-trivial accuracy computation, smoke test CI gate | 200 lines removed |
| **KEEP** | Nothing — read-only, self-contained, no side effects | Future health dashboard ready-made |
| **REWRITE** | N/A | N/A |

**Recommended: KEEP.** Cost of keeping is near zero (200 lines, read-only). Cost of rewriting later if needed is significant (weighted scoring logic, date-range handling, discrepancy explanations).

---

### Item E: /api/demo/analyze — KEEP (NOT DEAD CODE)

**File:** `src/app/api/demo/analyze/route.ts` (163 lines)

**Current state:** Powers the public-facing hero demo on the homepage. Receives file upload (no auth — middleware explicitly whitelists `/api/demo/*`), sends to MOP for text extraction, runs analysis via @milo/ai-client, stores result under `DEMO_PPC_ID` tenant, sets demo session cookie.

**Why flagged as NEEDS-OWNER-DECISION:** Original audit called it "likely dead or low-traffic" based on a shallow grep. Deep dive revealed it is actively called by `src/app/page.tsx` line 201 (the hero dropzone upload handler) and was last modified 2026-04-21 (3 days ago, D62 @milo/feedback wireup).

**The audit was wrong on this item.** This is live production code. Deleting it breaks the homepage.

**Recommended: KEEP.** The only cleanup opportunity: `parseAnalysisJSON()` is copy-pasted between this route and `analyze-bg`. Extract to shared util (Coder-1 task, low priority).

---

### Item F: build-buyer-profiles.ts — DELETE

**File:** `scripts/build-buyer-profiles.ts` (167 lines)

**Current state:** CLI script that queries `call_records` for 30 days, computes per-buyer stats, writes `docs/buyer-profiles.json`. Hardcoded service_role key on line 14. Last modified 2026-03-12 (D13, 43 days ago).

**Why flagged:** Hardcoded credential + unclear if superseded.

**Deep dive confirmed:** The `entity_knowledge` table (migration `20260409000001`) and `knowledge-engine.ts` fully supersede this approach. The output file `docs/buyer-profiles.json` has zero runtime consumers in `src/` — only consumed by `update-milo-knowledge.ts` (also dead, Item H).

**Recommended: DELETE.** Also delete output file `docs/buyer-profiles.json` (13.8 KB).

---

### Item G: build-publisher-profiles.ts — DELETE

**File:** `scripts/build-publisher-profiles.ts` (235 lines)

**Current state:** Same pattern as Item F but for publishers. Computes quality grades (A-F), payout rates, buyer routing, risk flags. Writes `docs/publisher-profiles.json`. Hardcoded service_role key on line 18. Last modified 2026-03-12.

**Deep dive confirmed:** Same supersession as Item F. Zero runtime consumers.

**Recommended: DELETE.** Also delete output file `docs/publisher-profiles.json` (15.9 KB).

---

### Item H: update-milo-knowledge.ts — DELETE

**File:** `scripts/update-milo-knowledge.ts` (177 lines)

**Current state:** Reads buyer + publisher profile JSONs (from F and G), compresses into <4000 token markdown context block, writes `docs/milo-context-block.md`. Hardcoded service_role key. Last modified 2026-03-12.

**Deep dive confirmed:** The output file `docs/milo-context-block.md` has zero runtime consumers. Contains stale data ("Feb 8 - Mar 13 2026" date ranges). `knowledge-engine.ts` with `entity_knowledge` table replaced this entirely.

**The entire F→G→H pipeline is dead:** F builds buyer JSON, G builds publisher JSON, H reads both and writes markdown. Nobody reads the markdown.

**Recommended: DELETE all three as a batch.** Also delete:
- `docs/buyer-profiles.json`
- `docs/publisher-profiles.json`
- `docs/milo-context-block.md`

**Credential note:** All three scripts contain the same hardcoded service_role key for project `tmzsmdkfqvjjjwkukysg`. This key is in git history permanently. Rotating it is recommended but is a separate task (the project is frozen, limiting blast radius).

---

### Item I: smoke.spec.ts — REWRITE

**File:** `tests/smoke.spec.ts` (169 lines)

**Current state:** Playwright test suite hitting `https://milo-ops.netlify.app` (dead Netlify URL). Tests 8 page loads (`/`, `/campaigns`, `/invoices`, `/pipeline`, `/pings`, `/jen`, `/tiffani`, `/reports`), 6 API endpoints (`/api/health`, `/api/stats`, `/api/briefing`, `/api/accuracy`, `/api/alerts`, `/api/buyer-reports/buyers`), and 1 file upload (MediaRite CSV).

**d180-smoke.spec.ts overlap:** Near zero. d180-smoke tests only `/operator` PillBar rendering against `tlp.justmilo.app`. It does NOT cover the 8 page loads, 6 API endpoints, or upload flow that smoke.spec.ts covers.

**Three options:**

| Option | What's lost | What's gained |
|---|---|---|
| **DELETE** | 8 page + 6 API + 1 upload test coverage | 169 lines removed |
| **KEEP as-is** | Nothing gained — tests fail against dead URL | Nothing |
| **REWRITE** | Nothing — coverage preserved | Working broad smoke suite against current deploy |

**Recommended: REWRITE.** The test structure is sound and covers routes that no other test file covers. Fix:
1. Update `BASE` to `https://tlp.justmilo.app`
2. Add session injection (same pattern as d180-smoke)
3. Update test #1 ("command center loads" → hero page assertion)
4. Review route list against current app structure

**Effort: S** — URL swap + session injection + assertion updates. No new test logic needed.

---

### Item J: auth-server.ts query-param fallback — REWRITE (URGENT SECURITY FIX)

**File:** `src/lib/auth-server.ts` lines 132-174

**Current state:** `resolveRequestUser()` has a fallback: if no valid Supabase session cookie is present, it blindly trusts `?userId=X&isAdmin=true` from the query string. No shared secret, no IP restriction, no verification that the userId exists.

**This is an unauthenticated privilege escalation.** Any internet user who crafts a URL with `?userId=anything&isAdmin=true` (and omits the Supabase session cookie) can access admin-scoped data from 5 API routes:

| Route | What it exposes with admin scope |
|---|---|
| `/api/briefing` | Full business briefing — revenue, payout, pipeline, anomalies |
| `/api/operator` | Full operator page data — calls, campaigns, pipeline, alerts |
| `/api/operator/pill` | Pill data — training content, performance metrics |
| `/api/notifications/count` | Notification data for any user |
| `/api/profile/pill-preferences` | Read/write pill preferences for any user |

**Who legitimately uses this:**
1. `/api/operator/route.ts` line 72 — server-to-server fetch to `/api/briefing` passing `?userId=...&isAdmin=true`. This is a Vercel function calling another Vercel function on the same deployment. Can be replaced by a direct function import.
2. `Topbar.tsx` lines 999, 1053 — client-side fetch to `/api/notifications/count` with `?userId=...&isAdmin=...`. Redundant — the Supabase session cookie is already present on client requests.

**No cron jobs use this.** Cron routes use `CRON_SECRET` bearer tokens, not `resolveRequestUser`.

**Three options:**

| Option | What's lost | What's gained |
|---|---|---|
| **DELETE fallback** | The 2 callers break (operator-to-briefing fetch, Topbar notification count) | Privilege escalation vulnerability closed |
| **KEEP** | Nothing — vulnerability stays open | Zero |
| **REWRITE** | Nothing lost if done correctly | Vulnerability closed + legitimate callers preserved |

**Recommended: REWRITE (urgent).**
1. Remove the query-param fallback from `resolveRequestUser()` entirely
2. Refactor `/api/operator/route.ts` line 72 to call briefing logic as a direct function import instead of HTTP round-trip
3. Update `Topbar.tsx` to stop passing `userId`/`isAdmin` as query params (session cookie handles auth)
4. If any future no-cookie use case arises, add `x-internal-secret: process.env.INTERNAL_API_SECRET` header check

**Effort: S** — remove ~15 lines from auth-server, refactor 1 fetch to function call, remove 2 query-param constructions from Topbar.

**Priority: BEFORE any other cleanup work.** This is a live production security issue.

---

## Recommended Execution Order

| Priority | Item | Action | Owner | Effort |
|---|---|---|---|---|
| **P0** | J | REWRITE — close auth vulnerability | Coder-1 | S |
| **P1** | F+G+H | DELETE — scripts + output files (6 files) | Coder-4 | S |
| **P2** | A | DELETE code (3 files), KEEP docs (2 files) | Coder-4 | S |
| **P3** | B | DELETE routes (2 files), update docs | Coder-4 | S |
| **P4** | C | DELETE routes (2 files), remove env var | Coder-4 | S |
| **P5** | I | REWRITE — update BASE URL + session injection | Coder-2 | S |
| — | D | KEEP | — | — |
| — | E | KEEP (NOT dead code) | — | — |

**Total cleanup if all approved:** 16 files deleted, ~1,477 LOC removed + 3 output files (~34 KB). 1 security fix. 1 test rewrite.

---

Cross-references: Dead code audit (5045eb7), D118 (ai-client migration), D9 (accuracy lockdown), D13 (data audit), D21 (conversation capture), D62 (@milo/feedback), Sprint 2 reviewer infrastructure (2026-04-10).
