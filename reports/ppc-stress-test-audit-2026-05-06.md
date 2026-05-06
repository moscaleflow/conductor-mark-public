# PPC Workflow Stress Test -- Operator Day-in-the-Life Audit

**Date:** 2026-05-06
**Auditor:** Coder-3 (Research Lane)
**Repos read:** milo-for-ppc (primary), milo-engine (packages)
**Perspective:** Virtual assistant doing publisher recruitment, vetting, onboarding, outreach, and pipeline management daily

---

## Scenario Walkthrough (Steps 1--41)

### Morning: VA logs in

**1. What does the operator dashboard show?**
- **Status:** WORKS
- **Evidence:** `src/app/operator/page.tsx` L1-568. Non-admin users get MorningBriefing (greeting + needs-you cards), PillBar (role-filtered quick-action pills), OnboardingChecklist (all active runs, default collapsed), and SearchBar. Admin users get Top5Frame with executive cards.
- **Gap:** No "what needs attention today?" summary sentence that synthesizes across alert types. The `summary_line` from `/api/operator` is generic ("Milo processed X calls overnight. N things need you") but doesn't prioritize or categorize. The morning_priority_briefing tool (L300-421) does Fab's 4-tier ranking, but it's only available through the chat -- there's no widget on the operator page that auto-renders the ranked list.

**2. Are yesterday's alerts still visible?**
- **Status:** PARTIAL
- **Evidence:** MorningBriefing renders `needs_you` items from `/api/operator`. The briefing API generates alerts fresh each load. Alerts table persists in Supabase, but the operator page only shows the current briefing -- there's no "yesterday" tab or "unresolved from yesterday" section.
- **Gap:** An alert resolved yesterday is gone. An alert from yesterday that was NOT resolved may or may not reappear depending on whether the briefing logic re-generates it. No explicit "carry-forward" mechanism for unresolved items.

**3. Can alerts be assigned to specific VAs?**
- **Status:** WORKS
- **Evidence:** `src/lib/alert-routing.ts` L1-150. Alerts have `target_user_id` column. Resolution chain: explicit `route_to` name > CRM `assigned_to` > entity-type default (publisher->Jackie, buyer->Fab, did->Malvin) > broadcast (null). EntityVetCard has an Assign dropdown (L1306-1336) that calls `onAssign(name)`.
- **Gap:** No "my alerts" vs "all alerts" filter on the operator page. The PillDrawer fetches pill-specific data, but there's no personal alert inbox. Alerts are routed server-side but the operator UI doesn't filter by `target_user_id`.

**4. Is there a "stalled items" view?**
- **Status:** PARTIAL
- **Evidence:** OnboardingChecklist (L1-774) shows active onboarding runs with stall detection (stall banner when a step has been active 3+ days). The Kanban board shows pipeline entities with relative time ("5d ago"). OutreachEngine (L225-280) identifies follow-up needs.
- **Gap:** No unified "stalled items" view. Onboarding stalls show in OnboardingChecklist. Pipeline stalls show on the Kanban. Outreach stalls are only visible via the morning-priority-briefing tool in chat. A VA must check 3 different places.

### Task 1: Find new publishers

**5. VA asks Milo "find publishers for auto insurance in Texas"**
- **Status:** PARTIAL
- **Evidence:** The /api/ask route (L1-120+) handles free-text queries. The campaign-matching engine (`src/lib/campaign-matching.ts` L1-257) can match publishers to campaigns by vertical and call type. But `matchPublishersToCampaign` only looks at EXISTING publishers already on other campaigns -- it doesn't search externally.
- **Gap:** Milo cannot search for NEW publishers on demand. The publisher-discovery cron (`/api/cron/publisher-discovery`) uses Claude with web_search to find new publishers, but it runs automatically once daily at 9 AM Mountain -- there's no way for a VA to trigger it on-demand from chat. A VA asking "find publishers" would get campaign-matching results (existing publishers who could be added to a campaign), not net-new discovery.

**6. Does Milo know what campaigns are currently running?**
- **Status:** WORKS
- **Evidence:** campaign-matching.ts queries `campaigns` table for status='active', including vertical, call_type, buyer_rate, publisher_rate. The system prompt (milo-prompt.ts L8) says "Campaign configurations (verticals, billing types, rates)". Multiple tools query campaigns.
- **Gap:** The system prompt does NOT inject the current campaign list into every conversation. Campaign awareness requires an explicit query. Milo knows campaigns exist and can look them up, but doesn't proactively include "here are our active campaigns" in context.

**7. Does Milo know current cap availability?**
- **Status:** MISSING
- **Evidence:** No `cap` or `capacity` field found in campaign-matching.ts or campaign queries. No reference to "uncapped" or "cap_remaining" anywhere in the codebase.
- **Gap:** There is no cap tracking. Milo cannot tell a VA "Campaign X has unlimited cap" or "Campaign Y is capped at 50 calls/day and we're at 40."

**8. Can Milo proactively suggest "Campaign X is uncapped, find publishers for it"?**
- **Status:** MISSING
- **Evidence:** No proactive campaign-to-publisher suggestion mechanism exists. The publisher-discovery cron checks all active verticals but doesn't specifically target uncapped campaigns (because cap data doesn't exist).
- **Gap:** Without cap data, Milo cannot distinguish between campaigns that need more traffic and those that are saturated.

**9. When results come back, does each entity card have a clear NEXT STEP?**
- **Status:** WORKS
- **Evidence:** `src/lib/next-action.ts` L1-104 provides stage-specific hints. EntityVetCard has hero buttons that vary by verdict: "Add to CRM" for likely_real, "Add to Blacklist" for likely_scam, "Draft verification message" for unclear. Research prospect output includes `next_action` field (CAM_TLP2 / ask_clarifying_question / reject).
- **Gap:** The next-action engine is good but it's not displayed on every entity rendering. KanbanBlock cards don't show a next-action hint -- just entity name, vertical, and relative time.

### Task 2: Vet a publisher

**10. VA clicks "Vet" on an entity card -- what happens?**
- **Status:** WORKS
- **Evidence:** EntityVetCard (L1-1502) renders full vet results. The `/api/ask` route handles vet intents, calling `research_prospect` tool which classifies as BUYER/PUBLISHER/BROKER/UNKNOWN, runs blacklist screen, produces flags/facts/recommendation. The vet result includes `verdict`, `verdict_summary`, `facts`, `flags`, `recommendation`.
- **Gap:** None significant -- this flow is well-built.

**11. Does vetting check: blacklist, existing CRM, web presence, LinkedIn?**
- **Status:** PARTIAL
- **Evidence:** research-prospect.ts (L388-393) runs blacklist screen via `/api/blacklist/screen` in parallel with LLM analysis. vet-lookup.ts checks for existing CRM entries. The LLM prompt (L225-266) instructs analysis of operator-pasted LinkedIn text. Web search is used in publisher-discovery cron but NOT in research_prospect tool directly.
- **Gap:** No automated web presence check during vetting. The LLM analyzes whatever text the operator pastes, but doesn't independently verify the company's website or LinkedIn. The VA must manually check web presence and paste findings.

**12. Can the VA upload a LinkedIn screenshot for manual review?**
- **Status:** PARTIAL
- **Evidence:** The /api/ask route (L346) accepts `imageFile: { name, data, mimeType }`. The route sends images to Claude via base64 (L1616-1620). So image upload to Milo chat IS supported.
- **Gap:** There's no dedicated "upload LinkedIn screenshot for vetting" button on the EntityVetCard. The VA would need to use the general chat image upload and say "vet this person based on this screenshot." The research_prospect tool only accepts `pasted_linkedin_text` (string), not images. So the image goes to generic Claude, not the structured vetting pipeline.

**13. After vetting, what's the next step?**
- **Status:** WORKS
- **Evidence:** EntityVetCard hero button system (L564-591): likely_real -> "Add to CRM", likely_scam -> "Add to Blacklist", unclear -> "Draft verification message". Below the hero button: Assign, Draft outreach, Ask Milo more, Re-vet, Edit facts, Copy.
- **Gap:** No "send onboarding link" button directly on the vet card. After adding to CRM, the VA must separately ask Milo to generate an onboarding link.

**14. Does the vet result persist?**
- **Status:** WORKS
- **Evidence:** `saveVetResult` (imported in ask route) saves to `vet_results` table linked via `chat_log_id`. The history API (L118-130) joins vet_results to restore vet cards on page reload. EntityVetCard has cached/cachedDaysAgo display and "Re-vet" button.
- **Gap:** None -- vet results persist and are restorable.

### Task 3: Send onboarding link

**15. How does the VA get an onboarding link?**
- **Status:** WORKS
- **Evidence:** `publisher-onboarding.ts` (L1-80+) is the convergence point. The system prompt says "Generate onboarding links for new publishers or buyers -- just say who and Milo kicks it off." The `setup-new-publisher` tool and `onboarding-token.ts` generate HMAC-signed tokens with 72-hour expiry.
- **Gap:** None -- asking Milo works.

**16. Does the link go through Milo chat?**
- **Status:** WORKS
- **Evidence:** System prompt L83-90 confirms Milo handles onboarding intents. The /api/ask route processes ONBOARD_PUBLISHER intent and returns a pre-built response.
- **Gap:** No dedicated button outside of chat. Everything goes through the conversation.

**17. Is the link signed (HMAC + expiry)?**
- **Status:** WORKS
- **Evidence:** `onboarding-token.ts` L85-99. `signToken()` creates `base64url(payload).base64url(hmac)` with configurable expiry (default 72 hours). `verifyToken()` validates signature and checks `exp` field. Legacy unsigned tokens are also accepted with a console warning.
- **Gap:** Legacy unsigned tokens are still accepted (L177-220). This is a security concern -- old tokens with no expiry or signature are honored.

**18. What information is pre-filled in the link?**
- **Status:** WORKS
- **Evidence:** `OnboardingTokenPayload` (L57-68) includes: lead_id, company, contact, rep, role, email, phone, verticals. All can be pre-filled from pipeline data.
- **Gap:** None.

### Task 4: Draft outreach

**19. VA asks Milo to "draft an outreach message for [Publisher Name]"**
- **Status:** WORKS
- **Evidence:** Two paths exist:
  1. `draft-first-outreach` tool (L1-482): Full VEE L2 writer. Requires a `prospect_brief` from `research_prospect`. Produces 3 LinkedIn variations (Direct/Conversational/Curiosity).
  2. `/api/ask/draft-outreach` route (L1-80): Simpler path from EntityVetCard. Produces linkedin_intro (300 char), linkedin_followup (60 words), email_cold.
- **Gap:** The two paths produce different formats and have different guardrails. The tool path is more rigorous; the API path is lighter. A VA might not know which they're getting.

**20. Does the draft follow the HORMONES method?**
- **Status:** MISSING (HORMONES not referenced)
- **Evidence:** Grep for "HORMONES", "PASTOR", "AIDA", "PAS" across the entire src/ tree returned zero results. The outreach drafting uses a custom architecture: Hook > Bridge > Soft Ask (from VEE L2 skill, L193-229).
- **Gap:** No named sales framework (HORMONES or otherwise) is referenced. The VEE L2 architecture (Hook/Bridge/Soft Ask) is a custom framework specific to this codebase. If HORMONES is expected, it's not implemented.

**21. Does the draft avoid revealing pricing?**
- **Status:** WORKS
- **Evidence:** The draft-first-outreach tool prompt (L193-228) says "Soft ask -- answerable in under 10 words. The ask must never be a request for a meeting, call, or demo." The simpler `/api/ask/draft-outreach` route (L14-37) prompt says "Be specific about TLP's value: we buy/broker calls across SSDI, Auto, Home Services..." but does NOT explicitly say "never include pricing." Publisher-discovery cron (L18) explicitly says "NEVER includes pricing/payout data."
- **Gap:** The `/api/ask/draft-outreach` route does NOT have an explicit "never reveal pricing" guardrail. It mentions specific verticals and pitches "payouts" generically. The tool-based path is safer, but the simpler API path could leak pricing hints. The system prompt (milo-prompt.ts) also has no pricing guardrail for outreach drafts.

**22. Can the VA get multiple draft variations?**
- **Status:** WORKS
- **Evidence:** draft-first-outreach tool always produces 3 variations (Direct/Conversational/Curiosity) with a recommendation. The API path produces 3 channel-specific drafts (LinkedIn intro, LinkedIn followup, cold email).
- **Gap:** None.

**23. Is the draft tracked?**
- **Status:** WORKS
- **Evidence:** draft-first-outreach logs to `ai_action_log` (L449-465). The EntityVetCard outreach copy button logs to `/api/ask/outreach-log` with `advancePipeline: true` (L1024-1035). Pipeline stage advances when outreach is sent.
- **Gap:** Outreach drafts are tracked when copied. But if a VA composes their own message outside Milo, that's invisible.

**24. What sales frameworks are referenced?**
- **Status:** Custom framework only
- **Evidence:** VEE L2 (Hook/Bridge/Soft Ask) in draft-first-outreach.ts. handle-objection.ts uses objection type classification (rate/terms/timing/trust/competition). The system prompt has "PERSUASION AWARENESS" section (L133-140) with identity-based persuasion, aspiration framing, and "never fabricate traits."
- **Gap:** No industry-standard named framework (HORMONES, SPIN, PASTOR, Sandler, etc.). The custom approach is coherent but undocumented as a named methodology.

### Task 5: Follow up on pending items

**25. How does the VA know which publishers haven't responded to outreach?**
- **Status:** PARTIAL
- **Evidence:** OutreachEngine (L225-280) identifies follow-up needs based on `last_contact_at` and stage-specific thresholds (outreach: 3 days, qualifying: 5 days). Alert-step-chain (L37-43) defines outreach-followup flow: outreach_sent -> waiting_on_reply (5 days stale) -> follow_up_if_stale. The `stale-relationships` cron runs daily.
- **Gap:** This data is generated server-side but not surfaced in a dedicated "follow-up queue" on the operator page. The VA sees it in the morning briefing needs-you cards, or by opening the chat and asking. There's no persistent "pending responses" panel.

**26. How does the VA know which publishers started onboarding but stalled?**
- **Status:** WORKS
- **Evidence:** OnboardingChecklist (L340-348) shows stall detection with days-active counter and timeout. The `onboarding-stall-check` cron (vercel.json L64) runs daily at 7 AM Mountain. Stalled steps get a warning banner.
- **Gap:** Stall alerts are generated but the OnboardingChecklist on the operator page starts collapsed by default. A VA must expand it to see stalls.

**27. Is there a "follow-up queue"?**
- **Status:** MISSING (as a dedicated surface)
- **Evidence:** Follow-ups are generated in outreach-engine.ts and surfaced through morning briefing and alert-step-chain. But there is no dedicated "/follow-ups" page or PillDrawer tab that shows a filtered list of all entities awaiting response.
- **Gap:** A VA must either: check the morning briefing cards, ask Milo via chat, or scan the Kanban board manually. No single-click "show me everything I need to follow up on today."

**28. Does Milo proactively remind about stale items?**
- **Status:** PARTIAL
- **Evidence:** The morning-assignments cron (vercel.json L12) runs at 8 AM Mountain and sends personalized task lists via Teams DMs. The stale-relationships cron detects dormant entities. Alert-step-chain auto-spawns follow-up alerts when step 2 is stale for 5+ days.
- **Gap:** Milo doesn't proactively message within the app. Proactive reminders go to Teams DMs (external). Inside the Milo app, the VA must log in and look at the briefing -- Milo doesn't push notifications or in-app alerts that say "Hey, you haven't followed up with X."

### Task 6: Contract management

**29. When a publisher sends back a signed MSA, what's the flow?**
- **Status:** WORKS
- **Evidence:** `ingest-returned-contract` tool handles uploaded contracts. `analyze-contract` tool runs risk analysis. `@milo/contract-analysis` engine (milo-engine packages) does post-processing, scoring, jurisdiction checks. ContractAnalysisDisplay and IssueCard components render results.
- **Gap:** None in the analysis flow.

**30. Does uploading the signed doc auto-advance the onboarding step?**
- **Status:** PARTIAL
- **Evidence:** The onboarding state machine (`@milo/onboarding` package) has step advance/skip capabilities. OnboardingChecklist has "Mark Complete" and "Skip" buttons per step. But `ingest-returned-contract` tool does NOT call the onboarding advance API.
- **Gap:** Uploading a signed contract does NOT automatically mark the MSA onboarding step as complete. The VA must manually click "Mark Complete" on the onboarding checklist after uploading.

**31. Can the VA see current contract status for any entity?**
- **Status:** WORKS
- **Evidence:** System prompt L84-89 lists document capabilities. PillDrawer has 'contracts' and 'esign' pill IDs. ContractClauseList component exists for operator view.
- **Gap:** None.

**32. Do alerts fire when contracts stall?**
- **Status:** WORKS
- **Evidence:** onboarding-stall-check cron (vercel.json L64) runs daily. OnboardingChecklist shows stall warnings. Alert routing creates alerts for stalled onboardings.
- **Gap:** None for onboarding-linked contracts. Standalone contract negotiations outside onboarding are not auto-monitored.

**33. Is there a clear path: alert -> contract -> single next action?**
- **Status:** PARTIAL
- **Evidence:** Alert cards have `action_prompt` that opens Milo chat with context. From chat, the VA can ask about contract status. But there's no single-click path from an alert card directly to the contract view for that entity.
- **Gap:** Alert -> Chat -> "show me the contract" is 2 clicks too many. Should be: alert -> entity contract panel.

### Task 7: Pipeline board usage

**34. Does the Kanban board show all entities correctly?**
- **Status:** WORKS
- **Evidence:** KanbanBlock.tsx (L1-494) renders 6 columns: Unverified, Vetted, Outreach, Qualifying, Onboarding, Activation. Cards show entity name, contact, vertical, ownership (Vee/Cam), relative time, source badge, outreach status badge, risk dot.
- **Gap:** No "Active" column on the Kanban -- graduated entities (active/dormant/blacklisted) show only as counts in a summary line at the bottom.

**35. Can a VA drag an entity and get confirmation of what happens?**
- **Status:** PARTIAL
- **Evidence:** KanbanBlock supports drag-drop (L112-189). Dashed border and color highlight show the target column. `onStageChange` callback fires on drop.
- **Gap:** No confirmation dialog. The drag-drop is immediate -- the VA drops, the stage changes. No "Are you sure you want to move X from Outreach to Qualifying?" No indication of what gates or requirements exist for the target stage. No undo.

**36. Are activation gates enforced (MSA + tax doc + IO required)?**
- **Status:** MISSING
- **Evidence:** The Kanban drag-drop calls `onStageChange(prospectId, targetStage)` which maps to the column's default stage. There is no gate check. A VA can drag an entity from Unverified directly to Activation without any documents.
- **Gap:** Critical. No activation gates are enforced on the Kanban board. The onboarding flow has step requirements, but the Kanban board bypasses them entirely. A VA could accidentally activate an entity with no MSA, no W-9, no IO.

**37. Does the board show which entities are stalled?**
- **Status:** PARTIAL
- **Evidence:** Cards show relative time ("5d ago", "12d ago"). Risk dots show red/yellow for high/medium risk. But there's no explicit "stalled" badge or visual indicator based on time-in-stage thresholds.
- **Gap:** A card showing "12d ago" in Outreach doesn't scream "stalled" the way a red "STALLED" badge would. The VA must mentally calculate whether 12 days in outreach is normal.

### Task 8: End of day / handoff

**38. Can a VA log what they did today?**
- **Status:** WORKS
- **Evidence:** EODModal.tsx (L1-60+) exists with auto-generated content: milo_queries, alerts_resolved, tasks_completed, conversation_count, AI summary. Also has manual_additions, tasks_in_progress, questions_blockers, notes_suggestions fields.
- **Gap:** None -- EOD reporting is solid.

**39. If they log out and come back tomorrow, is all context preserved?**
- **Status:** PARTIAL
- **Evidence:** `/api/ask/history` (L1-179) returns today's chat logs from `chat_logs` table, filtered by IP hash and today's Denver date. Vet results are joined and restored. Session includes up to 30 message pairs.
- **Gap:** History is IP-based, not user-based. If a VA logs in from a different IP (different office, VPN change), their history is gone. History is also limited to today -- yesterday's context is lost. The `conversationHistory` parameter in the ask route allows passing prior messages, but the client must load and supply them.

**40. Does the session history show today's full work?**
- **Status:** WORKS
- **Evidence:** History API returns all of today's messages (session gap logic was removed per comment at L109-110). Capped at 30 message pairs.
- **Gap:** 30-pair cap could truncate a heavy day. No pagination for older messages.

**41. Are assigned alerts/action items preserved across sessions?**
- **Status:** WORKS
- **Evidence:** Alerts and action items live in Supabase tables (alerts, action_items). They persist regardless of session. The operator page re-fetches briefing on load, and refreshes when chat closes (L286-289).
- **Gap:** Items are preserved in the database, but the operator page only shows the current briefing generation. Resolved items disappear. There's no "completed today" view.

---

## Specific Concern Audits

### Campaign Awareness

- **Campaign routes:** No `/api/campaigns/` directory exists. Campaign data is queried directly from Supabase in lib files (campaign-matching.ts, publisher-discovery cron).
- **System prompt context:** The system prompt (milo-prompt.ts L6-7) says Milo knows "Campaign configurations (verticals, billing types, rates)" but does NOT inject a current campaign list into the prompt. Campaigns are queried on-demand when the VA asks.
- **Proactive campaign finding:** The publisher-discovery cron (vercel.json L56) runs daily and uses `getActiveVerticals()` to find campaigns needing traffic. But there's no mechanism for Milo to say "Hey, we have a new Auto Insurance campaign that's uncapped -- should I find publishers?"
- **Cap awareness:** MISSING entirely. No cap column, no cap check, no "uncapped campaign" concept.

### Outreach Guardrails

- **draft-first-outreach tool:** Strong guardrails. 19 banned phrases (L103-123). Character limits enforced (300 for connection requests, 500 for DMs). Hard 500-char global cap. Banned phrase detection on output. Custom VEE L2 architecture (Hook/Bridge/Soft Ask). Classification-specific soft asks. Personalization check.
- **`/api/ask/draft-outreach` route:** Weaker guardrails. No banned phrase list. No explicit "never reveal pricing" instruction. Says "Be specific about TLP's value" which could lead to pricing mentions. No character limit enforcement post-generation.
- **System prompt persuasion section:** Identity-based persuasion rules (L133-140). "NEVER fabricate traits, use guilt/shame/fear, or lie about scarcity." Has opt-out: "If operator says 'too salesy' -- drop all persuasion framing."
- **Pricing protection:** Publisher-discovery cron explicitly says "NEVER includes pricing." The outreach tools do NOT have this explicit rule. handle-objection.ts (L185-232) says "Ground any rate/margin claims in the campaign context" and has access to actual rates. A VA could get rate info leaked through an objection response.

### Alert Assignment

- **Schema:** Alerts table has `target_user_id` column. Alert-routing.ts resolves targets through 4 tiers.
- **"My alerts" filter:** MISSING. No `/api/alerts?user_id=me` endpoint. No PillDrawer configuration for "my alerts." The needs_you briefing cards are NOT filtered by target_user_id -- all operators see all alerts.
- **Assignment UI:** EntityVetCard has assign dropdown. PillDrawer items can be actioned inline. But there's no way to reassign an existing alert from the briefing view.

### LinkedIn / Image Upload

- **Image upload in chat:** WORKS. The /api/ask route accepts `imageFile` with base64 data and sends to Claude vision (L1616-1620).
- **Image upload in vetting flow:** MISSING. research_prospect tool (L319-320) only accepts `pasted_linkedin_text` (string). No image parameter. A VA cannot paste a LinkedIn screenshot and have it flow through the structured vetting pipeline. They can send it to general chat, but it won't produce a structured EntityVetCard with verdict/facts/flags.
- **Dedicated screenshot upload:** MISSING. No "attach LinkedIn screenshot" button on the vet flow. The document dropzone (DocumentDropzone.tsx) exists for contract uploads but is not wired to the vetting flow.

### Proactive Milo Behavior

- **Milo initiates:** PARTIAL via crons only. 16 crons in vercel.json:
  - sync (every 15 min) -- data ingestion
  - evaluate (every 5 min) -- alert generation
  - health-check (hourly) -- system health
  - morning-assignments (daily 8 AM MT) -- VA task lists via Teams
  - publisher-discovery (daily 9 AM MT) -- find new entities
  - onboarding-stall-check (daily 7 AM MT) -- detect stalled steps
  - stale-relationships (daily 8 AM MT) -- dormant entity alerts
  - linkedin-draft (weekly Monday 10 AM MT) -- pre-draft LinkedIn messages
  - error-digest (daily 8 AM MT) -- error summary
- **In-app proactive:** MISSING. Milo never pushes a notification inside the app. All proactive behavior is either: alerts (which the VA must check by loading the page) or Teams DMs (external).
- **Daily briefing feature:** WORKS via morning_priority_briefing tool and MorningBriefing component. But the VA must log in to see it -- no push notification.

---

## Top 10 Bugs/Gaps (Ranked by Operator Impact)

### 1. No activation gates on Kanban drag-drop (CRITICAL)
**File:** `src/components/shared/KanbanBlock.tsx` L166-189
**Impact:** A VA can drag an entity from Unverified to Activation without MSA, W-9, or IO. Bypasses the entire onboarding state machine. Could activate an unvetted, undocumented publisher.
**Fix needed:** Gate check on `onStageChange` -- query onboarding_runs for required step completion before allowing stage advancement to onboarding/activation.

### 2. No "my alerts" personal inbox (HIGH)
**Impact:** In a multi-VA operation, every VA sees every alert. Jackie sees Fab's billing alerts. Vee sees Jackie's QC alerts. Alert routing assigns `target_user_id` server-side but the UI never filters by it. VAs waste time scanning irrelevant items.
**Fix needed:** PillDrawer or operator page needs a "My Tasks" view filtered by session user's `user_id` matching `target_user_id`.

### 3. No follow-up queue as a dedicated surface (HIGH)
**Impact:** VAs who need to follow up on 15 pending outreach messages have no single place to see them. They must check morning briefing (shows top items), ask Milo in chat, or scan the Kanban board manually. Outreach follow-ups fall through cracks.
**Fix needed:** A "Follow-ups Due" pill or widget that shows all entities where `last_contact_at` exceeds the stage threshold.

### 4. No on-demand publisher discovery (HIGH)
**Impact:** A VA who needs publishers for a new campaign TODAY must wait for the daily cron at 9 AM. No way to trigger discovery from chat. Campaign-matching only shows existing publishers.
**Fix needed:** Expose the publisher-discovery logic as a callable tool (not just a cron) so the VA can say "find new publishers for Auto Insurance."

### 5. Signed document upload does NOT auto-advance onboarding (MEDIUM-HIGH)
**File:** `src/lib/tools/ingest-returned-contract.ts` -- no call to onboarding advance
**Impact:** VA uploads a signed MSA, then must separately find and click "Mark Complete" on the onboarding checklist. If they forget, the entity appears stalled when it shouldn't be.
**Fix needed:** After successful contract ingestion, check if there's an active onboarding run for the same entity and auto-advance the relevant step.

### 6. No cap tracking for campaigns (MEDIUM-HIGH)
**Impact:** VAs cannot know if a campaign needs more traffic or is saturated. They might recruit publishers for a campaign that's already at cap. Or miss a campaign that desperately needs volume.
**Fix needed:** Campaign table needs a `daily_cap`, `remaining_cap`, or `cap_status` field. Campaign matching should factor in capacity.

### 7. Chat history is IP-based, not user-based (MEDIUM)
**File:** `/api/ask/history` L18-20
**Impact:** VA switches from office WiFi to VPN, loses all context. VA shares a computer -- sees the other VA's history. Two VAs on the same network IP see each other's conversations.
**Fix needed:** History should be keyed on authenticated `user_id` (available from auth context), not IP hash.

### 8. No pricing guardrail on `/api/ask/draft-outreach` route (MEDIUM)
**File:** `src/app/api/ask/draft-outreach/route.ts` L14-37
**Impact:** The simpler outreach API (used by EntityVetCard's "Draft outreach" button) says "Be specific about TLP's value" and mentions "payout rates." It has no explicit "never reveal pricing" instruction. Could leak rate info in outreach to entities who should only see generic value propositions.
**Fix needed:** Add "NEVER include specific dollar amounts, payout rates, or pricing in any draft" to the prompt.

### 9. Legacy unsigned onboarding tokens still accepted (MEDIUM)
**File:** `src/lib/onboarding-token.ts` L177-220
**Impact:** Old tokens with no expiry and no signature are honored. If a legacy token is leaked, anyone can use it indefinitely.
**Fix needed:** Set a deprecation date. After that date, reject legacy tokens. Or at minimum, force a short expiry on legacy-decoded tokens.

### 10. No unified stalled-items view (MEDIUM)
**Impact:** Stalled onboardings show in OnboardingChecklist (collapsed). Stalled outreach shows in morning briefing. Stalled pipeline entities show on Kanban with time badges. A VA must check 3 places to know what's stuck.
**Fix needed:** A "Needs Attention" widget or pill that aggregates: stalled onboardings + overdue follow-ups + aging pipeline entities.

---

## Outreach & Sales Guardrails Assessment

### What exists:
- **Banned phrases:** 19 phrases banned in draft-first-outreach tool (BANNED_PHRASES constant)
- **Character limits:** 300 for connection requests, 500 for DMs, hard 500 global cap
- **Personalization check:** Model must confirm each version is personalized
- **Classification-aware asks:** Different soft-ask patterns for BUYER vs PUBLISHER vs BROKER
- **Persuasion framework:** Identity-based persuasion with opt-out ("too salesy" override)
- **Anti-fabrication:** "NEVER fabricate traits, use guilt/shame/fear, or lie about scarcity"
- **Audit trail:** Outreach drafts logged to ai_action_log, copies logged to outreach-log

### What's missing:
- **No explicit "never reveal pricing" on the simpler draft-outreach API path**
- **No HORMONES or other named sales framework** -- custom Hook/Bridge/Soft Ask only
- **No A/B tracking** -- drafts are generated but there's no way to know which version performed better
- **No template library** -- every draft is generated fresh. No ability to save and reuse successful templates
- **No compliance review step** -- drafts go from Milo directly to VA clipboard. No manager approval gate for first-time outreach to sensitive entities
- **handle-objection tool has access to real rates** and could leak them in a response draft

---

## Proactive Milo Capabilities

### What exists:
| Capability | Mechanism | Frequency |
|---|---|---|
| Publisher/buyer discovery | publisher-discovery cron | Daily 9 AM MT |
| Morning task assignments | morning-assignments cron | Daily 8 AM MT |
| Onboarding stall detection | onboarding-stall-check cron | Daily 7 AM MT |
| Stale relationship alerts | stale-relationships cron | Daily 8 AM MT |
| LinkedIn draft pre-generation | linkedin-draft cron | Weekly Monday 10 AM MT |
| Dispute detection | detect-disputes cron | Hourly :15 |
| Error digest | error-digest cron | Daily 8 AM MT |
| Health monitoring | health-check cron | Hourly :00 |
| Call data sync | sync cron | Every 15 min |
| Alert evaluation | evaluate cron | Every 5 min |

### What's needed but missing:
- **Push notifications inside the app** -- Milo only surfaces proactive insights when the VA loads a page or asks in chat
- **Proactive chat messages** -- "Hey, Publisher X hasn't responded in 5 days, want me to draft a follow-up?" should appear in chat without the VA asking
- **Campaign gap alerts** -- "Campaign Y has no publishers and is uncapped -- want me to find some?"
- **Cap monitoring** -- "Campaign Z hit 90% of daily cap, should I pause publisher onboarding?"
- **Performance anomaly notifications** -- The prediction engine exists but anomalies only appear in morning briefing, not as real-time alerts
- **Cross-entity intelligence** -- "Publisher A runs the same verticals as Publisher B who just churned -- should I reach out?"

---

## Campaign Awareness Gaps

1. **No campaign list in system prompt context** -- Milo knows campaigns exist but doesn't have the active list in every conversation. Must query on demand.
2. **No cap data anywhere** -- Cannot distinguish campaigns needing traffic from saturated ones.
3. **No "find publishers for X campaign" on-demand tool** -- Only the daily cron discovers new entities. VA cannot trigger discovery in real-time.
4. **No campaign health dashboard** -- No single view showing: campaign name, active publishers count, daily volume, cap utilization, buyer rate, margin.
5. **Campaign matching only matches EXISTING publishers** -- `matchPublishersToCampaign` only looks at publishers already active on other campaigns. Doesn't consider pipeline prospects.
6. **No campaign-to-outreach automation** -- When a new campaign goes live, there's no trigger to find publishers and start outreach. A human must notice and ask.

---

Report at: https://raw.githubusercontent.com/moscaleflow/conductor-mark-public/main/reports/ppc-stress-test-audit-2026-05-06.md
