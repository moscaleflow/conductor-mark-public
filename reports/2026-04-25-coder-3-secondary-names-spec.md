# Secondary Names UX Spec — Make "Also Mentioned" Entities Actionable

**Coder-3 Research | 2026-04-25**
**Directive:** Spec only — no code changes

---

## Current State

Secondary names are plain strings in the shape `"Name — brief context"`, returned by Claude in the `secondary_names` array of multi-entity vet responses. They render as a static list under an "Also mentioned" header in `page.tsx:1290-1307`. No persistence, no actions, no cross-referencing. They exist only in the chat session and vanish on page refresh.

The vet action API (`/api/ask/vet-action`) requires a `vetResultId` — secondary names have no vet result row, so none of the existing actions (CRM, blacklist, assign) can be called on them without first upgrading them.

---

## Question 1: Card vs Stub for Secondary Names

### Options Evaluated

| Option | Description | Pros | Cons |
|---|---|---|---|
| A. Compact stub + "Vet this" | 1-line row: name, context, classification badge, "Vet this" button | Minimal UI footprint, fast to render, no extra API calls | No signal preview — user must click to learn anything |
| B. Mini-card | Smaller EntityVetCard with quick-glance verdict from prompt extraction | Richer info at a glance | Requires Claude to do shallow research on every secondary name — multiplies token cost by 3-6x per paste |
| C. Mixed — stub default, expand on click | Stubs by default, click triggers inline expansion to mini-card | Best of both, progressive disclosure | Complex state management, two render modes per item |

### Recommendation: Option A — Compact Stub with "Vet this"

**Rationale:** The whole point of "secondary names" in the prompt is that they are NOT worth a full vet. They're mentioned in passing. Asking Claude to pre-research them defeats the purpose and balloons token cost. The user decides which ones deserve deeper investigation — the UI should make that one click away, not do it automatically.

**Stub layout:**

```
┌──────────────────────────────────────────────────────┐
│ [BADGE]  Wesley Torres — mentioned as routing contact │  [Vet this]
│ [BADGE]  Tiffani Kinkennon — TLP owner               │  [TLP Team]
│ [BADGE]  Dave Chen — unknown                          │  [Vet this]
└──────────────────────────────────────────────────────┘
```

Badge types: `TLP TEAM` (green), `IN CRM` (blue), `BLACKLISTED` (red), none for unknowns.

When "Vet this" is clicked, the stub is replaced in-place with a full EntityVetCard (loading state → card). The "TLP Team" badge has no action — it's informational.

**Effort estimate:** ~4 hours Coder-1 (new `SecondaryNameStub` component, wire up click handler, backend upgrade endpoint).

---

## Question 2: Known-Entity Recognition

### The Problem

When Milo extracts secondary names from a scam chat paste, it doesn't distinguish between:
- Tiffani Kinkennon (TLP owner — should never be flagged)
- Wesley Torres (unknown individual mentioned suspiciously — should be vettable)
- "MediaBridge Partners" (already in CRM as a buyer — should show relationship context)
- "FMedia" (blacklisted — should show red badge immediately)

### Recommendation: Server-Side Classification at Extraction Time

When the backend parses `secondary_names` from Claude's JSON response, run each name through a lightweight classification pipeline before returning to the frontend. This happens server-side in the non-streaming vet path (route.ts), adding negligible latency (3 DB queries, no LLM calls).

**Classification pipeline (per secondary name):**

```
1. Extract entity name from "Name — context" string (split on " — ")
2. Check team_roster.ts → TEAM
3. Check crm_counterparties (lookupCrmEntity) → CRM_KNOWN
4. Check blacklist (screenEntity) → BLACKLISTED
5. Default → UNKNOWN
```

**Return shape change:**

Current: `secondary_names: string[]`

Proposed: `secondary_names: Array<{ name: string; context: string; classification: 'team' | 'crm' | 'blacklisted' | 'unknown'; detail?: string }>`

Where `detail` is e.g. `"TLP owner"` for team, `"Publisher in CRM"` for CRM, `"Blacklisted: fraud"` for blacklisted.

**Team config storage:** Use the existing `src/lib/team-roster.ts` which already has 11 structured entries with `display_name` and `role`. This is the right source — it's checked into the codebase, fast to read (no DB call), and already maintained. For matching, normalize and compare against `display_name` from the roster. No new table needed.

**Note:** The EntityVetCard assign dropdown currently hardcodes 4 names (`['Tiffani', 'Fab', 'Cam', 'Vee']`) while team-roster.ts has 11. This should be fixed as part of this work — pull from team-roster.ts instead of hardcoding.

**Effort estimate:** ~2 hours Coder-1 (classification pipeline in route.ts, structured secondary_names shape, update frontend rendering).

---

## Question 3: One-Click "Vet This" Upgrade

### Recommendation: New `upgrade_vet` Action in `/api/ask/vet-action`

**Flow:**

1. User clicks "Vet this" on a secondary name stub
2. Frontend sends POST to `/api/ask/vet-action` with `{ action: 'upgrade_vet', entityName: 'Wesley Torres', originalContext: 'mentioned as routing contact in paste about National Media Solutions' }`
3. Backend calls `callClaudeStream` (no-op callback, same as current non-streaming vet path) with:
   - The vet system prompt
   - Message: `Vet ${entityName}` (optionally with original context for better research)
   - web_search tools enabled
4. Backend parses JSON, saves to `vet_results`, returns the full vet result
5. Frontend replaces the stub with a full EntityVetCard in-place

**Why a new action vs. re-using the main `/api/ask` endpoint:**
- The main endpoint creates a new chat message, new assistant message, scrolls the chat. An upgrade should happen in-place within the existing message.
- The vet-action endpoint already handles per-entity operations (CRM, blacklist, assign). Adding upgrade is consistent.
- The upgrade result should be associated with the same `chat_log_id` as the original paste.

**Loading state:** Replace the "Vet this" button with animated thinking dots (same pattern as the main vet thinking state). The stub row height stays fixed to prevent layout shift. When the full card arrives, it expands the section.

**Edge case — entity already vetted:** Before calling Claude, check `lookupExistingVet()`. If fresh, return the cached result immediately. This makes repeated "Vet this" clicks instant.

**Effort estimate:** ~3 hours Coder-1 (new action handler, frontend stub → card swap, loading state).

---

## Question 4: Profile Building Loop

### The Problem

Mark wants to "build profiles on those people" — meaning a vet of "Wesley Torres" today should accumulate with a vet of "Wesley Torres" from a different paste next week.

### Recommendation: `vet_results` Table is Sufficient — No New Table

The existing `vet_results` table already stores per-entity vet results with `entity_name_normalized`, `verdict_data` (full JSON), `last_researched_at`, and `chat_log_id`. Multiple vets of the same entity create multiple rows. The dedup check (`lookupExistingVet`) returns the most recent row.

**What's needed for profile building:**

1. **Aggregation query:** When rendering a vet card, query `vet_results` for all rows matching `entity_name_normalized` + `tenant_id`. Return count, earliest `created_at`, all linked `chat_log_id`s. This gives "Wesley flagged in 3 scam chats since 2026-03-15" for free.

2. **Verdict history:** Store each vet as a new row (current behavior). Don't overwrite — the history IS the profile. A `likely_real` verdict in March followed by a `likely_scam` verdict in April tells a story.

3. **Profile card enhancement:** Add a "History" section to EntityVetCard (collapsed by default) showing prior verdicts with dates. Only shows when `historyCount > 1`.

**Why not a separate profiles table:**
- `vet_results` already has the right shape — one row per research event, linked to chat_log for context
- A profiles table would duplicate the entity normalization logic and require sync between two tables
- The aggregation query is a simple `GROUP BY entity_name_normalized` — no denormalization needed

**Effort estimate:** ~2 hours Coder-1 (aggregation query in vet-lookup.ts, history section in EntityVetCard, wire to non-streaming vet response).

---

## Question 5: Cross-Paste Linking

### The Problem

Same entity mentioned across 3 different scam chat pastes over time. The profile should surface: "Wesley flagged in 3 scam chats since 2026-03-15."

### Recommendation: Aggregation Badge on Vet Card + Link to Chat Logs

**Implementation:**

1. **At vet time:** After saving the vet result, query `vet_results` for all rows with the same `entity_name_normalized`. Count them. If count > 1, include aggregation data in the response.

2. **Aggregation data shape:**
```typescript
{
  priorVetCount: number;
  firstSeen: string; // ISO date
  verdictHistory: Array<{ verdict: string; date: string; chatLogId?: string }>;
}
```

3. **Card rendering:** If `priorVetCount > 0`, show a badge below the verdict:
```
SEEN BEFORE — 3 prior vets since Mar 15, 2026
```
This is a red-flag amplifier: if Milo has seen this entity before across multiple pastes, that's signal.

4. **Secondary name stubs:** Also check `vet_results` for secondary names. If a secondary name has been previously vetted, show a badge:
```
[PREVIOUSLY VETTED]  Wesley Torres — last vetted 2026-03-15, verdict: likely_scam
```
This changes the stub from "unknown" to "has history" without re-running the vet.

**No new tables needed.** The `vet_results` table with `entity_name_normalized` + `chat_log_id` already contains all the data. The aggregation is a query, not a schema change.

**Effort estimate:** ~2 hours Coder-1 (aggregation query, badge rendering, secondary name history check).

---

## Implementation Sequence

| Phase | Work | Owner | Estimated Hours | Dependencies |
|---|---|---|---|---|
| 1 | Structured `secondary_names` shape + classification pipeline | Coder-1 | 2h | None — backend-only change |
| 2 | `SecondaryNameStub` component with badges | Coder-1 | 2h | Phase 1 |
| 3 | `upgrade_vet` action + stub → card swap | Coder-1 | 3h | Phase 2 |
| 4 | Cross-paste aggregation query + badges | Coder-1 | 2h | Phase 1 (can parallel with 2-3) |
| 5 | Profile history section in EntityVetCard | Coder-1 | 2h | Phase 4 |
| 6 | Fix assign dropdown to use team-roster.ts | Coder-1 | 0.5h | None — standalone |

**Total estimated: ~11.5 hours across 6 phases.**

Phases 1-3 are the core ask ("make secondary names actionable"). Phases 4-5 are the profile building loop. Phase 6 is a cleanup that should ship with Phase 2.

---

## Key Existing Primitives Used (No New Tables)

| Primitive | Package | Used For |
|---|---|---|
| `lookupCrmEntity()` | `@/lib/ask/vet-lookup` | CRM classification of secondary names |
| `screenEntity()` | `@/lib/blacklist-client` | Blacklist classification of secondary names |
| `TEAM_ROSTER` | `@/lib/team-roster` | Team member recognition |
| `saveVetResult()` | `@/lib/ask/vet-results` | Persisting upgrade-to-full-vet results |
| `lookupExistingVet()` | `@/lib/ask/vet-lookup` | Dedup on upgrade + cross-paste history |
| `/api/ask/vet-action` | API route | New `upgrade_vet` action |

---

## Summary of Recommendations

| Question | Recommendation |
|---|---|
| 1. Card vs stub | **Option A: Compact stub** with classification badge + "Vet this" button |
| 2. Known-entity recognition | **Server-side classification** at extraction time using team-roster.ts + CRM lookup + blacklist screen |
| 3. One-click upgrade | **New `upgrade_vet` action** in vet-action API, returns full card, swapped in-place |
| 4. Profile building | **No new table** — vet_results aggregation query gives history for free |
| 5. Cross-paste linking | **Aggregation badge** on vet card + prior-vet badge on secondary name stubs |
