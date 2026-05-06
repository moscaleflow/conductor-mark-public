# PPC Workflow Front-to-Back Audit

**Date:** 2026-05-06
**Auditor:** Coder-3 (Research lane)
**Repos:** milo-for-ppc, milo-engine
**Scope:** Entity discovery through live traffic -- every stage of the PPC broker workflow

---

## Stage 1: Entity Discovery & First Contact

**Status:** WORKING

**What works:**
- `POST /api/ask/entity-add` -- creates CRM counterparty + prospect_pipeline entry from entity discovery cards in /ask chat. Includes dedup check via `checkCounterpartyExists()` before creating. (`src/app/api/ask/entity-add/route.ts`)
- `POST /api/ask/entity-check` -- batch check up to 50 entity names against CRM. Uses `normalizeCompanyName()` (suffix-stripping: LLC, Inc, Corp, etc.) for fuzzy matching. Loads all ~664 counterparties and compares client-side. (`src/app/api/ask/entity-check/route.ts`)
- `findOrCreateCounterparty()` -- the CRM gateway function. Exact case-insensitive match first, then normalized fuzzy match using longest-word ilike anchor. Picks most recently created on tie. (`src/lib/crm-client.ts`)
- `findOrCreatePipelineEntry()` -- dedup gateway for prospect_pipeline. Case-insensitive lookup by entity_name + entity_type. All pipeline creation paths route through this. (`src/lib/crm-client.ts`)
- Entity verification via Claude AI -- `verifyEntity()` takes entity name + context signals (PPC platform, display name, named person, industry terms, peer companies) and returns confidence score + tier (verified/likely/review_needed). Generic name detection with pattern-based penalty. (`src/lib/entity-verification.ts`)

**What's broken or incomplete:**
- `normalizeCompanyName()` strips legal suffixes but does NOT handle abbreviation matching. "NLD" will NOT resolve to "Next Level Direct" -- there is no abbreviation/acronym expansion. The normalization is suffix-only, not fuzzy in the Levenshtein/phonetic sense.
- Entity verification is AI-powered but has no persistent disambiguation UI. The operator sees confidence tiers in chat but cannot confirm/deny a match interactively -- the AI decides. `needs_human_review: true` is set but there is no UI component that surfaces this flag for operator action.
- No inbound webhook or monitoring for "new entity approaches us" outside the /ask chat flow. Discovery is operator-initiated (they type into Milo chat or use entity cards).

**What's missing:**
- No interactive disambiguation UI -- "Is NLD the same as Next Level Direct?" requires operator to manually use `/api/entities/aliases` POST endpoint to create an alias. There is no prompt-driven flow.
- No automated monitoring for new entities appearing in external systems (TrackDrive, email, etc.) -- all discovery flows start from operator action in Milo chat.

---

## Stage 2: Vetting & Blacklist

**Status:** WORKING

**What works:**
- `POST /api/ask/entity-vet-inline` -- full inline vet via Claude + web_search tool (max 3 searches). Returns verdict (blacklisted/likely_scam/likely_real/unclear), summary, flags, facts. Caches results in `vet_results` table with 30-day freshness window. (`src/app/api/ask/entity-vet-inline/route.ts`)
- Vet result caching with alias resolution -- `lookupExistingVet()` checks exact match, then entity_aliases table, then prefix match. Prevents redundant web searches. (`src/lib/ask/vet-lookup.ts`)
- `POST /api/blacklist/screen` -- screens entity against @milo/blacklist. Checks exact_name, fuzzy_name, alias, contact_name, email_domain, ip_address, linkedin_url. Logs every screening to ai_action_log. (`src/app/api/blacklist/screen/route.ts`)
- Blacklist check is AUTOMATIC on onboarding trigger -- `triggerPublisherOnboarding()` calls `screenEntity()` as its first step. If blocked, logs to ai_action_log and returns error. (`src/lib/publisher-onboarding.ts`)
- `POST /api/ask/vet-action` -- four actions from vet cards: `add_crm` (creates counterparty + pipeline entry + logs CRM activity), `add_blacklist` (adds to @milo/blacklist), `assign` (assigns vet to team member), `update_vet_result` (updates entity name/type). (`src/app/api/ask/vet-action/route.ts`)
- Vet result persistence with normalized names, CRM cross-references, pipeline entity links. (`src/lib/ask/vet-results.ts`)

**What's broken or incomplete:**
- Vet check is NOT automatic on entity-add. When an operator clicks "Add to CRM" from an entity card (`/api/ask/entity-add`), there is no blacklist screen. The blacklist gate only fires when onboarding is explicitly triggered.
- No vet history viewer per entity. An operator can look up individual vet results via `GET /api/ask/vet-action?vetResultId=X`, but there is no "show me all vets for this entity" endpoint or UI panel.

**What's missing:**
- No automatic vet trigger on entity creation -- operator must manually trigger `/ask` vet.
- No vet history timeline UI per entity (multiple vets over time, freshness indicators).

---

## Stage 3: Onboarding Flow

**Status:** WORKING (with gaps)

**What works:**
- **12-step publisher flow** defined in `onboarding-client.ts`: add_to_mop -> msa_sent -> msa_signed -> w9_sent -> w9_signed -> creatives_submitted -> creatives_approved -> did_provisioned -> config_validated -> io_sent -> io_signed -> go_live. Each step has dependencies, timeout_days, and integration metadata.
- **8-step buyer flow** defined: io_sent -> io_signed -> td_buyer_created -> campaigns_assigned -> bid_price_confirmed -> crm_counterparty_synced -> qa_complete -> ready_to_launch.
- `startPublisherOnboarding()` / `startBuyerOnboarding()` -- creates run in @milo/onboarding (shared Supabase), creates step rows with proper dependency blocking. (`src/lib/onboarding-client.ts`)
- **Three entry paths** converge in `triggerPublisherOnboarding()`: Path A (existing pipeline entity), Path B (trusted shortcut -- no prior pipeline record), Path C (inbound form submission). All run blacklist screen, create pipeline entry via dedup gateway, start onboarding run, create action_items, log to ai_action_log. (`src/lib/publisher-onboarding.ts`)
- **External publisher form** at `/onboard/publisher/[token]` -- cinematic multi-step form with verticals, call types, creatives upload. Sends to `POST /api/onboard/publisher` which calls `handleInboundPublisherSubmission()`. Actually starts an onboarding run. (`src/app/onboard/publisher/[token]/page.tsx`, `src/app/api/onboard/publisher/route.ts`)
- **Creative upload** via PUT on the same route -- stores in Supabase storage `onboarding-creatives` bucket, creates action_item for TCPA compliance review.
- **Stall detection cron** -- `GET /api/cron/onboarding-stall-check` finds steps active > 72 hours, creates deduplicated action_items with entity name resolution. Runs daily at 6 AM Mountain. (`src/app/api/cron/onboarding-stall-check/route.ts`)
- **Run state & progress** -- `getOnboardingStatusByCounterparty()` and `getOnboardingStatusByRunId()` return completion_pct, step statuses, stalled_days, next activatable steps.
- **Step operations** in @milo/onboarding engine: `advanceStep()`, `failStep()`, `skipStep()` with proper state machine transitions. Auto-activates blocked steps when dependencies complete. Auto-completes run when all required steps done. (`packages/onboarding/src/steps.ts`)

**What's broken or incomplete:**
- **No operator UI to VIEW onboarding progress** as a dedicated page. The pipeline-board page (`/pipeline-board`) shows entities in columns but does not surface onboarding checklist progress inline. The `EntityDetailPanel` component exists and loads data, but the onboarding checklist rendering inside it depends on partner service data, not the @milo/onboarding engine data.
- **No manual step advancement UI.** The @milo/onboarding engine supports `advanceStep()`, `skipStep()`, `failStep()` but there is no API route in milo-for-ppc that exposes these to the operator. Steps can only be advanced programmatically (e.g., by a webhook from contract signing).
- **No webhook-driven auto-advancement.** The onboarding step definitions include `integration` metadata (e.g., `{ type: 'contract_signing', documentType: 'msa' }` on `msa_sent`), but there is no webhook handler that listens for "document signed" events and calls `advanceStep('msa_signed')`. This is wired in the step definition but not implemented in the consumer.
- **Document generation route** (`POST /api/onboard/generate-docs`) proxies to Milo Engine but depends on `MILO_ENGINE_API_URL` and `MILO_ENGINE_API_KEY` env vars. If Milo Engine is down or unconfigured, doc generation fails silently.

**What's missing:**
- No operator-facing onboarding checklist panel with manual step controls (advance/skip/fail buttons).
- No webhook listener that auto-advances onboarding steps when contracts are signed.
- No buyer onboarding external form (only publisher form exists at `/onboard/publisher/[token]`).

---

## Stage 4: Contract Generation & E-Signatures

**Status:** PARTIAL

**What works:**
- **@milo/contract-signing engine** -- full signing lifecycle in milo-engine. Creates signing documents with tokens + short codes, renders HTML from templates, captures views, submits signatures (with SHA-256 hashing, audit trail, consent tracking, IP/UA capture). Supports decline and void flows. (`packages/contract-signing/src/documents.ts`)
- **Document types supported:** MSA (buyer + publisher variants), IO, W-9, W-8BEN, W-8BENE, addendum. Type detection from filename + content in `detectDocumentType()`. (`src/app/api/contract/process/route.ts`)
- **Contract processing pipeline** (`POST /api/contract/process`): extract text via Milo Engine, detect metadata (company, doc type, redline status), cross-system entity search (Supabase + Milo Partner), entity verification, redline detection with version linking, store in `contract_documents` with dedup. Handles version auto-increment for redlines.
- **Contract analysis** -- two paths:
  1. `POST /api/contract/analyze-proxy` -- forwards to Milo Engine for async analysis (extract -> analyze-async -> jobId). (`src/app/api/contract/analyze-proxy/route.ts`)
  2. `analyze_contract` tool -- synchronous analysis via Claude Opus. Reads from `contract_documents` row, produces clause-by-clause severity assessment, risk summary, signing readiness. Persists back to row. (`src/lib/tools/analyze-contract.ts`)
- **Contract review page** at `/contract-review/[id]` -- loads contract document, displays flagged clauses with risk levels (critical/high/medium/low/favorable), supports accept/reject/counter decisions per clause, inline redline editor component, improvements suggestions. (`src/app/contract-review/[id]/page.tsx`)
- **Contract queue page** at `/contract-queue` -- lists all contracts with risk filtering, document type labels, age filtering, document dropzone for new uploads. (`src/app/contract-queue/page.tsx`)
- **Negotiation submit** (`POST /api/contract/negotiate`) -- creates negotiation via partner service from clause decisions, generates review link for counterparty, persists decisions back to contract_documents, creates follow-up action_items. Also supports "review_later" action that creates a task.
- **Publish for buyer review** (`publish_contract_for_buyer_review` tool) -- generates 6-char short_slug, builds public review URL, sets status='sent', records buyer email + deadline. NOTE: actual email notification is TODO v1.1.
- **Ingest returned contract** (`ingest_returned_contract` tool) -- creates new version linked to parent, runs Claude Opus diff against prior version, produces structured change entries with severity, chains to analyze_contract.
- **Document status endpoint** (`GET /api/onboard/doc-status?entity_name=X`) -- checks Milo Engine for live signing status, falls back to local contract_documents.
- **Generate docs endpoint** (`POST /api/onboard/generate-docs`) -- proxies to Milo Engine with agreed_terms data for IO merge fields. Guards: IO requires confirmed agreed_terms with non-empty qualifiers.

**What's broken or incomplete:**
- **Signing flow is split across two systems.** @milo/contract-signing (in milo-engine) has the full signing lifecycle, but milo-for-ppc's contract_documents table is a separate system. The `signing_token` field exists on contract_documents but there is no route in milo-for-ppc that renders a signing page using @milo/contract-signing. The signing UI lives in the partner service (PPCRM/Milo Engine), not in milo-for-ppc.
- **Email notification on signing is conditional.** The @milo/contract-signing `submitSignature()` sends confirmation email via ResendEmailProvider IF an emailProvider is configured. In milo-for-ppc, there is no direct integration -- signing happens through the partner service. The publisher onboarding form does send a welcome email via Resend.
- **No countersign flow.** The signing lifecycle supports one signer (counterparty). There is no "operator countersigns after counterparty signs" step. The system assumes TLP's signature is implicit via the template.
- **publish_contract_for_buyer_review** explicitly states "TODO v1.1: send the actual notification email." The publish tool records intent but does not send the email.
- **The `/c/[slug]` short URL route** is noted as "out of scope for this sprint" in the publish tool. Short codes are generated but the resolver route may not exist in milo-for-ppc.
- **analyze_contract tool reads text from `notes` column** as a fallback. The TODO says "v1.1: fetch document_url, parse PDF/DOCX -> text." If the contract text is not staged in notes, analysis fails.

**What's missing:**
- No signing page rendered by milo-for-ppc itself -- relies on partner service for signing UI.
- No countersign workflow.
- No email notification when publishing contracts for review (TODO v1.1).
- No `/c/[slug]` short URL route for contract review links.
- No PDF generation/download from within milo-for-ppc. The @milo/contract-signing has `generated_pdf_url` field but no PDF renderer.

---

## Stage 5: Contract Analysis

**Status:** WORKING

**What works:**
- **@milo/contract-analysis engine** in milo-engine -- full analysis pipeline. `createAnalysisEngine()` configures: AI model (Claude Sonnet), rule registry, vertical rule packs, scoring profiles, jurisdiction detection, post-processing, mandatory checks. (`packages/contract-analysis/src/engine.ts`)
- **Async job system** -- `analyzeAsync()` creates background jobs, `getJobStatus()` polls. Stored in Supabase.
- **Revision comparison** -- `compareRevisions()` diffs original vs revised contract text, verifies which changes were implemented.
- **Jurisdiction detection** -- auto-detects state from contract text, factors into scoring.
- **Scoring** -- normalized risk score with buckets, severity-based scoring with rule-specific weights, profile-based modifiers.
- **Display config** -- vertical rule packs can customize how issues are presented.
- **Proxy route** in milo-for-ppc (`POST /api/contract/analyze-proxy`) -- two-step: extract text -> submit for async analysis. Returns jobId for polling.
- **Inline analysis tool** (`analyze_contract`) -- synchronous Opus-tier analysis for Milo chat. Returns clause-by-clause assessment with severity, recommendation, signing readiness.
- **ContractAnalysisDisplay component** exists for rendering analysis results in UI. (`src/components/shared/ContractAnalysisDisplay.tsx`)
- **Contract review page** renders analysis results with interactive clause decisions.

**What's broken or incomplete:**
- **No versioning UI for analysis.** When a contract is re-analyzed after redlining, the new analysis overwrites the old one on the same row. The `contract_documents` table supports versioning via `contract_group_id` + `parent_version_id`, but the analysis_result is per-row, not a separate versioned entity.
- **Document links from analysis.** Analysis results are stored as JSON in `analysis_result` column but there is no "link to contact/counterparty" explicitly in the analysis flow. The link exists via `counterparty_name` + `prospect_id` on the contract_documents row.

**What's missing:**
- No red-line tracking UI that shows the evolution of analysis across contract versions side-by-side.
- No analysis history (multiple analyses of the same contract over time -- only latest is stored).

---

## Stage 6: Document Storage & Retrieval

**Status:** PARTIAL

**What works:**
- **contract_documents table** stores all contracts with: document_type, version, counterparty_name, status, risk_level, analysis_result, signing_token, contract_group_id (for version groups), parent_version_id (for version chains).
- **GET /api/contract/documents?entity=<name>** -- fetches all contract_documents for an entity, groups by contract_group_id, sorts versions chronologically. Returns flagged clause counts, decision summaries per version. (`src/app/api/contract/documents/route.ts`)
- **GET /api/documents/entity?entity_name=<name>** -- unified document view merging partner service documents with local contract_documents. Both sources sorted by created_at. (`src/app/api/documents/entity/route.ts`)
- **Supabase storage** for creatives: `onboarding-creatives` bucket. Upload via `PUT /api/onboard/publisher`.
- **Library documents** in @milo/contract-signing: `uploadLibraryDocument()`, `getLibraryDocument()`, `listLibraryDocuments()`, `archiveLibraryDocument()` -- stored in configurable Supabase storage buckets.

**What's broken or incomplete:**
- **No PDF download endpoint in milo-for-ppc.** The @milo/contract-signing engine has `generated_pdf_url` field on signing_documents, but milo-for-ppc does not render PDFs or provide a download route. The signing confirmation email links to `{baseUrl}/sign/{token}` which renders the HTML version.
- **Document storage is split.** Signing documents are in milo-engine's `signing_documents` table. Contract documents (analysis, redlines) are in milo-for-ppc's `contract_documents` table. Partner service has its own documents. Three separate storage locations.
- **Documents linked to entities** via `counterparty_name` string match (ilike), not a foreign key. This means name changes can break the link.

**What's missing:**
- No unified document library page (a single view of all documents for all entities).
- No PDF generation from signed HTML contracts.
- No document download button in the UI for signed contracts.

---

## Stage 7: Email Notifications

**Status:** PARTIAL

**What works:**
- **Publisher onboarding confirmation email** -- sent via Resend when publisher submits the external onboarding form. Full HTML template with TLP branding, submission summary, and "what happens next" steps. (`src/app/api/onboard/publisher/route.ts`, `buildConfirmationEmail()`)
- **Signing confirmation email** -- @milo/contract-signing's `ResendEmailProvider` sends "Document Signed Successfully" email with download link after signature submission. HTML template with document details + E-SIGN Act disclaimer. (`packages/contract-signing/src/email.ts`)
- **Webhook on signing** -- @milo/contract-signing fires webhook to configured URL after signing, with document_id, signed_at, signature_hash, counterparty info. Retry on failure. Audit trail updated. (`packages/contract-signing/src/documents.ts`)

**What's broken or incomplete:**
- **Signing confirmation only fires IF emailProvider is configured** on the signing client. In the partner service context this is likely configured; in direct @milo/contract-signing usage from milo-for-ppc, it depends on client setup.
- **No email notification when a contract is SENT for signature.** The `publish_contract_for_buyer_review` tool explicitly says "TODO v1.1: send the actual notification email." The tool generates the review URL but does not email it.
- **No notification when onboarding steps complete.** The stall detection cron creates action_items but does not send emails.

**What's missing:**
- No "contract sent for your signature" email to counterparty.
- No email notifications for onboarding step completion or stall alerts.
- No email digest/summary for operators (daily onboarding status, contract status).
- No email tracking (open/click) on any sent emails.

---

## Stage 8: CRM Pipeline Integration

**Status:** WORKING

**What works:**
- **Entity-add creates pipeline entry** -- `POST /api/ask/entity-add` creates both CRM counterparty and prospect_pipeline row. Pipeline defaults to 'unverified' stage.
- **Vet-to-CRM creates pipeline entry** -- `vet-action` with `add_crm` creates pipeline entry at 'vetted' stage via `findOrCreatePipelineEntry()`.
- **Onboarding advances pipeline** -- `triggerPublisherOnboarding()` updates pipeline to 'onboarding' stage. Also handles auto-advancing from early stages (outreach, qualifying, drip) on inbound submissions.
- **Pipeline board** at `/pipeline-board` -- full Kanban board with drag-and-drop (react-dnd). Columns: unverified, vetted, outreach, qualifying, drip, onboarding, activation, active, dormant, blacklisted. Entity detail panel on click. Search via Fuse.js fuzzy matching. Days-in-stage tracking.
- **Stage change API** (`POST /api/pipeline/[id]/stage`) -- validates stage transitions, activation gate (requires signed MSA + tax doc + IO), OVERRIDE mechanism, auto-move detection, ai_action_log tracking.
- **Pipeline listing** (`GET /api/pipeline`) -- filterable by stage, type, search. Optional 7-day call stats enrichment.
- **Activity logging** via @milo/crm `logActivity()` -- logs CRM activities when entities are created/vetted.
- **Activation gate** -- moving to 'active' requires signed documents. Checks Milo Engine first, falls back to local contract_documents. Missing docs block the move unless operator types "OVERRIDE".

**What's broken or incomplete:**
- **Contract signing does NOT auto-advance pipeline.** There is no webhook listener in milo-for-ppc that moves pipeline from 'onboarding' to 'activation' when all documents are signed. The activation gate checks for signed docs when an operator manually drags to 'active', but does not proactively advance.
- **Onboarding completion does NOT auto-advance pipeline.** @milo/onboarding auto-completes the run when all required steps are done, but there is no listener that updates prospect_pipeline stage from 'onboarding' to 'activation'.
- **No full entity timeline.** The EntityDetailPanel loads contacts, blacklist matches, routes, captures, creatives, but does not show a unified chronological timeline of all events (vet, pipeline changes, contract events, onboarding steps, calls, outreach).

**What's missing:**
- No auto-advancement from onboarding -> activation -> active based on completion signals.
- No unified entity timeline view.
- No pipeline analytics (conversion rates between stages, average time in stage, bottleneck identification).

---

## Stage 9: Entity Matching & Disambiguation

**Status:** PARTIAL

**What works:**
- **normalizeCompanyName()** -- strips LLC, Inc, Corp, Ltd, LP, PLLC, DBA, etc. Iterative stripping handles "Acme Co. LLC" -> "acme". Collapses whitespace, lowercases. (`src/lib/crm-client.ts`)
- **Entity aliases** -- `entity_aliases` table with canonical_name, alias, alias_normalized, tenant_id. `GET /api/entities/aliases?name=X` looks up alias + pipeline matches + vet matches. `POST /api/entities/aliases` creates alias manually. (`src/app/api/entities/aliases/route.ts`)
- **Alias resolution in vet lookup** -- `lookupExistingVet()` checks entity_aliases when exact match fails, then falls back to prefix matching. (`src/lib/ask/vet-lookup.ts`)
- **AI entity verification** -- `verifyEntity()` uses Claude to assess whether a company name refers to the right entity given context signals. Returns possible_matches array with confidence scores. (`src/lib/entity-verification.ts`)
- **Generic name detection** -- 8 regex patterns for names like "First Choice Marketing", "National Media", etc. Generic names get confidence penalty unless 3+ context signals override.

**What's broken or incomplete:**
- **No interactive disambiguation flow.** When Milo finds a fuzzy match (e.g., "NLD" vs "Next Level Direct"), the operator sees confidence scores in the vet card but cannot click "yes, these are the same entity" to auto-create an alias. Alias creation requires a separate API call or manual action.
- **normalizeCompanyName is suffix-only.** "NLD" does NOT match "Next Level Direct" through normalization. Only exact + ilike + alias resolution would catch this. The alias must be manually created first.
- **No abbreviation expansion.** There is no dictionary or heuristic that maps common abbreviations to full names. The system relies entirely on manual alias creation or AI vet context.

**What's missing:**
- No "Is this the same entity?" confirmation dialog in the UI.
- No automatic alias creation from vet results (if the AI identifies "NLD" = "Next Level Direct", it should auto-suggest creating the alias).
- No bulk entity merge tool (combine two pipeline entries that are the same entity).
- No Levenshtein/phonetic matching (Soundex, Metaphone) for company names.

---

## Priority Matrix

Ordered by operator impact (how much it helps someone doing the job today):

### P0 -- Critical Gaps (blocks daily work)

| # | Gap | Stage | Impact |
|---|-----|-------|--------|
| 1 | **No onboarding checklist UI with manual step controls** | 3 | Operators cannot see or advance onboarding steps. The engine exists but has no consumer UI. They are blind to where each publisher/buyer is in the 12/8-step process. |
| 2 | **No webhook auto-advancement of onboarding steps** | 3, 8 | When an MSA is signed, nothing automatically marks `msa_signed` as complete. Operators must manually track this outside the system. |
| 3 | **No email notification when contracts are sent for signature** | 7 | The publish tool generates a review URL but does not email it. Operators must manually copy-paste the link to the counterparty. |

### P1 -- High Impact (daily friction)

| # | Gap | Stage | Impact |
|---|-----|-------|--------|
| 4 | **No auto-advancement pipeline (onboarding -> activation -> active)** | 8 | Operators must manually drag entities through pipeline stages even when completion criteria are met. |
| 5 | **No interactive entity disambiguation** | 9 | When Milo identifies a possible match, operators cannot confirm/deny inline. Alias creation is a separate manual step. |
| 6 | **No signed document PDF download** | 6 | Operators cannot download a PDF of signed contracts from milo-for-ppc. Must go to partner service. |
| 7 | **Blacklist check not automatic on entity-add** | 2 | An entity can be added to CRM without being screened. Blacklist only fires on onboarding trigger. |
| 8 | **No buyer onboarding external form** | 3 | Publishers have a polished external form; buyers do not. Buyer onboarding requires operator-initiated flow only. |

### P2 -- Medium Impact (operational polish)

| # | Gap | Stage | Impact |
|---|-----|-------|--------|
| 9 | **No unified entity timeline** | 8 | Operators cannot see all events (vets, contracts, calls, pipeline changes) for one entity in chronological order. |
| 10 | **No vet history viewer per entity** | 2 | Cannot see all historical vets for an entity. Only individual vet result lookup. |
| 11 | **No email tracking (open/click)** | 7 | No visibility into whether counterparties opened or clicked contract emails. |
| 12 | **Split document storage** | 6 | Documents spread across signing_documents, contract_documents, and partner service. No single source of truth. |

### P3 -- Nice to Have (future iterations)

| # | Gap | Stage | Impact |
|---|-----|-------|--------|
| 13 | **No countersign workflow** | 4 | System assumes TLP signature is implicit. No two-party signing flow. |
| 14 | **No contract version side-by-side comparison UI** | 5 | Analysis exists per version but no visual diff across versions. |
| 15 | **No pipeline analytics** | 8 | No conversion rates, time-in-stage metrics, or bottleneck identification. |
| 16 | **No bulk entity merge** | 9 | Cannot combine two pipeline entries discovered to be the same entity. |
| 17 | **No abbreviation/phonetic matching** | 9 | Name matching is exact + suffix-strip + alias only. No fuzzy algorithms. |

---

## Summary

The PPC workflow has substantial infrastructure built across all 9 stages. The strongest areas are **vetting** (production-ready with web search, caching, and blacklist integration), **pipeline management** (full Kanban with activation gates), and **contract analysis** (two analysis paths with scoring, jurisdiction detection, and revision comparison).

The most significant gap is the **disconnect between the onboarding engine and the operator UI**. The @milo/onboarding engine in milo-engine has complete step management (advance, skip, fail, stall detection), but milo-for-ppc has no page or panel that lets operators view and interact with onboarding steps. This means the 12-step publisher flow and 8-step buyer flow exist as backend state machines with no front-end controls.

The second critical gap is the **lack of event-driven automation** between systems. Contract signing should auto-advance onboarding steps. Onboarding completion should auto-advance pipeline stages. Neither connection exists -- the integration metadata is defined in the step definitions but no webhook handlers consume the events.

Report at: https://raw.githubusercontent.com/moscaleflow/conductor-mark-public/main/reports/ppc-workflow-front-to-back-audit-2026-05-06.md
