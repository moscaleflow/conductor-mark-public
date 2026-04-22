---
directive: "@milo/contracts pre-extraction audit"
lane: research
coder: Coder-3
started: 2026-04-19 ~21:00 MDT
completed: 2026-04-19 ~21:45 MDT
---

# @milo/contracts Pre-Extraction Audit — 2026-04-19

## 1. Executive Summary

MOP's contract system is ~26,000 LOC spanning a 73-rule AI contract analyzer, a fully custom e-signing workflow (no DocuSign), an HTML template engine for MSAs/IOs/tax docs, and a multi-round negotiation platform. The horizontal/vertical split is **moderately clean** — the rule engine framework, signing infrastructure, and negotiation workflow are generic, but the rules, templates, and entity types are deeply PPC-specific. A clean `@milo/contracts` v0.1.0 extraction of ~9,500 LOC is feasible. Critical finding: **milo-ops has a full parallel contract analysis system** (~3,500 LOC with its own evolved rule set and `contract_documents` table) that must be reconciled before or during extraction. PPCRM is a pure API consumer with zero local contract logic. mysecretary and milo-outreach have zero contract code.

## 2. Source Paths Table

**Primary repo: `~/MOP` (branch `claude/lead-penguin-mop-mvp-011CUoVx9tU8haWeq8nMGZ4p`, freeze commit `e713a91`)**

### 2a. Contract Checker Library (`lib/contract-checker/`)

| File | LOC | Purpose | H/V |
|------|-----|---------|-----|
| `rules/registry.ts` | 1,204 | 73 rule definitions (rule_id, weights, tags, rider refs) | Mixed |
| `system-prompt.ts` | 355 | Claude system prompt — PPC analysis instructions (A-K publisher, A-I buyer) | Vertical |
| `scoring.ts` | 444 | Weighted scoring framework, risk buckets, two-party consent escalation | **Horizontal** |
| `issue-postprocessing.ts` | 1,112 | Canonicalization, dedup, reclassification, missed-rule injection pipeline | Mixed |
| `profiles.ts` | 288 | Analysis profiles (TEST/SCALE/BROKER/BUYER) with weight multipliers | Mixed |
| `playbook-rules.ts` | 709 | 35 configurable playbook rules with threshold presets | Mixed |
| `bot-helpers.ts` | 403 | Text extraction (PDF/DOCX/TXT/OCR) + non-streaming analysis pipeline | Mixed |
| `export.ts` | 1,132 | DOCX/PDF/redline/rider export generators | **Horizontal** |
| `field-mapper.ts` | 189 | AI-extracted IO field mapping (payout types, transfer types, states) | Vertical |
| `merge.ts` | 157 | Fuzzy text replacement for negotiation merge | **Horizontal** |
| `jurisdiction.ts` | 110 | US state jurisdiction detection from contract text | **Horizontal** |
| `ai-persona.ts` | 40 | "Alex, Senior Contract Analyst at Lead Penguin" persona | Vertical |
| `supabase.ts` | 1,097 | All CRUD for analyses, decisions, resolutions, versions, finalized docs | Mixed |
| `rule-ids.ts` | 226 | Backward-compat re-exports of rule ID constants + RIDER_MAP | Vertical |
| **Subtotal** | **7,466** | | |

| Test/Fixture Files | LOC | Purpose |
|---|---|---|
| `postprocessing.test.ts` | 1,074 | Post-processing pipeline tests |
| `registry.test.ts` | 186 | Rule registry validation |
| `regression.test.ts` | 343 | Regression tests for known failure modes |
| 5 fixture files | 205 | Sample contracts for testing |
| **Subtotal** | **1,808** | |

### 2b. Template Files (`lib/templates/`)

| File | LOC | Purpose | H/V |
|------|-----|---------|-----|
| `buyer-msa-template.ts` | 394 | Buyer MSA legal text in HTML | Vertical |
| `publisher-msa-template.ts` | 491 | Publisher MSA legal text in HTML | Vertical |
| `buyer-io-template.ts` | 584 | Buyer Insertion Order | Vertical |
| `publisher-io-template.ts` | 601 | Publisher Insertion Order | Vertical |
| `rider-template.ts` | 448 | Amendment/Rider to MSA or IO | **Horizontal** |
| `invoice-template.ts` | 199 | Branded HTML invoice | **Horizontal** |
| `w9-template.ts` | 470 | IRS Form W-9 | **Horizontal** |
| `w8ben-template.ts` | 404 | IRS Form W-8BEN (non-US individual) | **Horizontal** |
| `w8bene-template.ts` | 497 | IRS Form W-8BEN-E (non-US entity) | **Horizontal** |
| **Subtotal** | **4,088** | | |

### 2c. Signing Infrastructure

| File | LOC | Purpose | H/V |
|------|-----|---------|-----|
| `app/api/sign/[token]/submit/route.ts` | 594 | Core signing endpoint — hash, re-render, webhook, email | Mixed |
| `app/api/sign/[token]/decline/route.ts` | 157 | Decline with webhook | **Horizontal** |
| `app/api/sign/[token]/capture-view/route.ts` | 80 | View tracking (IP/UA/audit trail) | **Horizontal** |
| `app/sign/[token]/page.tsx` | 452 | Public signing page | Mixed |
| `app/sign/[token]/hooks/useDocumentLoader.ts` | 96 | Load document by token | **Horizontal** |
| `app/sign/[token]/hooks/useSigningForm.ts` | 701 | Full form state + canvas signature | Mixed |
| `app/sign/[token]/hooks/useLivePreview.ts` | 192 | Real-time template re-render | Mixed |
| `app/sign/[token]/utils/generateSignedDocument.ts` | 350 | Signed doc with Certificate of Execution | Mixed |
| `components/signing/SigningForm.tsx` | 638 | Three-section signing form | Mixed |
| `components/signing/SigningSidebar.tsx` | 285 | Right-side accordion sidebar | **Horizontal** |
| `components/signing/SigningConfirmation.tsx` | 226 | Post-signing confirmation + download | **Horizontal** |
| `components/signing/SigningOverlay.tsx` | 60 | "Start Signing" CTA overlay | **Horizontal** |
| `components/signing/SigningPreview.tsx` | 55 | Left-side live preview iframe | **Horizontal** |
| `lib/webhooks.ts` | 83 | HMAC-SHA256 webhook signing + retry | **Horizontal** |
| `lib/documents/supabase.ts` | 304 | Partner documents CRUD | **Horizontal** |
| `lib/documents/compliance.ts` | 223 | Compliance tracking (has_msa/has_io flags) | Mixed |
| `lib/documents/storage.ts` | 114 | Supabase Storage helpers | **Horizontal** |
| `lib/companySettings.ts` | 48 | Fetch company settings from DB | **Horizontal** |
| `lib/companySettingsDefaults.ts` | 52 | Default company values (TLP-specific data) | Vertical |
| `lib/taxDocRouting.ts` | 43 | Route tax doc type (W-9/W-8BEN/W-8BEN-E) | **Horizontal** |
| `app/api/s/[code]/route.ts` | 54 | Short URL resolver | **Horizontal** |
| **Subtotal** | **4,857** | | |

### 2d. API Routes — Contract Analysis

| File | LOC | Purpose | H/V |
|------|-----|---------|-----|
| `app/api/contract/analyze/route.ts` | 287 | Streaming analysis via Claude | **Horizontal** |
| `app/api/contract/analyze-async/route.ts` | 199 | Async job creation with hash caching | **Horizontal** |
| `app/api/contract/analyze-process/route.ts` | 253 | Background analysis processor | **Horizontal** |
| `app/api/contract/analyze-status/[jobId]/route.ts` | 238 | Job polling + on-read scoring | **Horizontal** |
| `app/api/contract/ai-assist/route.ts` | 140 | Per-issue AI refinement ("Alex" persona) | Vertical |
| `app/api/contract/help-chat/route.ts` | 148 | VA help chatbot (EN/Tagalog) | Vertical |
| `app/api/contract/extract/route.ts` | 373 | File upload + text extraction (PDF/DOCX/OCR) | **Horizontal** |
| `app/api/contract/extract-fields/route.ts` | 112 | AI IO field extraction | Vertical |
| `app/api/contract/compare-revision/route.ts` | 220 | AI revision comparison | **Horizontal** |
| `app/api/contract/export-redline/route.ts` | 55 | DOCX redline export | Vertical |
| `app/api/contract/translate/route.ts` | 77 | EN-to-Tagalog translation | Vertical |
| `app/api/contract/match-company/route.ts` | 182 | Fuzzy company name matching | Vertical |
| `app/api/contract/analysis/[id]/json/route.ts` | 88 | JSON export of stored analysis | **Horizontal** |
| **Subtotal** | **2,372** | | |

### 2e. API Routes — Bot Integration

| File | LOC | Purpose | H/V |
|------|-----|---------|-----|
| `app/api/bot/contracts/analyze/route.ts` | 277 | Bot contract analysis + webhook | Mixed |
| `app/api/bot/contracts/compare/route.ts` | 469 | Bot revision comparison + webhook | Mixed |
| `app/api/bot/contracts/[entity]/route.ts` | 104 | Entity analysis lookup | Vertical |
| `app/api/bot/contracts/[entity]/status/route.ts` | 108 | Async job status polling | **Horizontal** |
| `app/api/bot/contracts/history/route.ts` | 49 | Recent analyses listing | **Horizontal** |
| `app/api/bot/contracts/rider/route.ts` | 108 | Rider generation from analysis | **Horizontal** |
| `app/api/bot/contracts/rider/[id]/versions/route.ts` | 63 | Rider version history | **Horizontal** |
| **Subtotal** | **1,178** | | |

### 2f. API Routes — Bot Documents

| File | LOC | Purpose | H/V |
|------|-----|---------|-----|
| `app/api/bot/documents/generate/route.ts` | 421 | Generate signing document | Mixed |
| `app/api/bot/documents/route.ts` | 83 | List partner documents | **Horizontal** |
| `app/api/bot/documents/signed/route.ts` | 109 | List signed documents | **Horizontal** |
| `app/api/bot/documents/[id]/route.ts` | 93 | Get document details | Mixed |
| `app/api/bot/documents/[id]/status/route.ts` | 83 | Document status check | **Horizontal** |
| `app/api/bot/documents/[id]/void/route.ts` | 118 | Void a pending document | **Horizontal** |
| `app/api/bot/documents/history/[entityName]/route.ts` | 54 | Document history by entity | **Horizontal** |
| **Subtotal** | **961** | | |

### 2g. API Routes — Negotiations

| File | LOC | Purpose | H/V |
|------|-----|---------|-----|
| `app/api/negotiations/create/route.ts` | 90 | Create negotiation from analysis | **Horizontal** |
| `app/api/negotiations/[id]/finalize/route.ts` | 81 | Finalize negotiation | **Horizontal** |
| `app/api/negotiations/[id]/cancel/route.ts` | 80 | Cancel negotiation | **Horizontal** |
| `app/api/negotiations/[id]/generate-link/route.ts` | 103 | Generate counterparty review link | **Horizontal** |
| `app/api/negotiations/[id]/send-round/route.ts` | 148 | Create new broker round | **Horizontal** |
| `app/api/negotiations/[id]/download/route.ts` | 503 | Download merged agreement (DOCX/HTML) | Mixed |
| `app/api/bot/negotiations/create/route.ts` | 105 | Bot: create negotiation | **Horizontal** |
| `app/api/bot/negotiations/[id]/route.ts` | 96 | Bot: get negotiation state | **Horizontal** |
| `app/api/bot/negotiations/[id]/finalize/route.ts` | 77 | Bot: finalize | **Horizontal** |
| `app/api/bot/negotiations/[id]/generate-link/route.ts` | 95 | Bot: generate review link | **Horizontal** |
| `app/api/bot/negotiations/[id]/download/route.ts` | 499 | Bot: download merged agreement | Mixed |
| **Subtotal** | **1,877** | | |

### 2h. Document Management + Admin

| File | LOC | Purpose | H/V |
|------|-----|---------|-----|
| `app/api/documents/list/route.ts` | 33 | List partner documents | **Horizontal** |
| `app/api/documents/[id]/route.ts` | 77 | Get/delete partner document | **Horizontal** |
| `app/api/documents/upload/route.ts` | 92 | Upload document to storage | **Horizontal** |
| `app/signing/page.tsx` | 200 | Internal signing dashboard | Mixed |
| `netlify/functions/run-analysis-background.mts` | 156 | Netlify BG function for long analysis | **Horizontal** |
| **Subtotal** | **558** | | |

### 2i. Types

| File | LOC | Purpose | H/V |
|------|-----|---------|-----|
| `types/contract-checker.ts` | 339 | ContractIssue, AnalysisResult, PlaybookRule, etc. | Mixed |
| `types/documents.ts` | 80 | PartnerDocument, ComplianceStatus, DocumentType | **Horizontal** |
| `types/negotiations.ts` | 80 | ContractNegotiation, NegotiationRound, NegotiationLink | **Horizontal** |
| `types/database.ts` (signing section) | ~50 | SigningDocument interface | Mixed |
| **Subtotal** | **~549** | | |

### Grand Total: ~25,714 LOC (MOP contract code)

## 3. External Dependencies

| Package | Used By | Purpose |
|---------|---------|---------|
| `@anthropic-ai/sdk` | bot-helpers, 8 API routes | Claude Sonnet 4.5 contract analysis |
| `mammoth` | bot-helpers, extract route | DOCX text extraction |
| `pdfjs-dist` | bot-helpers, extract route | PDF text extraction |
| `tesseract.js` | bot-helpers, extract route | OCR fallback for scanned PDFs |
| `pngjs` | bot-helpers | PNG handling for OCR pipeline |
| `docx` | export.ts, negotiation download | DOCX document generation |
| `jspdf` | export.ts | PDF generation (redline export) |
| `pdf-parse` | contract analyzer | PDF parsing |
| `resend` | sign submit route | Transactional email to signers |
| `@supabase/supabase-js` | All routes, hooks, storage | Database + Storage |
| `crypto` (Node built-in) | sign submit/decline, webhooks | SHA-256 hashing, HMAC signing |
| Google Fonts (Great Vibes) | All templates | Cursive signature font |
| Google Fonts (Roboto) | Tax form templates | Form body font |
| Nominatim (OpenStreetMap) | useSigningForm hook | Address autocomplete |

## 4. Internal Coupling

### What contract code imports from the rest of MOP

Only **3 lightweight imports** from outside the contract domain:

| Import | Source | Used By |
|--------|--------|---------|
| `@/lib/supabase` / `@/lib/supabaseAdmin` | Supabase client instance | All routes, supabase.ts |
| `@/lib/logger` | Logger utility | contract-checker/supabase.ts |
| `@/lib/constants` (`APP_TIMEZONE`) | Timezone constant | contract-checker/supabase.ts |
| `@/lib/apiAuth` (`validateBotApiKey`) | API key validation | All bot/* routes |
| `@/lib/utils/titleCase` | String utility | MSA templates |

The contract system is **remarkably self-contained**. The only meaningful coupling is the Supabase client and API auth — both trivially replaceable.

### Template dispatch duplication

The entity_type + document_type -> render function switch is copy-pasted in **4 locations**:
1. `app/api/bot/documents/generate/route.ts`
2. `app/api/sign/[token]/submit/route.ts`
3. `app/sign/[token]/utils/generateSignedDocument.ts`
4. `app/sign/[token]/hooks/useLivePreview.ts`

A template registry pattern would eliminate this.

## 5. Horizontal Extraction Candidates (Recommended @milo/contracts Scope)

### 5a. Contract Analysis Engine
- Weighted scoring framework (`scoring.ts`) — 444 LOC
- Post-processing pipeline framework (`issue-postprocessing.ts` — engine portion) — ~600 LOC
- Profile multiplier framework (`profiles.ts` — framework portion) — ~150 LOC
- Jurisdiction detection (`jurisdiction.ts`) — 110 LOC
- Text merge engine (`merge.ts`) — 157 LOC
- Text extraction pipeline (PDF/DOCX/TXT/OCR from `bot-helpers.ts`) — ~200 LOC
- Async job orchestration (analyze-async/analyze-process/analyze-status pattern) — ~700 LOC
- Revision comparison engine — 220 LOC
- **Subtotal: ~2,581 LOC**

### 5b. Signing Infrastructure
- Token-based signing flow (generate, view, sign, decline, void lifecycle)
- SHA-256 document/signature hashing
- Audit trail (DocuSign-style JSONB event log)
- HMAC-SHA256 webhook signing with retry
- Short URL resolution
- Certificate of Execution generation
- Signing UI shell (overlay, preview, sidebar, confirmation)
- **Subtotal: ~3,200 LOC**

### 5c. Negotiation Platform
- Multi-round broker/counterparty workflow
- Status state machine (draft -> broker_review -> counterparty_review -> finalized/cancelled)
- Tokenized counterparty review links
- Round management with decision tracking
- Merged agreement download (DOCX/HTML)
- **Subtotal: ~1,800 LOC**

### 5d. Document Management
- Partner document CRUD
- Supabase Storage helpers
- Compliance tracking (has_msa/has_io flags)
- Company settings fetch pattern
- Tax document routing
- **Subtotal: ~800 LOC**

### 5e. Types + Schema
- Horizontal type definitions
- DB migration DDL (horizontal tables)
- **Subtotal: ~500 LOC**

### 5f. Export Framework
- DOCX generation (redline, rider, merged agreement)
- PDF generation framework
- **Subtotal: ~600 LOC**

**Estimated @milo/contracts v0.1.0: ~9,500 LOC**

## 6. Vertical Layer (Stays in scaleflow)

| Component | LOC | Why Vertical |
|-----------|-----|-------------|
| 73 PPC rule definitions (`rules/registry.ts`) | 1,204 | Every rule references TCPA, call recording, lead quality, payout terms |
| PPC system prompt (`system-prompt.ts`) | 355 | A-K publisher checks, A-I buyer checks — all lead-gen-specific |
| IO field mapper (`field-mapper.ts`) | 189 | DID, payout, call_type, billing_model, daily_cap, concurrency |
| "Alex" AI persona (`ai-persona.ts`) | 40 | "Lead Penguin" brand identity |
| Buyer/Publisher MSA templates | 885 | Full legal text referencing leads, campaigns, tracking pixels |
| Buyer/Publisher IO templates | 1,185 | PPC-specific: DID, vertical, payout, hours of operation, states |
| Company settings defaults | 52 | "The Lead Penguin" / Denver / TLP bank details |
| Rule ID constants + RIDER_MAP | 226 | PPC-specific rider section references |
| Tagalog translation endpoint | 77 | VA training feature |
| Help chatbot endpoint | 148 | VA training feature |
| IO field extraction endpoint | 112 | Pay-per-call field vocabulary |
| Channel Edge redline format | 55 | Brand-specific export |
| Entity fuzzy matching | 182 | Matches against publishers/buyers tables |
| **Subtotal** | **~4,710** | |

## 7. The 73-Rule Analyzer: Engine vs Rules Separation

### Architecture (already exists in code, not formalized)

```
HORIZONTAL ENGINE                        VERTICAL RULES
(extractable)                            (stays in scaleflow)
                                        
scoring.ts (444 LOC)                     rules/registry.ts (1,204 LOC)
  - weighted sum framework                - 43 PUB_* rules
  - risk bucket calc (0-100)               - 26 BUY_* rules + 2 aliases
  - two-party consent escalation           - 4 canonical rule_ids
  - normalization                          
                                         system-prompt.ts (355 LOC)
issue-postprocessing.ts (1,112 LOC)        - PPC analysis instructions
  - canonicalization pipeline              - A-K publisher checks
  - severity enforcement                   - A-I buyer checks
  - dedup framework                        
  - reclassification framework           playbook-rules.ts (709 LOC)
  - missed-rule injection framework        - 35 configurable PPC rules
                                           - threshold presets
profiles.ts (288 LOC)                        (full_audit, publisher_defensive,
  - profile multiplier framework              buyer_aggressive, quick_review)
  - auto-escalation lists                  
                                         field-mapper.ts (189 LOC)
jurisdiction.ts (110 LOC)                  - PPC IO field vocabulary
  - US state detection                   
                                         ai-persona.ts (40 LOC)
merge.ts (157 LOC)                         - "Lead Penguin" persona
  - fuzzy text replacement               
                                         rule-ids.ts (226 LOC)
export.ts (1,132 LOC)                      - PPC-specific constants
  - DOCX/PDF framework                  
```

### Is the split clean?

**Moderately clean.** The engine files are generic frameworks that take rule definitions as input. The vertical files are data/config that feed the engine. However:

1. **Post-processing has hardcoded PPC patterns.** `issue-postprocessing.ts` contains 6 regex patterns for detecting "catch-all rejection language" and 6 for "low-trigger indemnity" — these are PPC-specific patterns embedded in the engine layer. They would need to move to a configurable "pattern injection" system.

2. **Scoring has PPC-specific two-party consent logic.** The `twoPartyConsentStates` list and escalation logic in `scoring.ts` is telecom/call-recording-specific. It would need to become a pluggable scoring modifier.

3. **Export has TLP branding.** `export.ts` hardcodes "Lead Penguin" in DOCX headers and watermarks. Needs parameterization via CompanySettings.

4. **6 phantom rule IDs.** The system prompt references 6 rule IDs (`PUB_FIN_REJECTION_CATCH_ALL`, `PUB_FIN_INVOICE_FORFEITURE`, `PUB_LIAB_INDEMNITY_ONE_SIDED`, `PUB_OPS_CROSS_REF_ERROR`, `PUB_OPS_INSURANCE_ADDITIONAL_INSURED`, `PUB_OPS_VENUE_JURY_WAIVER`) that aren't in the registry. These pass through post-processing without registry metadata, falling back to severity-based defaults. This is a gap.

### Rule storage and versioning

- **Storage:** All in code (`RULE_REGISTRY` constant in `rules/registry.ts`). No database, no config file.
- **Versioning:** `RULESET_VERSION = '1.0.0'` and `SCORING_VERSION = '1.0.0'` stored with every analysis. Comment says "do not change existing rule_ids" — strategy is additive. Both versions still at 1.0.0.
- **No migration path** for existing analyses when rules change. Old analyses retain original scores.

### AI usage in the rule engine

**Layered: AI-primary + static guardrails.**

1. **AI-primary (Claude Sonnet 4.5):** The system prompt instructs Claude to analyze contract text and return structured JSON with rule_ids, severities, and explanations. Claude does the actual legal analysis. Model: `claude-sonnet-4-5-20250929`, temp 0.2, 16K tokens.

2. **Static pattern matching (guardrails):** After Claude returns, post-processing runs regex to inject rules Claude missed (catch-all rejection, low-trigger indemnity), detect liability cap context, fix directional errors, and replace aggressive revision language with market-credible templates.

3. **Scoring is purely static** — weighted sums with profile multipliers, no AI.

## 8. Data Model Summary

### Tables

| Table | Rows (freeze manifest) | Critical Data | H/V |
|-------|------------------------|---------------|-----|
| `contract_analyses` | 33 | Yes — all analyzed contracts with issues, summary, AI model used | **Horizontal** |
| `contract_issue_decisions` | ~0 (legacy) | No | **Horizontal** |
| `contract_issue_resolutions` | low | Yes — resolved_by defaults to 'TLP Compliance' | Vertical |
| `contract_analysis_jobs` | low | No — transient job queue | **Horizontal** |
| `finalized_documents` | low | Yes — signed/executed contract storage paths | **Horizontal** |
| `partner_documents` | ~0 | No | **Horizontal** |
| `signing_documents` | 109 | **Yes** — all e-signed documents, audit trails, signatures | Mixed |
| `contract_negotiations` | low | Yes — negotiation state + webhook config | **Horizontal** |
| `negotiation_rounds` | low | Yes — per-round decisions | **Horizontal** |
| `negotiation_links` | low | Yes — tokenized review URLs | **Horizontal** |
| `rider_templates` | low | No — reusable clause library | **Horizontal** |

### Storage Buckets

| Bucket | Files (freeze) | Max Size | Purpose |
|--------|----------------|----------|---------|
| `contract-originals` | 22 | 50 MB | Uploaded PDFs/DOCXs for analysis |
| `finalized-contracts` | 0 (empty) | 50 MB | Signed/executed final docs |
| `partner-documents` | 0 (empty) | 25 MB | Partner document library |

### Multi-tenant readiness

Migration `20300101` (far-future, not applied) adds `tenant_id UUID` to all 10 contract tables and prepares tenant-scoped RLS policies. Currently all RLS is wide-open `USING(true)`.

### Key relationships

```
publishers / buyers
  |
  +--< contract_analyses (publisher_id, buyer_id)
  |     |
  |     +--< contract_analyses (parent_analysis_id — self-ref version chain)
  |     +--< contract_issue_decisions (analysis_id)
  |     +--< contract_issue_resolutions (analysis_id)
  |     +--< contract_analysis_jobs (analysis_id)
  |     +--< finalized_documents (analysis_id)
  |     +--< contract_negotiations (analysis_id)
  |           |
  |           +--< negotiation_rounds (negotiation_id)
  |           +--< negotiation_links (negotiation_id)
  |
  +--< partner_documents (publisher_id / buyer_id, contract_analysis_id)
  +--< signing_documents (entity_id — polymorphic, no FK constraint)
```

`signing_documents.entity_id` is a plain UUID with no FK constraint — this makes extraction cleaner since it's not tied to publishers/buyers schema.

## 9. AI Usage Summary

Claude is called in **8 distinct endpoints**, all using `claude-sonnet-4-5-20250929`:

| Endpoint | Purpose | Max Tokens | Temp | Tier |
|----------|---------|-----------|------|------|
| `/api/contract/analyze` | Full analysis (streaming) | 16K | 0.2 | Judgment |
| `/api/contract/analyze-process` | Full analysis (async) | 16K | 0.2 | Judgment |
| `run-analysis-background.mts` | Full analysis (Netlify BG) | 16K | 0.0 | Judgment |
| `/api/contract/compare-revision` | Revision diff verification | 8K | 0.2 | Judgment |
| `/api/contract/extract-fields` | IO field extraction | 1K | 0.1 | Structured |
| `/api/contract/ai-assist` | Per-issue AI refinement | 1.5K | 0.4 | Judgment |
| `/api/contract/help-chat` | VA help chatbot | 500 | 0.7 | Classify |
| `/api/contract/translate` | EN-to-Tagalog translation | 1.5K | 0.3 | Structured |

Bot routes (`/api/bot/contracts/analyze`, `/api/bot/contracts/compare`) call Claude indirectly via `bot-helpers.runContractAnalysis()`.

**Implication for @milo/ai-client:** The analysis engine (judgment tier) is the heaviest AI consumer. Extraction should use `@milo/ai-client` for all Claude calls, with the system prompt injected as vertical config.

## 10. Recommended Extraction Sequence

### Phase 1: Core Engine (`@milo/contracts` v0.1.0)

1. **Types first** — extract horizontal type definitions (ContractIssue, AnalysisResult, NegotiationRound, etc.) into `@milo/contracts/types`
2. **Scoring engine** — `scoring.ts` with parameterized two-party consent (move PPC states to config)
3. **Post-processing pipeline** — extract framework, move PPC-specific regex patterns to a pluggable pattern config
4. **Jurisdiction detection** — already fully generic
5. **Text merge engine** — already fully generic
6. **Text extraction** — PDF/DOCX/OCR pipeline from `bot-helpers.ts`

### Phase 2: Signing Platform

7. **Signing lifecycle** — token generation, view tracking, signature capture, audit trail, HMAC webhooks
8. **Template registry** — abstract the dispatch pattern, templates become vertical plugins
9. **Tax document routing** — already generic
10. **Short URL resolution** — already generic

### Phase 3: Negotiation Platform

11. **Negotiation state machine** — rounds, links, status transitions
12. **Merged agreement generation** — DOCX/HTML download with parameterized branding
13. **Document management** — CRUD, storage, compliance tracking

### Phase 4: DB Schema

14. **Horizontal table migrations** — all tables except `contract_issue_resolutions` (needs resolved_by parameterization)
15. **Storage bucket setup** — contract-originals, finalized-contracts, partner-documents

## 11. Open Questions for Mark

1. **milo-ops parallel system.** milo-ops has its own contract analysis system (~3,500 LOC in `src/lib/tools/`, `src/lib/contract-analysis-prompt.ts`, `src/app/api/contract/`) with its own `contract_documents` table and evolved rule set (60+ rules as inline prompt text). This is a **full duplicate** of MOP's contract analyzer with PPC-specific enhancements. **Decision needed:** Does `@milo/contracts` extraction pull from MOP, milo-ops, or reconcile both? The milo-ops version is the operational one (it's what Milo uses); MOP's is the original with the formal rule registry.

2. **Rule engine extraction boundary.** The post-processing pipeline has 12 PPC-specific regex patterns (catch-all rejection, low-trigger indemnity) hardcoded into the engine layer. Clean extraction requires a "pattern injection" extension point. Is this worth the effort for v0.1.0, or should the engine ship with a "PPC patterns included" flag and defer the plugin system?

3. **6 phantom rule IDs.** The system prompt references 6 rule IDs not in the registry (`PUB_FIN_REJECTION_CATCH_ALL`, `PUB_FIN_INVOICE_FORFEITURE`, `PUB_LIAB_INDEMNITY_ONE_SIDED`, `PUB_OPS_CROSS_REF_ERROR`, `PUB_OPS_INSURANCE_ADDITIONAL_INSURED`, `PUB_OPS_VENUE_JURY_WAIVER`). Should these be added to the registry before extraction, or cleaned up?

4. **Signing document templates vs database.** All 8 templates are hardcoded HTML in TypeScript functions. For multi-vertical support, should MSA/IO legal text move to a database-driven template system, or should each vertical just provide its own render functions?

5. **CompanySettings.** Defaults are hardcoded to "The Lead Penguin". The `company_settings` DB table already exists and is used at runtime. Should `@milo/contracts` require `vertical.config.ts` for defaults (aligned with Thesis 2.5), or keep the DB-driven approach?

6. **HMAC webhook inconsistency.** The submit route uses `signBody()` (timestamp.body format) but the decline route constructs its own HMAC (body-only format). A consumer verifying both would need two different signature formats. Fix before or during extraction?

7. **Signed HTML storage.** Signed documents are stored as full HTML in the `rendered_html` TEXT column — no PDFs generated server-side. Is this acceptable for `@milo/contracts`, or should server-side PDF generation be added?

8. **Multi-tenant migration.** The `20300101` migration already adds `tenant_id` to all contract tables. Should `@milo/contracts` ship multi-tenant from v0.1.0, or defer?

## 12. Estimated LOC for @milo/contracts v0.1.0

| Component | LOC |
|-----------|-----|
| Types + interfaces | 500 |
| Scoring engine (parameterized) | 500 |
| Post-processing pipeline (framework) | 700 |
| Jurisdiction detection | 110 |
| Text merge engine | 157 |
| Text extraction (PDF/DOCX/OCR) | 400 |
| Signing lifecycle | 1,500 |
| Template registry (abstract) | 200 |
| Tax document routing | 43 |
| Short URL resolution | 54 |
| HMAC webhook signing | 83 |
| Negotiation state machine | 800 |
| Document management | 500 |
| Export framework (DOCX/PDF) | 600 |
| DB migrations (horizontal tables) | 400 |
| Async job orchestration | 700 |
| Storage helpers | 114 |
| Compliance tracking | 200 |
| Company settings | 50 |
| Package config (package.json, tsconfig, README) | ~100 |
| **Total** | **~7,711** |

Note: This is lower than the ~9,500 earlier estimate because several "Mixed" files will be split during extraction — only the horizontal portions ship. Test code adds ~1,800 LOC on top.

## Appendix A: Secondary Repo Findings

### PPCRM — Pure API Consumer (zero local contract logic)

- `supabase/functions/signature-webhook/index.ts` (419 LOC): Webhook handler for MOP signing events. Updates `partners` + `onboarding_checklists`. Posts Teams MessageCards.
- `supabase/functions/onboardbot/negotiate.ts` (508 LOC): Pure MOP API client — calls 6 MOP endpoints for analysis, negotiation, and document generation.
- `supabase/functions/_shared/mop-client.ts`: General MOP API client with 6 contract-specific functions.

**Verdict:** Integration glue only. Stays in PPC vertical.

### milo-ops — Full Parallel Contract System (~3,500 LOC)

- `src/lib/contract-analysis-prompt.ts` (495 LOC): 60+ rules as inline prompt text (superset of contract-checker)
- `src/lib/tools/analyze-contract.ts` (364 LOC): Milo tool with direct Claude Opus calls
- `src/lib/tools/ingest-returned-contract.ts` (334 LOC): Versioned contract ingestion
- `src/lib/tools/generate-redlined-version.ts` (290 LOC): Redline generation via Claude
- `src/lib/tools/publish-contract-for-buyer-review.ts` (170 LOC): Contract publishing with short slugs
- `src/app/contract-review/[id]/page.tsx` (1,603 LOC): Full contract review UI
- `src/app/api/contract/analyze-bg/route.ts` (379 LOC): Background analysis endpoint
- Own `contract_documents` table with versioning pipeline

**Verdict:** Highest overlap risk. The milo-ops version has diverged from MOP's contract-checker with PPC-specific rule IDs (PUB_*/BUY_*) and operational features not in contract-checker (plain_english/impact_to_us output, MSA improvement suggestions, template comparison, integrated negotiation). Reconciliation needed.

### contract-checker (standalone, `~/contract-checker/`) — Ancestor

~10,858 LOC. Next.js 14 app. 30+ configurable playbook rules with threshold profiles. Unique features not in milo-ops: Tagalog translations, DOCX export, revision comparison with "sneaked in" detection, playbook presets. Mark's direction: "very PPC specific" — belongs in PPC vertical, not milo-engine.

### mysecretary + milo-outreach — Clean (zero contract code)

## Appendix B: Template Variable Schemas

### Buyer MSA: `BuyerMsaVariables`
`effective_date`, `advertiser_name`, `advertiser_address`, `advertiser_city`, `advertiser_state`, `advertiser_zip`

### Publisher MSA: `PublisherMsaVariables`
`effective_date`, `publisher_name`, `publisher_address`, `publisher_city`, `publisher_state`, `publisher_zip`

### IO (both buyer and publisher): shared shape
`io_number`, `effective_date`, `client_name`/`publisher_name`, `contact_name`, `contact_email`, `contact_phone`, `did`, `vertical`, `call_type`, `payout`, `billing_model`, `payout_min`, `payout_max`, `duration`, `daily_cap`, `cc`, `bill_cycle`, `hours_of_operation`/`hoo_structured`, `states`, `qualifiers`/`qualifiers_notes`, `additional_terms`

### CompanySettings (26 fields, used across all templates)
Branding: `company_name`, `legal_name`, `branding_color` (#dd72a6 default)
Owner: `owner_name`, `owner_title`, `owner_email`
Address: `address`, `city`, `state`, `zip`, `phone`
Financial: `bank_name`, `bank_account_number`, `bank_routing_number`, etc.
Emails: `accounting_email`, `onboarding_email`
