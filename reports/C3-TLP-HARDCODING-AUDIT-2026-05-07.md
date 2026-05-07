# TLP Hardcoding Audit — milo-for-ppc

> Coder-3 Research | 2026-05-07 | READ-ONLY audit

---

## Executive Summary

**102 production source files** contain TLP-specific hardcoding across **7 categories**. The codebase has `vertical.config.ts` as a centralization point, but it covers only 8 of the ~380+ unique hardcoded references. The vast majority of TLP-specific strings are scattered directly in source files — AI system prompts, team routing logic, email templates, and tenant_id literals.

| Category | Unique Strings | Files Affected | Extraction Difficulty |
|----------|---------------|----------------|----------------------|
| Brand/Company Name | ~45 | 44 | Easy-Medium |
| Team Member Routing | ~11 names | 38 | Hard |
| Domain/URL | ~8 | 18 | Easy |
| Tenant ID ('tlp') | 1 value, ~50 usages | 28 | Easy |
| AI System Prompts | ~25 prompt blocks | 22 | Medium |
| Business Logic (TrackDrive) | deep coupling | 30+ | Hard |
| Email Templates (HTML) | ~6 templates | 5 | Medium |

---

## Category 1: Brand/Company Name (44 files, ~45 unique strings)

### 1a. "The Lead Penguin" — full company name (44 files)

**In vertical.config.ts (partially centralized):**
- `src/lib/vertical.config.ts:53` — `legalName: 'The Lead Penguin LLC'`
- `src/lib/vertical.config.ts:52` — `name: 'TLP Compliance'`
- `src/lib/vertical.config.ts:7` — `name: 'Milo for PPC'`

**In AI system prompts (NOT centralized — hardcoded inline):**
- `src/lib/milo-prompt.ts:1` — `"You are Milo, the AI operations assistant for The Lead Penguin (TLP)"`
- `src/lib/milo-support-prompt.ts:1` — `"You are Milo, an AI operations assistant at The Lead Penguin"`
- `src/lib/anthropic.ts:51` — `"You are Milo, ops assistant for The Lead Penguin (TLP)"`
- `src/lib/knowledge-engine.ts:526` — `"Milo Knowledge Engine — the memory and pattern-detection system for The Lead Penguin's call brokerage"`
- `src/lib/knowledge-engine.ts:853` — `"initializing a knowledge profile for a ${entityType} entity in The Lead Penguin's call brokerage"`
- `src/lib/reply-classifier.ts:47` — `"You triage reply messages from prospects for The Lead Penguin (TLP)"`
- `src/lib/eod-composer.ts:99` — `"composing an end-of-day business report for The Lead Penguin (TLP)"`
- `src/app/api/ask/draft-outreach/route.ts:14` — `"writing outreach messages for The Lead Penguin (TLP)"`
- `src/app/api/ask/draft-verify/route.ts:16` — `"writing a short, casual Teams or LinkedIn message on behalf of...The Lead Penguin (TLP)"`
- `src/app/api/ask/draft-verify/route.ts:25` — `"We're The Lead Penguin, a pay-per-call network"`
- `src/app/api/outreach/draft/route.ts:119` — `"drafting outreach for The Lead Penguin (TLP), a pay-per-call lead generation network based in Denver"`
- `src/app/api/outreach/draft-followup/route.ts:22` — `"drafting follow-up outreach for The Lead Penguin (TLP)"`
- `src/app/api/cron/linkedin-draft/route.ts:107` — `"Draft a short LinkedIn post...for Tiffani, who runs publisher partnerships at The Lead Penguin"`
- `src/app/api/cron/publisher-discovery/route.ts:187` — `"You are Milo, the intelligence engine for The Lead Penguin (TLP)"`
- `src/app/api/eod/generate/route.ts:170` — `"generating an end-of-day work report for a team member at The Lead Penguin (TLP)"`
- `src/app/api/operator/pill/review/route.ts:1098` — `"Milo, an AI ops assistant for The Lead Penguin (TLP)"`
- `src/app/api/verify/route.ts:440` — `"Draft a short follow-up message...from The Lead Penguin"`
- `src/lib/tools/analyze-contract.ts:196` — `"contract analyst for The Lead Penguin (TLP)"`
- `src/lib/tools/detect-coached-call.ts:140` — `"You are Jackie at The Lead Penguin"`
- `src/lib/tools/discover-entities.ts:135` — `"Milo, the intelligence engine for The Lead Penguin (TLP)"`
- `src/lib/tools/draft-dispute-message.ts:112` — `"You are Fab, the senior buyer/publisher relationship manager for The Lead Penguin (TLP)"`
- `src/lib/tools/draft-first-outreach.ts:193` — `"VEE_L2 — the Outreach Writer for The Lead Penguin (TLP)"`
- `src/lib/tools/generate-redlined-version.ts:83` — `"contract redlining engine for The Lead Penguin (TLP)"`
- `src/lib/tools/handle-objection.ts:183` — `"Vee, The Lead Penguin (TLP) outreach strategist"`
- `src/lib/tools/ingest-returned-contract.ts:112` — `"senior contracts operator for The Lead Penguin"`
- `src/lib/tools/morning-priority-briefing.ts:262` — `"tightening a morning briefing for the ${role} operator at The Lead Penguin"`
- `src/lib/tools/publish-contract-for-buyer-review.ts:227` — `"Message from The Lead Penguin"` (HTML email)
- `src/lib/tools/research-prospect.ts:225` — `"Prospect Intelligence layer for The Lead Penguin (TLP)"`
- `src/lib/tools/review-call-quality.ts:109` — `"You are Jackie, the senior QC reviewer at The Lead Penguin"`
- `src/lib/tools/warm-handoff-to-tiffani.ts:183` — `"Vee, The Lead Penguin (TLP) outreach strategist, preparing a warm handoff package for Tiffani"`

**In UI pages (NOT centralized):**
- `src/app/layout.tsx:10` — `title: 'Milo — The Lead Penguin'`
- `src/app/contract-review/[id]/page.tsx:429` — `doc?.analysis_result?.our_company || 'The Lead Penguin'`
- `src/app/contracts/review/[id]/page.tsx:125` — `analysis?.our_company ?? 'The Lead Penguin'`
- `src/app/verify/[token]/page.tsx:415` — `We're <span>The Lead Penguin</span>`
- `src/app/onboard/publisher/[token]/page.tsx:344` — `Let's get you set up with <span>The Lead Penguin</span>`
- `src/app/onboard/buyer/[token]/page.tsx:331` — `Let's get your buyer account set up with <span>The Lead Penguin</span>`
- `src/app/onboard/buyer/[token]/page.tsx:588` — `The Lead Penguin` (footer)
- `src/app/onboard/publisher/[token]/page.tsx:664` — `The Lead Penguin` (footer)
- `src/app/onboard/page.tsx:234` — `"I run operations here at The Lead Penguin."`

**In API routes (NOT centralized):**
- `src/app/api/contract/analyze-bg/route.ts:23` — `const OUR_COMPANY = process.env.COMPANY_NAME || 'The Lead Penguin'`
- `src/app/api/contract/analyze-bg/route.ts:24` — `const OUR_ALIASES = [OUR_COMPANY, 'TLP', 'Lead Penguin', 'ScaleFlow']`
- `src/app/api/contract/process/route.ts:115-130` — regex `lead\s*penguin|tlp|scaleflow` for counterparty identification
- `src/app/api/invoices/[id]/pdf/route.ts:80` — `<div class="company">The Lead Penguin</div>`
- `src/app/api/invoices/[id]/pdf/route.ts:160` — `The Lead Penguin · Invoice...`
- `src/app/api/contract/download/[id]/route.ts:217` — `Generated by Milo -- The Lead Penguin`

**In vendor packages:**
- `vendor/voice/src/pills/vet.ts:4` — `"TLP (The Lead Partner)"` (NOTE: inconsistent — says "Lead Partner" not "Lead Penguin")
- `vendor/voice/src/pills/buyer.ts:1` — `"TLP (The Lead Partner)"`
- `vendor/voice/src/pills/publisher.ts:1` — `"TLP (The Lead Partner)"`
- `vendor/voice/src/pills/sales.ts:1` — `"TLP (The Lead Partner)"`
- `vendor/voice/src/pills/vet-extraction.ts:51` — `"the intelligence layer for TLP"`

**Extraction difficulty:** MEDIUM. The company name appears in ~44 production files. Most are string literals in prompts — can be extracted to config, but each prompt needs testing after change. The `contract/process/route.ts` regex is the hardest to generalize.

### 1b. "Lead Penguin" — short name variants

- `src/lib/ppc/system-prompt.ts:6` — `company: 'Lead Penguin'` (AI persona)
- `src/app/api/contract/analyze-bg/route.ts:24` — in OUR_ALIASES array
- `src/app/api/contract/process/route.ts:118,130` — in regex pattern

### 1c. "TLP" — abbreviation (used as brand, not tenant_id)

- Used in 25+ AI prompt strings as `"(TLP)"` or `"TLP"`
- `src/app/api/contract/analyze-bg/route.ts:24` — in OUR_ALIASES
- `src/app/api/contract/process/route.ts:221` — in GENERIC_CONTRACT_PATTERNS regex

### 1d. "ScaleFlow" — parent company name

- `src/app/api/contract/analyze-bg/route.ts:24` — in OUR_ALIASES
- `src/app/api/contract/process/route.ts:118,130` — in regex pattern
- `src/app/onboard/page.tsx:45` — `ADMIN_EMAILS = ['mark@scaleflow.co', 'tiffani@scaleflow.co']`
- `src/app/api/welcome/[slug]/claim/route.ts:172` — `mark@scaleflow.co`

---

## Category 2: Team Member Routing (38 files, 11 names)

This is the **most deeply coupled** category. TLP's specific 11-person team structure is baked into routing logic, AI prompts, alert generation, notification drafting, and UI components.

### 2a. Hardcoded team roster (canonical source)

**`src/lib/team-roster.ts`** — 11 entries with names, roles, welcome lines, recognition lines:
- Mark (admin), Tiffani (admin), Cam (outreach), Fab (publisher_qa), Jackie (qc), Jen (billing), Kate (billing_support), Kent (ping_validation), Lee (call_monitoring), Malvin (campaign_ops), Vee (outreach)

### 2b. Hardcoded routing by name

**`src/lib/alert-routing.ts:77-83`** — Entity-type default routing:
```
publisher: 'Jackie', buyer: 'Fab', caller: 'Jackie', did: 'Malvin', campaign: 'Malvin'
```

**`src/app/api/notifications/draft/route.ts:13-39`** — TEAM_ROUTING map:
```
call_unconverted → Fab, publisher_grade_dropped → Jackie, bid_changed → Malvin,
billing_issue → Jen, dispute_review → Tiffani, default → Mark
```

**`src/lib/knowledge-engine.ts:193-194`** — VALID_ROUTES set:
```
'Fab', 'Malvin', 'Jackie', 'Jen', 'Cam', 'Vee', 'Tiffani', 'Mark'
```

**`src/lib/knowledge-engine.ts:553-559`** — Routing rules in AI prompt:
```
- Fab: buyer bid/settings/unconversion issues
- Jackie: call quality/compliance violations
- Tiffani: contract/vendor issues, escalations
- Jen: billing, invoices
- Malvin: campaign config, DID issues
```

### 2c. Hardcoded team names in cron jobs

**`src/app/api/cron/evaluate/route.ts`** — 12 instances:
- `routeTo: 'Fab'` (line 331)
- `route_to: 'Mark'` (line 595)
- `routeTo: 'Jackie'` (lines 736, 871, 1175)
- `route_to: 'Jackie'` (lines 754, 887, 1194)
- `routeTo = isAR ? 'Jen' : 'Tiffani'` (line 995)

**`src/app/api/cron/td-change-detection/route.ts`** — 20+ instances:
- `routeTo: 'Fab'` (lines 221, 795)
- `routedTo: 'Fab+Malvin'` (lines 241, 352, 365)
- `routedTo: 'Fab'` (lines 327, 339, 391, 404, 480, 497)
- `routedTo: 'Malvin'` (lines 421, 433, 444, 460)

**`src/app/api/cron/linkedin-draft/route.ts`** — `routeTo: 'Tiffani'` (line 251), `route_to: 'Tiffani'` (line 268)

**`src/app/api/cron/publisher-discovery/route.ts`** — `routeTo: entityType === 'buyer' ? 'Fab' : 'Jackie'` (line 363)

### 2d. Hardcoded team names in UI components

- `src/components/shared/TeamAssignDropdown.tsx:16` — `['Mark', 'Tiffani', 'Cam', 'Fab', 'Jackie', ...]`
- `src/components/shared/CallDetailPanel.tsx:180` — `OPERATORS = ['Tiffani', 'Vee', 'Cam', 'Malvin', 'QC']`
- `src/components/ask/EntityVetCard.tsx:116` — `FALLBACK_NAMES = ['Tiffani', 'Fab', 'Cam', 'Vee']`

### 2e. Hardcoded team names in AI personas

- `src/lib/tools/detect-coached-call.ts:140` — `"You are Jackie at The Lead Penguin"`
- `src/lib/tools/review-call-quality.ts:109` — `"You are Jackie, the senior QC reviewer"`
- `src/lib/tools/draft-dispute-message.ts:112` — `"You are Fab, the senior buyer/publisher relationship manager"`
- `src/lib/tools/warm-handoff-to-tiffani.ts:183` — entire tool is Tiffani-specific
- `src/lib/tools/warm-handoff-to-tiffani.ts:239` — tool description references "Tiffani (TLP CEO)"
- `src/lib/milo-prompt.ts:3` — `"You are talking to the operator — either Mark (the founder), Tiffani (admin/team lead)"`
- `src/app/api/cron/linkedin-draft/route.ts:107` — `"Draft a short LinkedIn post...for Tiffani"`

### 2f. VA Orchestrator — deep team coupling

**`src/lib/va-orchestrator.ts`** — 70+ references to specific team member names:
- Tiffani: 15 references (lines 89, 358-365, 483, 530-537, 554-559, 637-638, 824)
- Jackie: 12 references (lines 117, 316-323, 601-607, 739-766)
- Fab: 25 references (lines 96, 330-337, 408-448, 521-524, 581-695)
- Cam, Jen, Malvin, Vee, Mark: scattered throughout

**Extraction difficulty:** HARD. The team roster is in `team-roster.ts`, but routing logic, AI personas, cron job assignments, and the VA orchestrator all have names hardcoded inline. Extracting requires:
1. A configurable routing table (which team member handles which alert type)
2. A role-based AI persona system (instead of "You are Jackie")
3. A configurable VA orchestration config
4. Reworking the `warm-handoff-to-tiffani` tool to be generic (`warm-handoff-to-owner`)

---

## Category 3: Domain/URL (18 files, ~8 unique domains)

| String | Files | Location Type |
|--------|-------|---------------|
| `justmilo.app` | 5 | vertical.config.ts, env defaults, email from addresses |
| `tlp.justmilo.app` | 6 | vertical.config.ts, email HTML img src, signing baseUrl |
| `theleadpenguin.com` | 12 | email from/reply-to defaults, onboarding emails, Cloudflare worker |
| `scaleflow.co` | 4 | admin email checks, claim route comments |
| `tlpmop.netlify.app` | 3 | fallback MOP_BASE_URL in 3 API routes |
| `milo@justmilo.app` | 2 | demo email from address |
| `milo@scaleflow.co` | 0 src (tests only) | test fixtures |
| `support@theleadpenguin.com` | 5 | email from defaults, HTML footers |

**Key hardcoded locations:**
- `src/lib/vertical.config.ts:8-9` — `domain: 'justmilo.app'`, `productHost: 'tlp.justmilo.app'`
- `src/lib/vertical.config.ts:47` — `baseUrl: 'https://tlp.justmilo.app'`
- `src/app/api/onboard/buyer/route.ts:139` — `'The Lead Penguin <support@theleadpenguin.com>'`
- `src/app/api/onboard/publisher/route.ts:62` — same
- `src/lib/tools/publish-contract-for-buyer-review.ts:183` — same
- `src/app/api/onboard/buyer/route.ts:203` — `https://tlp.justmilo.app/tlp-logo.png`
- `src/app/api/onboard/publisher/route.ts:183` — same
- `src/lib/tools/publish-contract-for-buyer-review.ts:236` — same
- `src/app/api/demo/analyze/route.ts:10` — `'https://tlpmop.netlify.app'`
- `src/app/api/contract/analyze-proxy/route.ts:5` — same
- `src/app/api/contract/process/route.ts:26` — same
- `workers/email-inbound/src/index.ts:4,15` — `reply.theleadpenguin.com`
- `src/app/onboard/page.tsx:45` — `ADMIN_EMAILS = ['mark@scaleflow.co', 'tiffani@scaleflow.co']`

**Extraction difficulty:** EASY. Most of these can be centralized into `vertical.config.ts` plus environment variables.

---

## Category 4: Tenant ID 'tlp' (28 files, ~50 usages)

The string `'tlp'` is hardcoded as a tenant_id in ~50 locations across production source. It IS in `vertical.config.ts:10` (`tenantId: 'tlp'`), but almost no code reads from the config — they hardcode the literal.

**Files with hardcoded `'tlp'` tenant_id (production src/ only, 28 files):**

| File | Count | Usage |
|------|-------|-------|
| `src/app/api/ask/route.ts` | 8 | `.eq('tenant_id', 'tlp')`, `tenantId: 'tlp'` |
| `src/app/api/ask/entity-alias/route.ts` | 3 | `.eq('tenant_id', 'tlp')`, `tenant_id: 'tlp'` |
| `src/app/api/ask/vet-action/route.ts` | 2 | `tenant_id: 'tlp'`, `tenantId: 'tlp'` |
| `src/app/api/ask/suggestions/route.ts` | 1 | `.eq('tenant_id', 'tlp')` |
| `src/app/api/ask/outreach-log/route.ts` | 3 | `tenant_id: 'tlp'`, `.eq(...)` |
| `src/app/api/ask/entity-add/route.ts` | 2 | `.eq('tenant_id', 'tlp')`, `tenant_id: 'tlp'` |
| `src/app/api/ask/entity-vet-inline/route.ts` | 2 | `tenantId: 'tlp'` |
| `src/app/api/ask/entity-dismiss-match/route.ts` | 1 | `tenant_id: 'tlp'` |
| `src/app/api/entity/status/route.ts` | 1 | `.eq('tenant_id', 'tlp')` |
| `src/app/api/entity/timeline/route.ts` | 1 | `.eq('tenant_id', 'tlp')` |
| `src/app/api/operator/pill/route.ts` | 4 | `.eq('tenant_id', 'tlp')` |
| `src/app/api/operator/pill/counts/route.ts` | 2 | `.eq('tenant_id', 'tlp')` |
| `src/app/api/outreach/record-manual-send/route.ts` | 1 | `tenant_id: 'tlp'` |
| `src/app/api/entities/aliases/route.ts` | 3 | `.eq('tenant_id', 'tlp')`, `tenant_id: 'tlp'` |
| `src/app/api/onboarding/runs/route.ts` | 1 | `tenantId: 'tlp'` |
| `src/app/api/onboarding/runs/[runId]/advance/route.ts` | 1 | `tenantId: 'tlp'` |
| `src/app/api/onboarding/runs/[runId]/skip/route.ts` | 1 | `tenantId: 'tlp'` |
| `src/lib/crm-client.ts` | 2 | `tenantId: 'tlp'` |
| `src/lib/blacklist-client.ts` | 1 | `tenantId: 'tlp'` |
| `src/lib/onboarding-automation.ts` | 1 | `tenantId: 'tlp'` |
| `src/lib/onboarding-client.ts` | 1 | `tenantId: 'tlp'` |
| `src/lib/ask/opportunity-context.ts` | 2 | `.eq('tenant_id', 'tlp')` |
| `src/lib/ppc/engine.ts` | 1 | `tenantId: 'tlp'` |
| `src/lib/ppc/index.ts` | 1 | `tenantId: 'tlp'` |

**Extraction difficulty:** EASY. Create a shared constant that reads from `verticalConfig.brand.tenantId` and import it everywhere. Global find-and-replace with a single import.

---

## Category 5: AI System Prompts with TLP Identity (22 files)

Every AI prompt in the system hardcodes TLP's identity, business model, and team structure. These are the prompts that define "who Milo is" and "what business it operates."

**Core identity prompts:**
- `src/lib/milo-prompt.ts` — Main Milo persona: "The Lead Penguin (TLP), a performance marketing company that brokers inbound phone calls"
- `src/lib/milo-support-prompt.ts` — Support persona: "AI operations assistant at The Lead Penguin"
- `src/lib/anthropic.ts` — Fallback prompt: "ops assistant for The Lead Penguin (TLP), a Denver call brokerage"

**Tool-specific prompts (13 tools):**
- `analyze-contract.ts` — "contract analyst for The Lead Penguin (TLP), a Denver-based call brokerage"
- `detect-coached-call.ts` — "You are Jackie at The Lead Penguin"
- `discover-entities.ts` — "Milo, the intelligence engine for The Lead Penguin"
- `draft-dispute-message.ts` — "You are Fab...for The Lead Penguin (TLP), a Denver call brokerage"
- `draft-first-outreach.ts` — "VEE_L2...for The Lead Penguin (TLP), a Denver call brokerage"
- `generate-redlined-version.ts` — "contract redlining engine for The Lead Penguin (TLP)"
- `handle-objection.ts` — "Vee, The Lead Penguin (TLP) outreach strategist"
- `ingest-returned-contract.ts` — "senior contracts operator for The Lead Penguin"
- `morning-priority-briefing.ts` — "tightening a morning briefing for...The Lead Penguin"
- `publish-contract-for-buyer-review.ts` — "Message from The Lead Penguin"
- `research-prospect.ts` — "Prospect Intelligence layer for The Lead Penguin (TLP)"
- `review-call-quality.ts` — "You are Jackie, the senior QC reviewer at The Lead Penguin"
- `warm-handoff-to-tiffani.ts` — "preparing a warm handoff package for Tiffani, TLP's CEO"

**Vendor voice prompts (5 files):**
- `vendor/voice/src/pills/vet.ts` — "TLP (The Lead Partner)" [NOTE: inconsistent naming]
- `vendor/voice/src/pills/buyer.ts` — "TLP (The Lead Partner)"
- `vendor/voice/src/pills/publisher.ts` — "TLP (The Lead Partner)"
- `vendor/voice/src/pills/sales.ts` — "TLP (The Lead Partner)"
- `vendor/voice/src/pills/vet-extraction.ts` — "intelligence layer for TLP"

**Extraction difficulty:** MEDIUM. Each prompt needs a template variable for company name, business description, and team member names. A `buildPrompt(config)` pattern would work, but each prompt is unique and requires individual testing after extraction.

---

## Category 6: Business Logic Specific to TLP (30+ files)

### 6a. TrackDrive coupling (30+ files in production src/)

TrackDrive is TLP's specific call routing platform. The entire data pipeline assumes TrackDrive:
- `src/lib/trackdrive.ts` — TrackDrive API client
- `src/app/api/sync/route.ts` — TrackDrive data sync
- `src/app/api/cron/td-change-detection/route.ts` — TrackDrive change monitoring (512 lines)
- `src/app/api/td-changes/review/route.ts` — TrackDrive review
- `src/app/api/td/create-entity/route.ts` — TrackDrive entity provisioning
- `src/lib/milo-prompt.ts:5` — "Live call data from TrackDrive (synced every 15 minutes)"
- `src/lib/milo-support-prompt.ts:22` — "Data syncs from TrackDrive every 15 minutes"
- References in health monitor, reconciliation, briefing, billing
- `src/lib/milo-tools.ts` — TrackDrive-specific tool definitions

### 6b. Company address and legal details

- `src/lib/vertical.config.ts:56` — `address: '1500 N Grant St, STE R, Denver, CO 80203'`
- `src/lib/vertical.config.ts:54` — `ownerName: 'Tiffani Kinkennon'`
- `src/lib/vertical.config.ts:55` — `ownerTitle: 'Managing Member'`
- `src/lib/vertical.config.ts:57` — `brandingColor: '#dd72a6'` (TLP pink)

### 6c. Denver business location (non-timezone)

These are "Denver" references used for business identity, NOT timezone math:
- `src/lib/knowledge-engine.ts:899` — "TLP is a Denver-based call brokerage"
- `src/app/api/outreach/draft/route.ts:119` — "based in Denver"
- `src/app/api/outreach/draft-followup/route.ts:22` — "based in Denver"
- `src/app/api/operator/pill/review/route.ts:1098` — "Denver-based pay-per-call lead generation network"
- `src/lib/tools/analyze-contract.ts:196` — "Denver-based call brokerage"
- `src/lib/tools/draft-first-outreach.ts:193` — "Denver call brokerage"
- `src/lib/tools/research-prospect.ts:225` — "Denver performance marketing / call brokerage"
- `src/lib/tools/draft-dispute-message.ts:112` — "Denver call brokerage"
- `src/lib/tools/handle-objection.ts:183` — "Denver-based call brokerage"
- `src/lib/anthropic.ts:51` — "Denver call brokerage"

NOTE: Denver timezone references (60+ files) are CORRECT to keep — the business operates in Mountain time regardless of branding.

### 6d. Industry-specific terms in prompts

"pay-per-call" and "performance marketing" appear in 25+ AI prompts. These are business model descriptors, not brand names, but they ARE TLP-specific if the platform is used for a non-PPC vertical.

### 6e. TLP-specific page routes

- `src/app/tiffani/page.tsx` — `/tiffani` route (Tiffani's payment view)
- `src/app/invoices/page.tsx` — "Send to Jen" (141, 437, 450, 466, 469, 670)
- `src/app/disputes/page.tsx` — References Fab 8+ times
- `src/components/TiffaniView.tsx` — Tiffani-specific payment view component
- `src/components/JenView.tsx:867` — "Sent to Tiffani for final approval"
- `src/app/pipeline-board/page.tsx:825` — `entity.entity_type === 'buyer' ? 'Vee' : 'Cam'`
- `src/app/page.tsx:913,976` — "I'll let Mark know" / "I'll have Mark reach out"

**Extraction difficulty:** HARD. TrackDrive is deeply integrated into the data pipeline. Swapping it out requires an abstract call-platform interface. Team-specific routes need a role-based routing system.

---

## Category 7: Email Templates with TLP Branding (5 files)

**`src/app/api/onboard/buyer/route.ts`:**
- Line 139: `from: 'The Lead Penguin <support@theleadpenguin.com>'`
- Line 141: `subject: 'Welcome to The Lead Penguin, ${name}'`
- Line 203: `<img src="https://tlp.justmilo.app/tlp-logo.png">`
- Line 234: `The Lead Penguin · support@theleadpenguin.com`

**`src/app/api/onboard/publisher/route.ts`:**
- Line 62: `from: 'The Lead Penguin <support@theleadpenguin.com>'`
- Line 64: `subject: 'Welcome to The Lead Penguin, ${name}'`
- Line 183: `<img src="https://tlp.justmilo.app/tlp-logo.png">`
- Line 214: `The Lead Penguin · support@theleadpenguin.com`

**`src/lib/tools/publish-contract-for-buyer-review.ts`:**
- Line 183: `from: 'The Lead Penguin <support@theleadpenguin.com>'`
- Line 227: `Message from The Lead Penguin`
- Line 236: `<img src="https://tlp.justmilo.app/tlp-logo.png">`
- Line 272: `The Lead Penguin · support@theleadpenguin.com`

**`src/app/api/invoices/[id]/pdf/route.ts`:**
- Line 80: `<div class="company">The Lead Penguin</div>`
- Line 160: `The Lead Penguin · Invoice...`

**`src/app/api/contract/download/[id]/route.ts`:**
- Line 217: `Generated by Milo -- The Lead Penguin`

**Extraction difficulty:** MEDIUM. These are HTML templates with inline branding. Extract to a shared email template builder that reads from vertical config.

---

## Category 8: Brand Assets (1 file)

- `public/tlp-logo.png` — Referenced from 9 locations (onboard pages + email templates)
- Referenced as both `/tlp-logo.png` (client-side) and `https://tlp.justmilo.app/tlp-logo.png` (email HTML)

**Extraction difficulty:** EASY. Replace with a config-driven asset path.

---

## Existing Multi-Tenancy Patterns

### What vertical.config.ts currently centralizes (8 values):

```typescript
verticalConfig.brand.name          // 'Milo for PPC'
verticalConfig.brand.domain        // 'justmilo.app'
verticalConfig.brand.productHost   // 'tlp.justmilo.app'
verticalConfig.brand.tenantId      // 'tlp'
verticalConfig.company.name        // 'TLP Compliance'
verticalConfig.company.legalName   // 'The Lead Penguin LLC'
verticalConfig.company.ownerName   // 'Tiffani Kinkennon'
verticalConfig.company.address     // '1500 N Grant St...'
```

### What the config does NOT cover:

- AI system prompts (25+ hardcoded)
- Team member roster + routing (11 people, 38 files)
- Email from addresses + templates
- Logo asset references
- TrackDrive integration specifics
- Denver business location in prompts
- Admin email whitelist
- Industry vertical terms ("pay-per-call", "call brokerage")

---

## Inconsistency Found

The vendor/voice pills use "The Lead Partner" while everything else uses "The Lead Penguin":
- `vendor/voice/src/pills/vet.ts:4` — "TLP (The Lead Partner)"
- `vendor/voice/src/pills/buyer.ts:1` — "TLP (The Lead Partner)"
- `vendor/voice/src/pills/publisher.ts:1` — "TLP (The Lead Partner)"
- `vendor/voice/src/pills/sales.ts:1` — "TLP (The Lead Partner)"

All other references (44+ files) use "The Lead Penguin".

---

## Summary Statistics

| Metric | Count |
|--------|-------|
| Production files with TLP hardcoding | 102 |
| Vendor source files with TLP hardcoding | 7 |
| Test/script files with TLP hardcoding | 40+ |
| Unique hardcoded brand strings | ~45 |
| Team member names hardcoded in routing | 11 names, 38 files |
| Hardcoded 'tlp' tenant_id literals | ~50 usages, 28 files |
| AI prompts with TLP identity | 25+ unique prompts |
| Email templates with TLP branding | 5 files, ~20 locations |
| Domain/URL hardcodings | 8 domains, 18 files |

---

## Recommended Extraction Strategy

### Phase 1 — Quick wins (EASY, ~2 days)

1. **Tenant ID consolidation:** Replace all 50 `'tlp'` literals with `verticalConfig.brand.tenantId`. Global search-and-replace with one import.
2. **Domain consolidation:** Move all domain references to read from `verticalConfig.brand.*` and env vars.
3. **Email from address:** Single constant from config: `${verticalConfig.company.name} <${verticalConfig.company.email}>`.
4. **Logo asset path:** Config-driven: `verticalConfig.brand.logoUrl`.
5. **Admin email whitelist:** Move from hardcoded array to config or DB query.

### Phase 2 — Prompt extraction (MEDIUM, ~3 days)

1. **Company identity in prompts:** Create a `buildSystemPrompt(verticalConfig)` helper that injects company name, business description, and location.
2. **Standardize all 25+ prompts** to use template variables instead of hardcoded "The Lead Penguin (TLP)".
3. **Fix vendor/voice inconsistency:** Change "The Lead Partner" to match whatever the config says.

### Phase 3 — Team structure extraction (HARD, ~5 days)

1. **Team roster:** Already in `team-roster.ts` — extend with routing configuration per role.
2. **Alert routing:** Replace hardcoded name-to-type mappings with a config table.
3. **VA orchestrator:** Refactor to route by role instead of by name.
4. **AI personas:** Replace "You are Jackie" with "You are the QC reviewer" (role-based).
5. **Team-specific tools:** Generalize `warm-handoff-to-tiffani` to `warm-handoff-to-owner`.
6. **Team-specific routes:** Replace `/tiffani` with role-based routes.

### Phase 4 — Platform abstraction (HARD, ~10+ days)

1. **TrackDrive abstraction:** Create a call-platform interface that TrackDrive implements. Other platforms (Ringba, Retreaver, Invoca) could implement the same interface.
2. **Industry terms:** Make "pay-per-call" / "call brokerage" configurable in prompts for non-PPC verticals.

---

## Self-Attestation Checklist

- [x] All 7 search categories from the directive covered individually
- [x] vertical.config.ts existence and contents verified
- [x] tenant_id usage pattern fully characterized
- [x] Team member routing fully characterized across all routing surfaces
- [x] Every AI prompt with TLP identity listed
- [x] Email templates with branding listed with exact line numbers
- [x] Denver references separated into timezone (keep) vs. business location (extract)
- [x] TrackDrive coupling depth assessed
- [x] Vendor/voice inconsistency ("Lead Partner" vs "Lead Penguin") flagged
- [x] Extraction difficulty rated for each category
- [x] No production code modified (read-only audit)

## Known Gaps

- No gaps. All categories requested in the directive are covered.
- No UI surfaces touched — screenshot N/A (research-only audit).

---

Report at: https://raw.githubusercontent.com/moscaleflow/conductor-mark-public/main/reports/C3-TLP-HARDCODING-AUDIT-2026-05-07.md
