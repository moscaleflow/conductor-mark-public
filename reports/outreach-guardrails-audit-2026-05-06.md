# Outreach Guardrails & Sales Method Deep Audit

**Repo:** `milo-for-ppc`
**Date:** 2026-05-06
**Author:** Coder-3 (Research)
**Scope:** Read-only audit of all outreach drafting, tracking, and safety systems

---

## Section 1: Current Guardrails Inventory

### 1A. Draft Outreach API (`src/app/api/ask/draft-outreach/route.ts`, lines 14-37)

**System prompt guardrails:**
- No emojis (line 29)
- LinkedIn intro under 300 characters (line 30)
- LinkedIn follow-up under 60 words (line 31)
- "Sound like a real person, not a marketing template" (line 35)

**MISSING guardrails:**
- NO "no pricing" rule in this prompt
- NO sales methodology reference
- Actually ENCOURAGES mentioning "payout rates" in LinkedIn intro instruction (line 23: "Mention specific value (payout rates, call volume, verticals)")
- No banned phrases list
- No personalization check
- No intrigue/curiosity framework

**Risk: HIGH.** This is the simpler of two draft endpoints. It actively instructs Claude to mention payout rates in outreach.

### 1B. Draft Outreach Tool (`src/lib/tools/draft-first-outreach.ts`, lines 93-253)

**System prompt guardrails (VEE L2):**
- 19 banned phrases (lines 103-123) including "I wanted to reach out", "Would love to hop on a call", "Synergy", "Mutual benefit", etc.
- Character limits enforced: 300 for connection requests, 500 for DMs (lines 95-98)
- Hard max 500 chars regardless of format (line 101)
- Architecture pattern: Hook -> Bridge -> Soft Ask (lines 199-203)
- Soft ask must be "answerable in under 10 words" and "NEVER a request for a meeting, call, or demo" (line 181)
- No emojis, no em-dashes, no hashtags (line 213)
- Each version must reference prospect's vertical or outreach angle (line 212)
- Each version must have at least one detail that couldn't be copy-pasted to another prospect (line 213)
- Truncation safety net at character limits (lines 157-170)
- Banned phrase detection post-generation with warnings (lines 152-155, 401-408)
- `ready_to_send` gated on all versions present + under limit + zero banned phrases (lines 441-444)
- Personalization check output field (lines 436-438)
- Three versions: Direct, Conversational, Curiosity (lines 195-198)
- AI recommendation of best version based on seniority/classification (lines 419-432)

**MISSING guardrails:**
- NO "no pricing" rule in this tool's prompt
- NO explicit pricing protection
- Prospect brief includes `outreach_angle` and `likely_verticals` but no pricing data is passed in by design (the input schema has no rate/pricing fields)

**Risk: MEDIUM.** The tool doesn't have access to pricing data in its inputs, which is an implicit guard. But the system prompt doesn't explicitly forbid pricing mentions if the operator includes them in `outreach_angle` or `initial_notes`.

### 1C. Outreach Draft Route (`src/app/api/outreach/draft/route.ts`, lines 117-153)

**System prompt guardrails (Persuasion Engine):**
- Identity-based influence framework: Anchor -> Dissonance -> Bridge (lines 125-129)
- Contact-type-specific patterns for publishers, buyers, brokers, new contacts (lines 131-139)
- **FABRICATION REFUSAL hard rule** (lines 141-142): If operator asks to say something contradicted by data, Milo MUST refuse and explain why
- "NEVER: fabricate traits, use guilt/shame/fear, lie about scarcity, disparage competitors, promise unverifiable outcomes" (line 144)
- Tone-aware (casual/professional/direct, line 146-148)
- Platform-aware: email gets subject line, LinkedIn under 300 chars, Teams casual (lines 149-152)
- Data enrichment: pulls actual performance data, payment history, pipeline info before drafting (lines 37-115, 169-173)
- Only references numbers that appear in data context (line 139)

**MISSING guardrails:**
- NO "no pricing" rule in this prompt
- Actually ENRICHES outreach with real revenue data, payout data, answer rates, conversion rates (lines 176-181)
- Payment data (overdue amounts, days-to-pay) is included in context (lines 178-181)
- Pipeline info including verticals and campaign types is exposed (lines 183-186)

**Risk: HIGH.** This endpoint feeds real financial performance data into the outreach prompt. The model could (and is implicitly encouraged to) reference revenue figures, conversion rates, and even overdue payment amounts in outreach messages.

### 1D. Draft Verify Route (`src/app/api/ask/draft-verify/route.ts`, lines 17-38)

**System prompt guardrails:**
- "NEVER include pricing, payout rates, or deal terms. No dollar amounts." (line 36)
- "Mentioning specific rates leaks deal terms to prospects" (line 36)
- PPC notation clarity: "$X/Y means $X CPL with Y-second buffer duration" (line 37)
- Don't mention "we're verifying them" (line 35)
- Keep under 80 words (line 33)

**Risk: LOW.** This is the only initial-outreach endpoint with an explicit pricing prohibition.

### 1E. LinkedIn Draft Cron (`src/app/api/cron/linkedin-draft/route.ts`, lines 101-124)

**System prompt guardrails:**
- "NEVER include pricing, payout rates, or deal terms. No dollar amounts." (line 119)
- Under 150 words (line 107)
- No emojis, no hashtag spam (line 115)
- Tiffani's voice (not corporate announcement) (line 114)
- Posts are NEVER auto-published (line 13) -- Tiffani reviews manually

**Risk: LOW.** Has the explicit pricing prohibition.

### 1F. Verify Route (`src/app/api/verify/route.ts`, line 454)

**Guardrail:** "NEVER include pricing, payout rates, or deal terms. No dollar amounts."

**Risk: LOW.** Has the explicit pricing prohibition.

---

## Section 2: Pricing Protection Assessment

### Risk Level: HIGH -- Critical Gaps Exist

#### What pricing data exists in the system

| Data Field | Source Table | Exposed To |
|---|---|---|
| `buyer_rate` | `campaigns` | `handle_objection` tool, main ask route enrichment |
| `publisher_rate` | `campaigns` | `handle_objection` tool, main ask route enrichment |
| `payout_rate` | `setup-new-publisher` tool | Tool internals only |
| `revenue` | `call_records` | `outreach/draft` enrichment, main ask route |
| `payout` | `call_records` | `outreach/draft` enrichment, main ask route |
| `margin` | Computed | main ask route vertical data |
| `revenue_at_risk` | `disputes` | main ask route dispute context |
| `amount_due` | `invoices` | `outreach/draft` payment enrichment |
| `contracted_rates` | via `milo-queries` | main ask route [CONTRACTED RATES: ...] tags |

#### Explicit pricing guards (present)

1. `draft-verify/route.ts` line 36: "NEVER include pricing, payout rates, or deal terms."
2. `cron/linkedin-draft/route.ts` line 119: "NEVER include pricing, payout rates, or deal terms."
3. `verify/route.ts` line 454: "NEVER include pricing, payout rates, or deal terms."

#### Missing pricing guards (gaps)

1. **`ask/draft-outreach/route.ts`** -- NO pricing guard. Actively says "Mention specific value (payout rates, call volume, verticals)" on line 23.
2. **`outreach/draft/route.ts`** -- NO pricing guard. Feeds real revenue, payout, overdue invoice, and conversion data into the prompt.
3. **`tools/draft-first-outreach.ts`** -- NO pricing guard in system prompt. Relies on input schema not including pricing fields (implicit, not explicit).
4. **`tools/handle-objection.ts`** -- Feeds `buyer_rate` and `publisher_rate` from campaigns table directly into the prompt (line 180). The system prompt says "ground any rate/margin claims in the campaign context provided" (line 190), which means the model IS expected to reference rates in objection responses. While objection handling is mid-funnel (not cold outreach), this data could leak into copy-pasted messages.
5. **Sales pill (`@milo/voice/src/pills/sales.ts`)** -- No pricing prohibition. Outreach framework mentions "include a proof point -- a specific number" (line 13). Combined with [CONTRACTED RATES: ...] and [LIVE DATA: ...] tags injected by the main ask route, the sales pill could generate outreach containing actual rates.
6. **Publisher pill and Buyer pill** -- No outreach-specific pricing guards.

#### Prompt injection risk

**MEDIUM-HIGH.** An operator could type something like "Draft an outreach email to X and include our payout rates" into the sales or publisher pill. Because:
- The main ask route injects `[CONTRACTED RATES: publisher_rate=$X]` into the user message (line 673)
- The sales pill system prompt says "incorporate these specific numbers into your analysis" (line 59)
- There is no instruction saying "do not include these numbers in outreach drafts"

The system would likely comply and include real rates in the draft.

---

## Section 3: Sales Methodology Usage

### Referenced methodologies

| Methodology | Where | Implementation Depth |
|---|---|---|
| **Hormozi-influenced** | Sales pill (`sales.ts` line 12) | Partial: 5-point outreach framework. Not the full HORMONES acronym. |
| **Identity-based influence** | `outreach/draft/route.ts` lines 124-144 | Deep: Anchor-Dissonance-Bridge pattern with contact-type templates |
| **VEE L2 Architecture** | `draft-first-outreach.ts` lines 199-203 | Deep: Hook-Bridge-Soft Ask with banned phrases |
| **Fabrication Refusal** | `outreach/draft/route.ts` lines 141-142 | Hard rule: model must refuse fabricated claims |

### What the "Hormozi-influenced" framework includes

From the sales pill (lines 12-16):
1. Lead with the result they want, not your capabilities
2. Name the specific pain you solve
3. Include a proof point (specific number, named vertical, timeframe)
4. One clear CTA -- don't give three options
5. Short: under 150 words cold, under 100 follow-up

This is a good start but is NOT the full HORMONES method (Hook, Outcome, Reason, Method, Objection, Next Step, Evidence, Scarcity). It is a simplified cold-outreach adaptation.

### What's missing

1. **No formal HORMONES implementation** -- Only a subset of the Hormozi framework is used.
2. **No AIDA, SPIN, Challenger, or Sandler references** anywhere in the codebase.
3. **No template bank or swipe file** -- `message-templates.ts` has 6 operational templates (publisher check-in, buyer bid increase, quality warning, report request, pause warning, capacity outreach) but these are for partner management, NOT cold outreach.
4. **No content library** -- No stored example messages, no A/B tested templates, no reference library of winning messages.
5. **No follow-up message variations** -- The `draft-first-outreach` tool generates 3 variations of the SAME stage (first contact). There are no tools for Day 3 follow-up, Day 7 breakup, etc.

---

## Section 4: Outreach Tracking Completeness

### What's tracked

| Field | Tracked? | Where |
|---|---|---|
| Entity name | YES | `outreach_log.entity_name` |
| Entity type (buyer/publisher) | YES | `outreach_log.entity_type` |
| Channel | YES | `outreach_log.channel` (linkedin_intro, linkedin_followup, email_cold, teams, other) |
| Action | YES | `outreach_log.action` (drafted, copied, sent, responded) |
| Message text | YES | `outreach_log.message_text` |
| Follow-up date | YES | `outreach_log.follow_up_at` (auto-computed per channel) |
| Pipeline stage | YES | `outreach_log.pipeline_entity_id` links to `prospect_pipeline` |
| CRM counterparty | YES | `outreach_log.crm_counterparty_id` links to `crm_counterparties` |
| Template/approach used | NO | Not tracked |
| Which VA sent it | PARTIAL | `outreach_log` has no `sent_by` column. `send_log` stores operator context but `outreach_log` doesn't. |
| Which variation was chosen | NO | Not tracked -- tool generates 3 versions but doesn't know which one was copied |
| Tone/style used | NO | Not tracked |
| A/B test group | NO | Not tracked |
| Response received | YES | `outreach_log.responded_at` (via pending query) and `send_log.replied_at` + `reply_classification` |

### Manager visibility

- **All outreach for a given entity:** YES -- `GET /api/ask/outreach-log?entityName=X` (line 134-164)
- **All outreach by a specific VA:** NO -- No `sent_by` / `operator_id` field on `outreach_log`
- **Pending follow-ups:** YES -- `GET /api/ask/outreach-log?pending=true` (lines 150-155)
- **Pipeline stage tracking:** YES -- `prospect_pipeline.stage` updated on outreach (lines 54-89)

### Additional tracking surfaces

- **`ai_action_log`**: Draft generation logged with full data snapshot (`draft-first-outreach.ts` lines 449-466, `DraftMessageCard.tsx` lines 61-72)
- **`send_log`**: Email sends through milo-outreach recorded with resend_id, gate_state, subject, reply tracking
- **`entity_knowledge`**: Outreach activity bridged into hot_knowledge for Milo memory (`outreach/send/route.ts` lines 112-139)
- **`crm_counterparties.last_contact_at`**: Updated on send/manual-paste
- **`prospect_pipeline.last_contact_at`**: Updated on send/manual-paste
- **`contacts.last_contact_at`**: Updated when contact_id provided
- **`action_items`**: Each email send creates/updates an action item with send metadata

---

## Section 5: Follow-Up Automation State

### Follow-up cadence (what exists)

| Channel | Days Until Follow-Up |
|---|---|
| linkedin_intro | 3 days |
| linkedin_followup | 7 days |
| email_cold | 5 days |
| teams | 3 days |
| other | 7 days |

Source: `outreach-log/route.ts` lines 20-26

### Outreach engine follow-up thresholds

| Pipeline Stage | Days Until Follow-Up Needed |
|---|---|
| outreach | 3 days |
| qualifying | 5 days |

Source: `outreach-engine.ts` lines 82-84

### Drip campaigns

- `drip_campaigns` table exists and is queried by `outreach-engine.ts` (line 159)
- Pipeline stage `drip` exists in the stage enum
- `outreach-engine.ts` fetches due drip touches (next_touch_at <= today) and generates follow-up tasks
- Drip campaigns appear in the morning briefing via `generateOutreachReport()`

### What's missing

1. **No automated "suggest a follow-up" from the chat.** Milo doesn't proactively say "it's been 3 days since outreach to X, draft a follow-up." The follow-up logic lives in the outreach engine for the morning briefing, not in the conversational interface.
2. **No follow-up message templates.** `draft-first-outreach` only drafts first-contact messages. There is no `draft-followup-outreach` tool.
3. **No cadence orchestration.** There is no "Day 1: intro, Day 3: follow-up, Day 7: breakup" system. The `follow_up_at` dates are computed but there is no tool that generates stage-appropriate follow-up content.
4. **No breakup email support.** No tool or template for a final "closing the loop" message.
5. **Reply classification exists** (`reply-classifier.ts`) with 7 categories and state machine advancement, but there is no tool that drafts a response based on the classification. The `classifyReplyForAlert` function returns a `replyDraft` field but it's generated inline by the classifier's system prompt (line 66-70), not by a dedicated drafting tool with the same quality controls as `draft-first-outreach`.

---

## Section 6: Recommended Improvements (Prioritized)

### P0 -- Critical (pricing leaks)

1. **Add explicit "NEVER include pricing" rule to `ask/draft-outreach/route.ts`.**
   - File: `src/app/api/ask/draft-outreach/route.ts`, line 23
   - Current: "Mention specific value (payout rates, call volume, verticals)"
   - Fix: Remove "payout rates" from that instruction and add: "NEVER include pricing, payout rates, margins, or deal terms. No dollar amounts."

2. **Add explicit "NEVER include pricing" rule to `outreach/draft/route.ts`.**
   - File: `src/app/api/outreach/draft/route.ts`, SYSTEM_PROMPT (line 119)
   - Add to rules section: "NEVER include pricing, payout rates, margins, deal terms, or dollar amounts in external outreach messages. Revenue and payment data is for your analysis context only -- never surface it in the message you draft."

3. **Add explicit "NEVER include pricing" rule to `tools/draft-first-outreach.ts`.**
   - File: `src/lib/tools/draft-first-outreach.ts`, buildPrompt() system prompt (line 193)
   - Add to HARD RULES section: "NEVER include pricing, payout rates, margins, or deal terms in the message text. No dollar amounts."

4. **Add pricing-leak detection as post-generation check in `draft-first-outreach.ts`.**
   - After banned phrase detection (line 403), add a regex scan for dollar amounts (`$\d+`) and common pricing terms ("payout", "rate", "margin", "cost per") in the generated message text. If found, add a warning and set `ready_to_send = false`.

### P1 -- High (sales methodology)

5. **Implement full HORMONES framework as an optional parameter on `draft-first-outreach`.**
   - Add a `methodology` input field: `'hormozi' | 'vee_l2' | 'identity_based'`
   - Default to `'vee_l2'` (current behavior)
   - When `'hormozi'`, use a prompt that implements Hook, Outcome, Reason, Method, Objection-preempt, Next Step, Evidence, Scarcity (minus fabricated scarcity)

6. **Add a "no hard sell" guardrail to the sales pill (`@milo/voice/src/pills/sales.ts`).**
   - Between the outreach framework (line 16) and response format (line 18), add: "COLD OUTREACH RULE: When drafting outreach messages, NEVER include pricing, payout rates, or deal terms. Create intrigue about the partnership -- the prospect should want to learn more, not feel like they're being sold to. First-contact messages are about opening a conversation, not closing a deal."

### P2 -- Medium (tracking gaps)

7. **Add `operator_id` / `sent_by` column to `outreach_log` table.**
   - Currently there is no way for a manager to see "all outreach sent by Vee" vs "all outreach sent by Cam."
   - The `outreach-log/route.ts` POST handler should accept and store the operator identity.

8. **Track which message variation was chosen.**
   - After `draft-first-outreach` generates 3 versions, the DraftMessageCard copy action should include a `version_name` field in the telemetry payload.
   - Current: `ai_action_log` records `draft_copied` with `draft_text` but no version name.

9. **Track template/approach used.**
   - Add a `methodology` or `approach` field to `outreach_log` to record whether the message used VEE L2, Hormozi, identity-based, or manual drafting.

### P3 -- Low (follow-up automation)

10. **Build a `draft-followup-outreach` tool.**
    - Accepts `prospect_id` + `stage` (follow-up number) + `previous_messages` context
    - Generates stage-appropriate follow-up: Day 3 value-add, Day 7 breakup, etc.
    - Same quality controls as `draft-first-outreach` (banned phrases, character limits, pricing guard)

11. **Add proactive follow-up suggestions in the chat interface.**
    - When an operator opens a conversation, check `outreach_log` for entities with overdue `follow_up_at`. Surface a suggestion card: "It's been 4 days since your outreach to X. Draft a follow-up?"

12. **Multi-channel template differentiation.**
    - Currently `draft-first-outreach` only generates LinkedIn messages. Email and phone scripts are handled by the simpler `draft-outreach/route.ts` or `outreach/draft/route.ts`.
    - Unify into the tool with channel-specific formats: LinkedIn (300/500 char), Email (subject + 3-4 sentences), Phone script (talking points).

---

## File Reference Index

| File | Lines | What It Does |
|---|---|---|
| `src/app/api/ask/draft-outreach/route.ts` | 1-80 | Simple 3-channel draft generator (LinkedIn intro/followup + cold email). NO pricing guard. |
| `src/app/api/ask/draft-verify/route.ts` | 1-99 | Verification outreach draft. HAS pricing guard (line 36). |
| `src/app/api/ask/outreach-log/route.ts` | 1-164 | Log + query outreach activity. Tracks channel, action, follow-up dates. |
| `src/app/api/outreach/draft/route.ts` | 1-230 | Data-enriched outreach draft with Persuasion Engine. NO pricing guard. |
| `src/app/api/outreach/send/route.ts` | 1-215 | Email send via milo-outreach 4-gate safety system. Logs to CRM/pipeline. |
| `src/app/api/outreach/classify-reply/route.ts` | 1-211 | Manual reply classification with state machine advancement. |
| `src/app/api/outreach/record-manual-send/route.ts` | 1-152 | Manual-paste tracking (operator copied draft, sent externally). |
| `src/app/api/cron/linkedin-draft/route.ts` | 1-305 | Weekly LinkedIn recruitment posts. HAS pricing guard. |
| `src/lib/tools/draft-first-outreach.ts` | 1-482 | VEE L2 tool: 3 variations, banned phrases, character caps. NO pricing guard. |
| `src/lib/tools/research-prospect.ts` | 1-497 | Prospect intelligence: classification, scam check, outreach angle. |
| `src/lib/tools/handle-objection.ts` | 1-402 | Objection handling: fetches buyer/publisher rates from campaigns table. |
| `src/lib/tools/warm-handoff-to-tiffani.ts` | 1-376 | Handoff package for CEO. |
| `src/lib/outreach-engine.ts` | 1-383 | Morning briefing outreach report: pipeline snapshot, follow-ups, drip. |
| `src/lib/message-templates.ts` | 1-43 | 6 operational message templates (partner management, not cold outreach). |
| `src/lib/reply-classifier.ts` | 1-236 | 7-category reply classification with state machine actions. |
| `src/lib/dispute-drafter.ts` | 1-193 | Dispute message templates (initial, follow-up, escalation). |
| `src/components/shared/DraftMessageCard.tsx` | 1-241 | UI card with Edit/Regenerate/Copy + ai_action_log telemetry. |
| `@milo/voice/src/pills/sales.ts` | 1-84 | Sales pill system prompt with Hormozi-influenced framework. |
| `@milo/voice/src/pills/vet.ts` | 1-110 | Vet pill system prompt. |

---

Report at: https://raw.githubusercontent.com/moscaleflow/conductor-mark-public/main/reports/outreach-guardrails-audit-2026-05-06.md
