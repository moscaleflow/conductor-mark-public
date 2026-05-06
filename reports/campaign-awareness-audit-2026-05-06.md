# Campaign Awareness & Proactive Milo Behavior Audit

**Date:** 2026-05-06
**Repo:** milo-for-ppc
**Auditor:** Coder-3 (read-only)

---

## Section 1: Campaign Data Model

### Schema (campaigns table)

The campaigns table is queried via `/api/campaigns/route.ts` with these columns:

| Column | Purpose |
|--------|---------|
| `id` | UUID primary key |
| `name` | Full campaign name (from TrackDrive) |
| `vertical` | Inferred vertical (ssdi, mva, aca, medicare, fe, debt, solar, etc.) |
| `td_offer_id` | TrackDrive offer ID (foreign key to TD) |
| `billing_type` | CPA, RTB, CPL, CPQL |
| `publisher_rate` | Rate paid to publisher |
| `buyer_rate` | Rate charged to buyer |
| `duration_threshold_seconds` | Conversion duration threshold |
| `status` | Campaign status (active, paused, etc.) |
| `convoqc_enabled` | Whether ConvoQC analysis runs on this campaign's calls |
| `td_campaign_name` | Original TrackDrive name |
| `display_name` | Cleaned display name (strips "X - X" TD naming pattern) |
| `created_at` | Timestamp |
| `notes` | Free-text notes (editable via PATCH) |

### What IS tracked
- Campaign name, vertical, billing type
- Publisher and buyer rates
- Basic status (active/paused)
- ConvoQC enablement per campaign
- TrackDrive offer ID linkage

### What is NOT tracked
- **Daily cap / monthly cap**: No cap columns exist on the campaigns table. Cap data lives only in TrackDrive's buyer-level fields (connection_daily_limit, attempt_daily_limit, buyer_conversion_daily_limit, etc.) and is read from TD snapshots, not stored on campaigns.
- **Current fill vs cap**: No fill tracking. Milo cannot answer "how full is this campaign today?"
- **Campaign-level revenue today**: Not pre-aggregated. Must be computed from call_records on the fly.
- **Uncapped status**: There is no "capped" or "uncapped" status field. Caps are per-buyer, not per-campaign.

### TrackDrive Sync

`/api/sync/campaigns/route.ts` syncs campaigns from TrackDrive via `getOffers()`:
- Fetches all TD offers
- Upserts into campaigns table (matched by td_offer_id)
- Auto-infers vertical from offer name using heuristic keyword matching
- Sets status to 'active' for all synced offers
- Does NOT sync cap data, buyer assignments, or fill levels

`/api/campaigns/auto-detect/route.ts` runs three enrichment passes:
1. Vertical detection from campaign name
2. ConvoQC coverage detection from call_records
3. Display name generation

---

## Section 2: Milo's Campaign Awareness

### System Prompt Analysis

The `/api/ask/route.ts` system prompt is built from `@milo/voice` pill prompts (VET_PROMPT, SALES_PROMPT, PUBLISHER_PROMPT, BUYER_PROMPT). The system prompt does NOT include:
- A list of active campaigns
- Campaign cap status
- Which campaigns need publishers
- Current fill levels

Campaign context reaches Milo only through:
1. **CRM context injection** (`findCrmContext`): When a VA asks about an entity, Milo looks up that entity in crm_counterparties and entity_knowledge. If the entity is a buyer, Milo gets buyer health, alerts, dispute summaries, and pipeline stage -- but NOT the campaigns that buyer is on.
2. **Live data queries** (`milo-queries.ts`): Milo can pull buyer profiles, publisher profiles, vertical aggregates, entity alerts, invoice details, and pipeline stages.
3. **Entity knowledge** (hot_knowledge/warm_knowledge from knowledge engine): AI-written narrative about each entity, updated via the evaluate cron.

### Can Milo Answer "What campaigns are uncapped today?"

**No.** There is no mechanism for this:
- Caps are buyer-level in TrackDrive, not campaign-level in Milo
- No cap/fill data is synced to Supabase
- No query exists that computes "campaigns with available buyer capacity"
- The ask route does not inject campaign roster context

### Can Milo Answer "Which campaigns need publishers?"

**No.** Milo has no concept of "campaigns that need more traffic." This would require:
- Knowing which campaigns have buyers with unfilled caps
- Knowing current daily fill vs cap
- Matching available publisher verticals against buyer demand

---

## Section 3: Proactive Cron Inventory

16 cron jobs registered in `vercel.json`:

| # | Path | Schedule | Description | Proactive? |
|---|------|----------|-------------|------------|
| 1 | `/api/cron/sync` | Every 15 min | Core data sync: calls, payouts, ConvoQC, campaigns from TrackDrive. Direct import of runSync(). Triggers background quality enforcement + memory learning via waitUntil. Also syncs pings. | No (data pipeline) |
| 2 | `/api/cron/reconciliation` | Daily 13:00 UTC (6am Denver) | Revenue/payout reconciliation against TrackDrive | No (data validation) |
| 3 | `/api/cron/morning-assignments` | Daily 14:00 UTC (7am Denver) | Sends personalized morning task lists to each VA via Teams DMs. Calls `/api/morning-assignments`. | Yes -- daily task push |
| 4 | `/api/cron/daily-validation` | Daily 13:00 UTC (6am Denver) | Runs 20 synthetic tests; blocks morning briefing if critical failures. Runs 1 hour before morning-assignments. | Yes -- pre-briefing quality gate |
| 5 | `/api/cron/health-check` | Hourly | 7 parallel health checks on services; sends Teams alerts when degraded | Yes -- system monitoring |
| 6 | `/api/cron/billing-prepare` | Weekdays 13:00 UTC (6am Denver) | Prepares billing/invoicing data | No (financial ops) |
| 7 | `/api/cron/td-change-detection` | Every 15 min | Monitors TrackDrive change logs: buyer bid changes, pause/unpause, conversion config, cap changes, DID reassignments, call unconversions. Takes hourly buyer snapshots. Compares against agreed_terms for contract mismatches. Triggers EOD report when all buyers close. | Yes -- real-time TD monitoring, contract mismatch detection, EOD trigger |
| 8 | `/api/cron/evaluate` | Every 5 min | Knowledge engine: evaluates call records and events via Opus LLM. Detects anomalies, updates entity knowledge. Also runs daily aggregate checks (ping rejection, idle publishers, overdue invoices, repeat callers). | Yes -- AI anomaly detection + 4 daily aggregate checks |
| 9 | `/api/cron/detect-disputes` | Hourly at :15 | Detects billing disputes from data patterns | Yes -- dispute early warning |
| 10 | `/api/cron/mediarite-xref` | Daily 14:00 UTC (7am Denver) | Cross-references converted CIDs against MediaRite data to find missing records | Yes -- revenue leak detection |
| 11 | `/api/cron/demo-cleanup` | Hourly | Cleans up demo/test data | No (housekeeping) |
| 12 | `/api/cron/stale-relationships` | Daily 14:00 UTC (7am Denver) | Scans crm_counterparties for entities not contacted within threshold (publishers: 14d, buyers: 21d, pipeline: 7d). Creates alerts + auto-tasks. Auto-resolves cleared alerts. | Yes -- follow-up enforcement |
| 13 | `/api/cron/error-digest` | Daily 14:00 UTC (7am Denver) | Posts categorized error digest to Teams #operations. Shows recurring issues and new error patterns. | Yes -- error pattern alerting |
| 14 | `/api/cron/publisher-discovery` | Daily 15:00 UTC (8am Denver) | Uses Claude with web_search to discover 3-5 publishers/buyers per vertical. Cross-refs against CRM + pipeline + blacklist. Inserts net-new into prospect_pipeline. Creates opportunity alerts. Max 3 verticals/day. | Yes -- proactive publisher finding |
| 15 | `/api/cron/linkedin-draft` | Weekly Monday 16:00 UTC (9am Denver) | Generates LinkedIn recruitment post drafts for Tiffani. One per vertical. Never auto-published. | Yes -- recruitment content |
| 16 | `/api/cron/onboarding-stall-check` | Daily 13:00 UTC (6am Denver) | Finds onboarding steps stuck 72+ hours. Creates action_items per stalled step with dedup. | Yes -- onboarding follow-up |

**Not in vercel.json but exists:**
- `/api/cron/stale-waiting/route.ts`: Checks for "waiting_on_reply" alerts stale 5+ days and spawns follow-up step alerts. Not scheduled in vercel.json -- may be invoked externally or not yet activated.
- `/api/cron/partner-sync/route.ts`: Syncs TrackDrive traffic sources and buyers to prospect_pipeline. Not scheduled in vercel.json.

---

## Section 4: Alert Generation Logic

### Primary Alert Generator: `/api/alerts/generate/route.ts`

Called as part of the sync pipeline (triggered by `/api/cron/sync`). Generates and auto-resolves alerts:

**Alert Types Generated:**

| # | Alert | Type | Trigger Condition |
|---|-------|------|-------------------|
| 1 | Converted calls with $0 payout | Critical | CPA/CPL/CPQL calls with is_converted=true, payout=0, payout_corrected=false in 24h |
| 2 | Buyer line not answering | Critical | 3+ unanswered calls to same buyer in 1 hour (grouped) |
| 3 | Quality flag - [Publisher] | Warning | Publisher with >25% ConvoQC flag rate on 10+ calls in 24h |
| 4 | Call volume down X% | Warning | Today's calls < 60% of same time last week |
| 5 | Publisher capacity available | Opportunity | Publisher today < 30% of 7-day daily average (business hours only, 8am-8pm Denver) |

**Auto-Resolution Conditions:**
- Buyer line not answering: resolves when no buyer has 3+ unanswered calls in 1h
- Publisher capacity: resolves when all publishers above 50% daily average
- Call volume down: resolves when today >= 70% of last week same time
- Campaign revenue disappeared: resolves when all flagged campaigns show revenue
- Quality flag: resolves when flag rate drops below 25%
- Publisher grade dropped: resolves when grade drop < 2 levels
- RTB bids declining: resolves when decline < 15%
- Buyer health critical/declining: resolves when score recovers
- Getting outbid: resolves when acceptance rate drop < 20%
- Ping waste: resolves when volume < 1000 or acceptance >= 5%
- Revenue mismatch / payout below: auto-resolves after 48h
- Invoice overdue: resolves when no overdue invoices remain for entity

### Secondary Alert Sources (from evaluate cron):

| Alert | Type | Source |
|-------|------|--------|
| Ping rejection high (>50%) | Warning | runPingRejectionCheck() - daily |
| Idle publisher (48h no calls despite active routes) | Warning | runIdlePublisherCheck() - daily |
| Invoice overdue (auto-transition + alert) | Warning/Critical | runOverdueInvoiceCheck() - daily |
| Repeat caller (5+ calls in 30 days) | Warning | runRepeatCallerCheck() - daily |
| Bid floor violation (revenue < IO floor) | Critical | Per-event in evaluate loop |
| Evaluation queue backlog | Critical | When 1h+ unprocessed items |
| Opus truncation | Warning | When knowledge engine parse fails |
| Entity anomalies | Various | LLM-detected anomalies via processAnomalies() |

### TD Change Detection Alerts:

| Alert | Type | Trigger |
|-------|------|---------|
| Buyer bid changed | Warning | bid_price change in TD |
| Buyer paused/unpaused | Warning | paused field change |
| Buyer conversion rate changed | Critical | current_conversion_revenue change |
| Buyer conversion duration changed | Critical | current_conversion_duration change |
| Buyer cap changed | Warning | Any cap limit field change |
| Buyer business hours changed | Warning | business_hours_schedule change |
| Campaign paused/unpaused | Warning | Offer paused field change |
| Publisher paused/unpaused | Warning | TrafficSource paused field change |
| DID reassigned/paused/deleted | Warning/Critical | PhoneNumber changes |
| Call unconverted | Critical | buyer_converted set to non-Converted (grouped) |
| Contract mismatch | Critical | Live TD values differ from agreed_terms |

### Alert Assignment

All alerts use `resolveAlertTargetUser()` for per-operator routing:
- Publisher alerts -> Jackie
- Buyer alerts -> Fab
- DID alerts -> Malvin
- Invoice AR -> Jen
- Invoice AP -> Tiffani
- System alerts -> Mark
- Discovery -> Jackie (publishers) or Fab (buyers)

---

## Section 5: Gaps in Proactive Behavior

### Gap 1: No Cap/Fill Awareness (CRITICAL)

Milo does not know:
- Which campaigns have daily/monthly caps
- How full those caps are today
- Which campaigns are "uncapped" (unlimited buyer demand)
- Which campaigns are about to hit cap and need traffic diverted

**Impact:** Mark cannot ask Milo "what campaigns are uncapped today?" or "where should we send more traffic?" This is the foundational gap for proactive publisher matching.

**Required:** Sync buyer cap data (daily_cap, monthly_cap, current fill counts) from TrackDrive to a campaign_capacity table or enrich existing campaigns table. Compute fill % on each sync tick.

### Gap 2: No Campaign-to-Publisher Demand Matching

There is no system that:
- Maps campaigns to their current publisher roster
- Identifies campaigns with fewer publishers than needed
- Matches available publisher verticals against buyer demand

**Impact:** The publisher-discovery cron finds new publishers by vertical, but doesn't prioritize based on which campaigns actually need traffic.

### Gap 3: No "VA Follow-Up on Vet Results" Tracking

The vet -> outreach -> onboard pipeline has gaps:
- Milo performs vets when asked, but there is no automated check that the VA acted on the vet result
- If a vet comes back clean, there is no reminder to draft outreach
- If outreach was sent, there is no automated follow-up if no reply within X days (stale-waiting cron exists but is not in vercel.json)
- The pipeline board (prospect_pipeline) tracks stage but doesn't auto-advance or auto-remind

**Impact:** Vet results fall through the cracks. A VA may vet an entity, see "clean," then get busy and forget to do outreach.

### Gap 4: No Morning Briefing Campaign Summary

The morning briefing (`/api/briefing/route.ts`) includes:
- Yesterday/today call stats, revenue, margin
- Active alerts (critical, warning, opportunity)
- Invoice status
- RTB/ping summaries

It does NOT include:
- Campaign roster summary ("36 active campaigns, 12 at cap, 8 uncapped")
- "Campaigns needing attention" section
- Publisher-to-campaign matching recommendations

### Gap 5: No Automated "Suggest Publisher Outreach" for Uncapped Campaigns

The publisher-discovery cron discovers entities by vertical but does not:
- Check which campaigns are uncapped and accepting new publishers
- Prioritize discovery for verticals where caps have room
- Suggest specific existing pipeline publishers who could fill uncapped campaigns

### Gap 6: stale-waiting Cron Not Scheduled

`/api/cron/stale-waiting/route.ts` exists and works (checks for 5+ day stale "waiting_on_reply" alerts, spawns follow-up steps). But it is NOT in `vercel.json` crons, so it never runs automatically.

### Gap 7: No Proactive EOD Summary Push

The EOD system exists (`/api/eod/compose-business`, `/api/eod/generate`, `/api/eod/persist-and-send`) and is triggered when all buyers close. But there is no separate cron ensuring EOD reports are generated if the trigger misses (e.g., if td-change-detection misses the closing window).

### Gap 8: No Milo Context Injection for Campaigns

When a VA asks Milo about a campaign, the system prompt does not include:
- The campaign's current vertical, billing type, rates
- Active publishers on that campaign
- Fill vs cap status
- Recent performance metrics

---

## Section 6: Follow-Up Tracking Gaps

### Vet -> Outreach -> Onboard Pipeline

**What exists:**
- prospect_pipeline table with stages: qualifying -> vetted -> outreach -> onboarding -> active/dormant/blacklisted
- Pipeline API (`/api/pipeline`) with stage filtering and days-in-stage tracking
- Onboarding stall check (cron, daily, catches steps stuck 72+ hours)
- Stale relationship alerts (14d publishers, 21d buyers, 7d pipeline entities)

**What falls through the cracks:**

1. **Vet-to-action gap**: No automated reminder when a vet completes and the entity is clean but no outreach has been drafted/sent within 48h.

2. **Outreach-to-reply gap**: The stale-waiting cron (which would catch this) is not scheduled. An outreach can be sent and the "waiting_on_reply" state can sit indefinitely without automated escalation.

3. **No outreach tracking in pipeline**: The prospect_pipeline tracks stage but does not record when outreach was sent, how many times, or whether replies were received. That data lives in crm_counterparties.last_contact_at but is not connected to pipeline stage advancement logic.

4. **No pipeline stage auto-advancement**: Stages are manually advanced. If a publisher signs an MSA but nobody clicks "advance," the pipeline shows them stuck in the previous stage.

5. **No "days since vet" metric**: The operator cannot quickly see "these 15 entities were vetted 3+ days ago and nobody has done outreach."

---

## Section 7: Recommended Proactive Features (Prioritized)

### Priority 1: Campaign Cap/Fill Awareness (HIGH IMPACT)

**What:** Sync buyer cap data from TrackDrive on every 15-min sync tick. Store daily_cap, monthly_cap, current_fill, pct_filled on a campaign_buyers or campaign_capacity table. Expose via `/api/campaigns/capacity`.

**Why:** Foundation for all demand-aware features. Without this, Milo cannot answer cap questions, cannot prioritize publisher discovery, and cannot generate "uncapped campaigns" alerts.

**Estimated scope:** Medium. Requires new table, sync logic addition, API endpoint.

### Priority 2: Schedule stale-waiting Cron (QUICK WIN)

**What:** Add `/api/cron/stale-waiting` to vercel.json crons with a daily schedule (e.g., `0 14 * * *`).

**Why:** The code is written, tested, and ready. It just needs to be activated. Immediately closes the "outreach sent but no follow-up" gap.

**Estimated scope:** One-line change in vercel.json.

### Priority 3: Vet-to-Outreach Reminder Cron (HIGH IMPACT)

**What:** New cron that queries prospect_pipeline for entities in "vetted" stage where stage_changed_at > 48h ago and no outreach log exists. Creates action_item for the assigned VA.

**Why:** Closes the most common drop-off point: entity vetted clean, VA gets busy, nobody follows up.

**Estimated scope:** Small. New cron route, follows existing action_items + pipeline patterns.

### Priority 4: Campaign Context Injection for Ask Route (MEDIUM IMPACT)

**What:** When a VA asks about a campaign by name, inject campaign metadata (vertical, billing_type, rates, active publishers, recent call count) into the system prompt.

**Why:** VAs ask Milo about campaigns regularly. Right now Milo has no campaign data in context and gives generic responses.

**Estimated scope:** Medium. Requires campaign lookup in ask route, context building, prompt injection.

### Priority 5: Morning Briefing Campaign Summary (MEDIUM IMPACT)

**What:** Add a "Campaign Health" section to the briefing: total active, at-cap count (once cap data exists), uncapped count, campaigns with no calls in 24h.

**Why:** Mark's stated goal: "know what campaigns are running, what's capped vs uncapped." The briefing is the primary daily touchpoint.

**Estimated scope:** Medium. Requires Priority 1 (cap data) first.

### Priority 6: Demand-Aware Publisher Discovery (MEDIUM IMPACT)

**What:** Modify publisher-discovery cron to prioritize verticals where uncapped campaigns exist (once cap data is available). Score verticals by "unfilled daily buyer demand in dollars."

**Why:** Currently discovery picks up to 3 verticals arbitrarily. Demand-aware selection means discoveries are immediately actionable.

**Estimated scope:** Small modification to existing cron, but requires Priority 1.

### Priority 7: Pipeline Outreach Tracking (LOWER PRIORITY)

**What:** Add outreach_sent_at, outreach_count, last_reply_at columns to prospect_pipeline. Connect pipeline stage logic to outreach events.

**Why:** Currently pipeline and outreach tracking are separate. Operators must manually cross-reference. Connecting them enables automated "this entity was sent 3 messages with no reply -- escalate or archive."

**Estimated scope:** Medium. Schema change, outreach route integration, pipeline advancement logic.

### Priority 8: Daily Uncapped Campaign Alert (LOWER PRIORITY, NEEDS P1)

**What:** New daily cron that identifies campaigns where total buyer cap > today's projected fill and creates an "opportunity" alert: "These 5 campaigns have $X in unfilled buyer demand today. Consider prioritizing publisher outreach for [verticals]."

**Why:** The most directly actionable version of "proactively look for publishers for uncapped campaigns." Requires cap/fill data.

**Estimated scope:** Small cron, but requires Priority 1.

---

Report at: https://raw.githubusercontent.com/moscaleflow/conductor-mark-public/main/reports/campaign-awareness-audit-2026-05-06.md
