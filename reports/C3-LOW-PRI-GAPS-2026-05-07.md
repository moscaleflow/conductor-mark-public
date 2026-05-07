# Low-Priority Gap Build Specs

Coder-3 Research | 2026-05-07

---

## Gap A: Alert action_data.path UI Navigation

**Status: ALREADY IMPLEMENTED in TriageDrawer. Missing from BriefingPanel.**

### Current code locations

- **TriageDrawer** (`src/components/ask/TriageDrawer.tsx`, lines 190-206): `getEntityNavigation()` already reads `action_data.path` as Priority 1, falls back to entity page. Navigation button rendered at lines 729-759 using `onNavigate` -> `router.push()`.
- **Alert generation** (`src/app/api/alerts/generate/route.ts`, lines 971, 1002): Only 2 alert types set `path` today -- both use `path: '/invoices'` for billing alerts.
- **BriefingPanel** (`src/components/BriefingPanel.tsx`, lines 1092-1230): `handleAction()` does NOT read `action_data.path`. The `NeedsYouItem` type (line 40) has `action_label` but no `action_data` field. BriefingPanel renders `action_label` as a button but navigation is hard-coded by action type (bid review, snooze, dismiss, open_external).
- **MorningBriefing** (`src/components/operator/MorningBriefing.tsx`): Shows `action_label` chip but no click handler for navigation.

### Build spec

1. **BriefingPanel**: Add `action_data?: Record<string, unknown>` to `NeedsYouItem` interface (line ~40). In `handleAction()`, add an early check: if `item.action_data?.path` is a string starting with `/`, call `router.push(path)` and return. This needs `useRouter` import (not currently imported in BriefingPanel).
2. **MorningBriefing**: Add the same `action_data` field to the briefing item type (line ~38). If `action_data.path` exists, make the action chip clickable with `router.push()`.
3. **Briefing API** (`/api/briefing/morning/route.ts`): Ensure `action_data` is passed through when building briefing items from alerts (currently only `action_label` is forwarded).
4. **No TriageDrawer changes needed** -- it already works correctly.

### Estimated complexity: Small

Files to change: 3 (BriefingPanel.tsx, MorningBriefing.tsx, briefing/morning/route.ts). TriageDrawer is done.

---

## Gap B: AI suggestedAction Silent Failure on Timeout

**Status: Partially handled. Timeout exists but fallback is incomplete.**

### Current code locations

- **TriageDrawer** (`src/components/ask/TriageDrawer.tsx`, lines 276-312): The `useEffect` that fires the review call uses `AbortController` with a 10-second timeout (`setTimeout(() => controller.abort(), 10000)`). On abort or error, the `.catch(() => {})` at line 300-302 silently swallows the failure. Comment says "Fail silently -- static data stays visible as fallback."
- **suggestedAction rendering** (lines 1161-1194): Only renders when `reviewData?.suggestedAction` exists. If the AI call times out, `reviewData` stays `null`, so the suggestedAction button never appears.
- **Server-side fallback** (`src/app/api/operator/pill/review/route.ts`, lines 1331-1348): The API has a full fallback path that returns structured data even when Claude fails (`buildFallbackSections`). The fallback does NOT include a `suggestedAction` field.

### Build spec

1. **TriageDrawer** (lines 300-302): Replace the silent `.catch(() => {})` with:
   ```
   .catch(() => {
     setReviewData({
       whatISee: alert.detail ?? alert.title,
       impact: alert.revenue_at_risk ? `$${alert.revenue_at_risk.toLocaleString()} at risk` : '',
       myTake: 'AI analysis timed out. Review the alert details above.',
     });
   })
   ```
   This ensures the three sections always populate, even on timeout.

2. **Server-side fallback** (`pill/review/route.ts`, `buildFallbackSections` at line 1353): Add a `suggestedAction` field to the fallback return. Derive from alert type: `bid_floor_violation` -> `{ label: 'Review bids', actionType: 'draft_outreach' }`, `invoice_overdue` -> `{ label: 'Draft follow-up', actionType: 'draft_outreach' }`, default -> `null`.

3. **No new UI states needed.** The shimmer skeleton (lines 153-166) already displays during loading. The fix ensures content always replaces the shimmer.

### Estimated complexity: Small

Files to change: 2 (TriageDrawer.tsx catch block, pill/review/route.ts fallback builder).

---

## Gap C: IP to user_id Session Binding

**Status: Two separate chat systems, both IP-bound. Auth infrastructure exists but is unused for chat identity.**

### Current code locations

- **Ask route** (`src/app/api/ask/route.ts`, lines 74-76, 341-344): `hashIP(ip)` creates a SHA-256 hash of the IP. This `ipHash` is used for rate limiting (line 365, `ask_rate_limits` table) and stored in `chat_logs.ip_hash` (lines 534, 1147, 1319, 1528, 1889, 1962).
- **Ask history** (`src/app/api/ask/history/route.ts`, lines 99-103, 114): Retrieves history by matching `chat_logs.ip_hash`. Three-day rolling window.
- **Ask suggestions** (`src/app/api/ask/suggestions/route.ts`, lines 59-62, 79): Same IP hash for recent query lookup.
- **Milo route** (`src/app/api/milo/route.ts`, lines 80-125): Uses hardcoded `operator: 'default'` in `milo_conversations` table. Keyed by `(operator, date)`, not by user.
- **Milo history** (`src/app/api/milo/history/route.ts`, line 22): Queries by `operator: 'default'` -- all users share one conversation stream.
- **Auth server** (`src/lib/auth-server.ts`): `getEffectiveUser(req)` returns `effectiveUserId` from session cookie. Already imported in milo route (line 8) and used (line 145) but only for impersonation metadata, NOT for conversation keying.
- **Demo sessions** (`src/lib/demo/session.ts`): Cookie-based `milo_demo_session` for demo mode only.

### Build spec

**Phase 1: Ask system (chat_logs)**

1. In `/api/ask/route.ts`: After extracting IP, call `getEffectiveUser(req)` to get `effectiveUserId`. Use `effectiveUserId` as the primary key for `chat_logs` storage (add `user_id` column or use it in place of `ip_hash` when available). Keep `ip_hash` as fallback for anonymous/unauthenticated requests.
2. In `/api/ask/history/route.ts`: Query by `user_id` when auth cookie is present, fall back to `ip_hash` when not. This preserves anonymous browsing.
3. In `/api/ask/suggestions/route.ts`: Same pattern -- prefer `user_id`, fall back to `ip_hash`.
4. **Database**: Add `user_id` column (nullable UUID, FK to auth.users) to `chat_logs` table. Add index on `(user_id, created_at)`. Backfill is not required -- new messages will use both `user_id` and `ip_hash`.
5. **Rate limiting**: Keep IP-based rate limiting as a security measure (prevents abuse from shared accounts). Add a separate `user_id` column to `ask_rate_limits` for user-scoped limits.

**Phase 2: Milo system (milo_conversations)**

1. In `/api/milo/route.ts`: Replace `operator: 'default'` with `operator: effectiveUserId ?? 'anonymous'`. The `persistConversation` function (line 80) needs `effectiveUserId` passed in.
2. In `/api/milo/history/route.ts`: Query by the effective user's ID instead of `'default'`.

**Migration note**: Existing `operator: 'default'` rows become orphaned but are harmless. New conversations start clean per-user.

### Estimated complexity: Medium

Files to change: 5 (ask/route.ts, ask/history/route.ts, ask/suggestions/route.ts, milo/route.ts, milo/history/route.ts) + 1 DB migration (add user_id column to chat_logs).

---

## Gap D: Search Result Aggregate Indicators

**Status: No dedicated search results component exists. "Search" routes queries to Milo chat.**

### Current code locations

- **Operator SearchBar** (`src/components/operator/SearchBar.tsx`): A text input that calls `onSubmit(query)`. The submit handler (operator/page.tsx line 372-374) routes to `openChatWith(query)` -- it opens the Milo chat panel, not a search results page.
- **Topbar search** (`src/components/Topbar.tsx`, line 1211): `onSearchClick` opens the chat panel via `LayoutShell.tsx` line 124.
- **Pipeline page** (`src/app/pipeline/page.tsx`, lines 522, 1046-1062): Has its own local Fuse.js search over pipeline entities. Results show entity cards with stage, source, verticals. No alert counts or onboarding status.
- **Partners page** (`src/app/partners/page.tsx`, lines 169, 240-251): Local filter search over partner rows. No aggregate indicators.
- **Entity detail** (`src/app/api/entities/[name]/route.ts`): Returns entity profile but is not used in search results.
- **Entity status** (`src/app/api/entity/status/route.ts`): Returns current status data for a single entity.
- **Available data sources for aggregates**:
  - Alert counts: `alerts` table, filterable by `entity_name` and `status='active'` (already used in `getEntityAlerts()` in milo-queries.ts)
  - Onboarding status: `onboarding_runs` table + `getOnboardingStatusByCounterparty()` in onboarding-client.ts
  - Pipeline stage: `prospect_pipeline` table, `getPipelineStage()` in milo-queries.ts
  - Next-step: `use-entity-next-step.ts` hook

### Build spec

Since there is no standalone search results UI (search goes to Milo chat), this gap requires building a new search surface or augmenting existing pages.

**Option A: Augment pipeline page search results** (Recommended -- smaller scope)

1. **Pipeline page** (`src/app/pipeline/page.tsx`): After Fuse.js filters results, batch-fetch aggregate data for visible entities. Add a server endpoint:
   - `GET /api/entities/aggregates?names=Entity1,Entity2,...` -- returns `{ [name]: { active_alerts: number, onboarding_status: string | null, pipeline_stage: string } }`.
2. **Pipeline entity card**: Add a small indicator row below the entity name: `"3 alerts"` badge (red if critical), `"Onboarding: qualifying"` tag, next-step indicator.
3. **New API route** (`/api/entities/aggregates/route.ts`): Accepts comma-separated entity names (max 20). Runs parallel queries against `alerts` (count where status='active'), `onboarding_runs` (latest status), `prospect_pipeline` (stage). Returns map.

**Option B: Build a dedicated search overlay** (Larger scope, deferred)

A Cmd+K style overlay that searches across CRM, pipeline, and alerts simultaneously. Each result row shows entity name + type + aggregate badges. This is a significant new feature -- recommend deferring.

### Estimated complexity: Medium (Option A) / Large (Option B)

Files to change (Option A): 2 new (aggregates API route), 1 modified (pipeline/page.tsx entity cards).

---

## Self-Attestation

- **Gap A**: Confirmed TriageDrawer already implements action_data.path navigation. BriefingPanel and MorningBriefing do not. Spec covers both.
- **Gap B**: Confirmed the 10s timeout + silent catch pattern. Server fallback exists but lacks suggestedAction. Spec covers both client and server.
- **Gap C**: Confirmed both chat systems (ask + milo) use IP-only identification. Auth infrastructure (`getEffectiveUser`) exists and is already imported in the milo route. Spec uses it.
- **Gap D**: Confirmed no dedicated search results UI exists -- search routes to Milo chat. Pipeline page has local Fuse.js search without aggregates. Spec recommends augmenting pipeline page (smaller scope).
- **No UI surfaces touched** -- this is research/spec only. Screenshot N/A.
- **No known gaps** -- all 4 items fully characterized.

Report at: https://raw.githubusercontent.com/moscaleflow/conductor-mark-public/main/reports/C3-LOW-PRI-GAPS-2026-05-07.md
