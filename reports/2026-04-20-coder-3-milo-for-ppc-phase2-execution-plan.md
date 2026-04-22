---
directive: "Milo-for-PPC Phase 2 execution plan"
lane: research
coder: Coder-3
directive-number: 54
started: 2026-04-20
completed: 2026-04-20
depends-on:
  - reports/2026-04-20-coder-3-milo-for-ppc-ui-port-plan.md (f0c2160)
  - reports/2026-04-20-coder-3-open-mark-decisions.md (429f64b — D64 batch resolution)
  - reports/2026-04-20-coder-3-milo-for-ppc-rule-pack-spec.md (e487198)
  - reports/2026-04-20-coder-3-milo-for-ppc-scaffolding-plan.md (0dc5e0b)
---

# Milo-for-PPC Phase 2 Execution Plan

## Executive Summary

| Metric | Value |
|--------|-------|
| Total new LOC | ~5,250 |
| Total commits | ~38 (2-6 per UI, plus pre/post work) |
| Track A (Analysis) | 5 UIs, ~2,150 LOC |
| Track B (Signing/Negotiation) | 5 UIs, ~3,000 LOC |
| ~~Pre-Phase-2 work~~ | ~~1 task (~100 LOC)~~ CANCELLED (Decision 67) |
| Post-Phase-2 work | 2 tasks (~50 LOC) |
| Estimated calendar | ~2 weeks parallel (Track A + B), ~3.5 weeks sequential |

**Prerequisite:** Phase 1 scaffold complete (Coder-1 D49). The forked milo-ops repo exists as `milo-for-ppc`, @milo/* packages wired, PPC rules plugin created, `vertical.config.ts` configured, build passes, auth works.

**All open decisions locked (D64).** No blocking questions remain.

---

## ~~Pre-Phase-2: Publisher/Buyer ID Reconciliation Table~~ CANCELLED (Decision 67)

~~Per D-OPEN-13 (resolved D53): build the mapping table BEFORE Phase 2 UI work.~~

**CANCELLED:** Directive #58 stress-test found display names are already denormalized in migrated data. MOP's own UIs never JOIN to publishers/buyers — all 5 target pages use inline name fields. No mapping table needed. Decision 67 locked Option C.

### Display Name Fallback Chain (replaces mapping table)

Phase 2 UIs resolve counterparty names using this priority:

**analysis_records (33 rows):**
1. `metadata.extracted_company_name` — set by migration script, most reliable
2. `counterparty_info?.name` — AI-extracted during original analysis, varies by record
3. `display_name` column — human-set, may be document name not entity name
4. `document_name` — always present, last resort

**signing_documents (109 rows):**
1. `metadata.entity_name` — set by migration script from MOP's `entity_name` column
2. `counterparty_company` column — direct column, migrated from MOP
3. `counterparty_name` column — direct column, signer name

**negotiation_records (2 rows):**
1. `counterparty_name` — direct column, migrated from MOP

### Implementation
3-line display function per primitive. No table, no backfill, no cross-database query. Coder-1/Coder-2 implement inline during Track A/B commits.

---

## Track A: Contract Analysis UIs

Track A builds the analysis workflow: upload → results → history → settings → help → export. All depend on @milo/contract-analysis + PPC rules plugin (both from Phase 1).

---

### A1: Shared Contract Components (~500 LOC)

Build shared components used across all Track A and some Track B UIs. Must be first.

#### Commits

| # | Commit | LOC | Description |
|---|--------|-----|-------------|
| A1-1 | `feat: SeverityBadge + displayConfig filter` | ~100 | `src/components/contract/SeverityBadge.tsx` (CRITICAL=red, HIGH=orange, MEDIUM=yellow, LOW=blue). `src/lib/display-config.ts` with `filterAnalysisForDisplay()` per UI port plan §7. |
| A1-2 | `feat: IssueCard component` | ~250 | `src/components/contract/IssueCard.tsx` — expandable card with severity badge, plain English explanation, original clause, suggested revision, legal basis, negotiation text. Reference: `~/contract-checker/src/components/results/issue-card.tsx`. Styled with milo-ops Tailwind patterns (no shadcn). |
| A1-3 | `feat: IssueSidebar + RoleSelector + JurisdictionSelector` | ~150 | `src/components/contract/IssueSidebar.tsx` (jump-to nav). `src/components/contract/RoleSelector.tsx` (publisher/buyer). `src/components/contract/JurisdictionSelector.tsx` (two-party consent states). Reference: `~/contract-checker/src/components/upload/` and `~/contract-checker/src/components/results/issue-sidebar.tsx`. |

#### File Mapping

| Source | Target |
|--------|--------|
| `~/contract-checker/src/components/results/issue-card.tsx` (350 LOC) | `src/components/contract/IssueCard.tsx` (rebuild, ~250 LOC) |
| `~/contract-checker/src/components/results/issue-sidebar.tsx` (176 LOC) | `src/components/contract/IssueSidebar.tsx` (rebuild, ~100 LOC) |
| `~/contract-checker/src/components/upload/role-selector.tsx` | `src/components/contract/RoleSelector.tsx` (rebuild, ~50 LOC) |
| `~/contract-checker/src/components/upload/jurisdiction-selector.tsx` | `src/components/contract/JurisdictionSelector.tsx` (rebuild, ~50 LOC) |
| (new) | `src/components/contract/SeverityBadge.tsx` (~50 LOC) |
| (new) | `src/lib/display-config.ts` (~50 LOC) |

#### Config Changes (vertical.config.ts)

```typescript
analysis: {
  displayAggregateScore: false,
  displayLetterGrade: false,
  displayRiskBucket: false,
  issueOrdering: 'severity-desc',
  perIssueFields: ['severity', 'plainEnglish', 'impact', 'recommendation', 'riderRef'],
}
```

#### Dependencies
- Phase 1 complete (vertical.config.ts exists with analysis section)
- PPC rules plugin (for severity values)

#### Test Plan
- IssueCard renders all 4 severity levels with correct colors
- displayConfig filter strips score/grade/bucket from analysis results
- IssueCard expands/collapses, shows all per-issue fields
- IssueSidebar lists issues, clicking scrolls to card

#### Risk: LOW

---

### A2: Contract Analysis Upload — P4 (~600 LOC)

Core feature. Operators upload contracts and get analysis results.

#### Commits

| # | Commit | LOC | Description |
|---|--------|-----|-------------|
| A2-1 | `feat: contract analysis API route` | ~200 | `src/app/api/contract-analysis/route.ts` — POST handler. Accepts file upload + role + jurisdiction + profile. Calls `@milo/contract-analysis` `createAnalysis()` with PPC rule pack. Returns analysis_record ID. |
| A2-2 | `feat: contract analysis upload page` | ~250 | `src/app/contract-analysis/page.tsx` — file drop zone, RoleSelector, JurisdictionSelector, profile auto-detect (D-OPEN-6: auto-detect with manual override), submit button, loading state, results display using IssueCard + IssueSidebar. |
| A2-3 | `feat: analysis polling + results display` | ~150 | Async job polling for long analyses. Results page with severity-ordered issue list. Dashboard ContractAnalysisDisplay block wiring. |

#### File Mapping

| Source | Target |
|--------|--------|
| MOP `app/contract-analyzer/page.tsx` (1,465 LOC) | `src/app/contract-analysis/page.tsx` (~250 LOC — stripped to essentials) |
| MOP `app/api/contract/analyze-async/route.ts` | `src/app/api/contract-analysis/route.ts` (~200 LOC — rewired to @milo/*) |
| MOP `app/api/contract/analyze-status/[jobId]/route.ts` | `src/app/api/contract-analysis/[jobId]/status/route.ts` (~50 LOC) |
| milo-ops `src/lib/contract-analysis-prompt.ts` (496 LOC) | `src/lib/ppc/analysis-prompt.ts` (ported per D-OPEN-1: hand-crafted Phase 1) |

#### Data Rewiring
- MOP `contract_analyses` → shared Supabase `analysis_records` via `@milo/contract-analysis` client
- MOP `analysis_jobs` → shared Supabase `analysis_jobs` via client
- MOP `publishers`/`buyers` entity lookup → `@milo/crm` `crm_counterparties`
- All queries scoped with `tenant_id = 'tlp'`

#### Config Changes
- `vertical.config.ts` already has `analysis` section (from A1)
- Add `analysis.defaultProfile: 'SCALE'` (auto-selected for PPC operators)
- Add `analysis.model: TIER.structured` (Sonnet per D-OPEN-2)

#### Display Config
- `displayAggregateScore: false` — no score wheel
- `displayLetterGrade: false` — no letter grade
- `displayRiskBucket: false` — no risk bucket label
- Issues ordered severity-desc, CRITICAL first

#### Test Plan
- Upload PDF → analysis job created → results display within 30s
- Publisher role → publisher checks (A-K) executed
- Buyer role → buyer checks (A-I) executed
- Two-party consent state → jurisdiction rules fire
- No score/grade/bucket visible in results
- Issue count matches expected range for test contracts (use 3 of the 33 migrated analyses as fixture comparison)

#### Risk: MEDIUM
- @milo/contract-analysis client SDK may need tweaks for async job flow
- Fallback: direct Supabase queries (same pattern as Phase 1)

---

### A3: Export Toolbar — R4 (~300 LOC)

Operators export analysis results to share with counterparties or legal.

#### Commits

| # | Commit | LOC | Description |
|---|--------|-----|-------------|
| A3-1 | `feat: export toolbar + DOCX export` | ~200 | `src/components/contract/ExportToolbar.tsx` + `src/lib/export.ts`. DOCX export using `docx` package (contract-checker pattern). JSON export for API consumption. |
| A3-2 | `feat: wire export into analysis results` | ~100 | Add ExportToolbar to P4 results display. Export format excludes aggregate score per display config. |

#### File Mapping

| Source | Target |
|--------|--------|
| `~/contract-checker/src/components/results/export-toolbar.tsx` (199 LOC) | `src/components/contract/ExportToolbar.tsx` (~150 LOC) |
| `~/contract-checker/src/lib/export.ts` (224 LOC) | `src/lib/export.ts` (~150 LOC — DOCX + JSON, no PDF per D-OPEN-22) |

#### Dependencies
- A2 complete (analysis results exist to export)
- `docx` npm package added (Phase 1 or first commit here)

#### Display Config
- Exports exclude score section entirely
- Issue list in severity-desc order

#### Test Plan
- Export DOCX → opens in Word/Google Docs with correct formatting
- Export JSON → valid JSON with all issue fields
- Neither export format contains score/grade/bucket

#### Risk: LOW

---

### A4: Analysis History — R1 (~500 LOC)

Operators find past analyses, compare revisions, follow up via Teams.

#### Commits

| # | Commit | LOC | Description |
|---|--------|-----|-------------|
| A4-1 | `feat: analysis history list page` | ~200 | `src/app/contract-analysis/history/page.tsx` — search + filter, active/archived tabs, per-item severity count + top severity badge. Queries `analysis_records` via @milo/contract-analysis client. Entity name via display name fallback chain (Decision 67). |
| A4-2 | `feat: inline actions (rename, archive, delete)` | ~100 | Inline rename, archive/restore, permanent delete with confirmation. Auto-cleanup stats display. |
| A4-3 | `feat: revision comparison modal` | ~200 | Upload revised contract, diff against previous version. 4-column summary: Implemented / Partially / Not Done / Unexpected. Uses milo-ops `contract_documents.diff_from_previous` field (D-OPEN-8: milo-ops richer schema). Teams message generator for follow-up. |

#### File Mapping

| Source | Target |
|--------|--------|
| `~/contract-checker/src/app/history/page.tsx` (1,630 LOC) | `src/app/contract-analysis/history/page.tsx` (~500 LOC — rebuilt with milo-ops patterns) |
| MOP `app/contract-analyzer/history/page.tsx` (507 LOC) | Reference only — contract-checker version is better |

#### Data Rewiring
- `analysis_records` on shared Supabase via @milo/contract-analysis client
- Entity names via display name fallback chain (Decision 67 — no mapping table needed)
- Version tracking via `contract_group_id` + `parent_version_id` + `version` (D-OPEN-8)

#### Display Config
- History list shows issue count + top severity per item (no aggregate score)
- Revision comparison shows per-issue status change, no scores

#### Test Plan
- List shows all 33 migrated analyses with correct entity names
- Search filters by document name
- Archive moves to archived tab, restore moves back
- Revision comparison correctly diffs two analyses in same group
- Teams message generates correctly formatted follow-up text

#### Risk: LOW-MEDIUM
- ~~entity_id_mapping depends on PRE-1 completing first~~ REMOVED (Decision 67)
- Revision comparison depends on having 2+ analyses in the same contract_group (may need to create test data)

---

### A5: Help/KB Panel — R3 (~400 LOC)

AI-powered help sidebar available on all contract pages.

#### Commits

| # | Commit | LOC | Description |
|---|--------|-----|-------------|
| A5-1 | `feat: help knowledge base (PPC content)` | ~200 | `src/lib/help-knowledge.ts` — PPC-specific KB entries: pay-per-call terms, TCPA specifics, clawback/holdback definitions, lead quality metrics. 30+ glossary terms. 6 suggested questions. |
| A5-2 | `feat: HelpPanel + HelpTooltip components` | ~150 | `src/components/help/HelpPanel.tsx` — slide-in sheet with AI chat, suggested questions, per-issue "AI Assist" quick actions. `src/components/help/HelpTooltip.tsx` — glossary tooltips. Uses @milo/ai-client (TIER.classify for fast responses per D28). |
| A5-3 | `feat: wire help into contract pages` | ~50 | Add HelpPanel toggle to P4 analysis, R1 history. Add HelpTooltip to IssueCard for key terms. |

#### File Mapping

| Source | Target |
|--------|--------|
| `~/contract-checker/src/lib/help-knowledge.ts` (316 LOC) | `src/lib/help-knowledge.ts` (~200 LOC — rewritten for PPC domain) |
| `~/contract-checker/src/components/help/help-panel.tsx` (269 LOC) | `src/components/help/HelpPanel.tsx` (~150 LOC — milo-ops styling) |
| `~/contract-checker/src/components/help/help-tooltip.tsx` (72 LOC) | `src/components/help/HelpTooltip.tsx` (~50 LOC) |

#### Dependencies
- @milo/ai-client (already consumed by milo-ops — no new wiring)

#### Test Plan
- Panel opens/closes, persists chat history within session
- Suggested questions produce relevant answers
- "AI Assist" on an issue generates useful revision/explanation
- Glossary tooltips show definitions on hover

#### Risk: LOW

---

### A6: Settings/Playbook — R2 (~400 LOC)

Operators customize rule sensitivity. Not blocking — defaults work.

#### Commits

| # | Commit | LOC | Description |
|---|--------|-----|-------------|
| A6-1 | `feat: settings page with profile selector` | ~200 | `src/app/settings/contract-analysis/page.tsx` — profile selector (4 presets: Full Audit, Publisher Defensive, Buyer Aggressive, Quick Review). Rule list grouped by category (6 groups). Per-rule severity badge + applies-to badge. Stats: "X of Y active, Z critical". D-OPEN-7: profiles only Phase 1. |
| A6-2 | `feat: rule enable/disable + localStorage persistence` | ~150 | Per-rule toggle (except PPC_MANDATORY_CHECKS — locked on). Threshold display (read-only Phase 1). Save to localStorage. |
| A6-3 | `feat: wire settings into analysis upload` | ~50 | Analysis upload reads active rule set from localStorage. If no settings saved, use SCALE profile defaults. |

#### File Mapping

| Source | Target |
|--------|--------|
| `~/contract-checker/src/app/settings/page.tsx` (515 LOC) | `src/app/settings/contract-analysis/page.tsx` (~200 LOC) |
| `~/contract-checker/src/lib/playbook-rules.ts` (746 LOC) | PPC rules from Phase 1 `src/lib/ppc/rules.ts` (already exists) |

#### Data Rewiring
- 77 rules from PPC rules plugin (45 base + 32 PPC)
- Profiles from `src/lib/ppc/profiles.ts` (Phase 1)
- Persistence: localStorage (D-OPEN-7: profiles Phase 1, per-rule Phase 2)

#### Test Plan
- Profile selection changes active rule set
- Mandatory checks cannot be disabled (locked toggle)
- Settings persist across page reloads (localStorage)
- Analysis upload respects current settings

#### Risk: LOW

---

## Track B: Signing & Negotiation UIs

Track B builds the document lifecycle: generate → sign → negotiate → review. Depends on @milo/contract-signing and @milo/contract-negotiation (both from Phase 1 wiring).

---

### B1: Signing Dashboard — P6 (~200 LOC)

Operator overview of document pipeline. Build first to validate signing data layer.

#### Commits

| # | Commit | LOC | Description |
|---|--------|-----|-------------|
| B1-1 | `feat: signing dashboard page` | ~150 | `src/app/signing/page.tsx` — list all signing_documents with status filters (pending/signed/declined/expired/voided). Table with columns: document name, counterparty, type, status, created, actions. Queries shared Supabase via @milo/contract-signing client. |
| B1-2 | `feat: signing dashboard actions` | ~50 | Per-row actions: view, void (with confirmation), copy signing link. Generate new document button (links to future generate flow or existing bot endpoint). |

#### File Mapping

| Source | Target |
|--------|--------|
| MOP `app/signing/page.tsx` (200 LOC) | `src/app/signing/page.tsx` (~200 LOC — rewired) |

#### Data Rewiring
- MOP `signing_documents` → shared Supabase `signing_documents` via `@milo/contract-signing` client
- Entity names: `signing_documents.metadata.source_entity_name` or entity_id_mapping → crm_counterparties
- All queries: `tenant_id = 'tlp'`

#### Config Changes
- Add `signing.baseUrl` to vertical.config.ts: `process.env.NEXT_PUBLIC_APP_URL` (Vercel preview URL Phase 1, `tlp.justmilo.app` Phase 3)

#### Display Config
- No analysis results on this page (N/A)

#### Test Plan
- Dashboard shows all 109 migrated signing documents
- Status filter works (68 pending, 23 signed, 13 viewed, 5 voided)
- Void action updates status
- Copy link generates correct signing URL using Milo-for-PPC domain

#### Risk: LOW

---

### B2: Signing Page (Public) — P1 (~1,200 LOC)

Public page where counterparties sign documents. Highest LOC, most complex.

#### Commits

| # | Commit | LOC | Description |
|---|--------|-----|-------------|
| B2-1 | `feat: signing page shell + token resolution` | ~200 | `src/app/signing/[token]/page.tsx` — token → document resolution via @milo/contract-signing `resolveToken()`. Loading state, expired/voided/already-signed states. No auth (public, token-validated). |
| B2-2 | `feat: SigningForm component (port)` | ~350 | `src/components/signing/SigningForm.tsx` — personal info fields, company details, signature capture (typed + drawn), E-SIGN consent. Port from MOP `components/signing/SigningForm.tsx` (638 LOC), strip MOP cruft, restyle with Milo-for-PPC branding. |
| B2-3 | `feat: SigningSidebar + SigningPreview` | ~150 | `src/components/signing/SigningSidebar.tsx` — 3-step accordion (info, company, signature). `src/components/signing/SigningPreview.tsx` — document preview iframe. Port from MOP. |
| B2-4 | `feat: signing API routes (submit, decline, capture-view)` | ~300 | `src/app/api/signing/[token]/submit/route.ts` → @milo/contract-signing `submitSignature()`. `decline/route.ts` → `declineDocument()`. `capture-view/route.ts` → view analytics. Uses unified HMAC (D43 fix). |
| B2-5 | `feat: SigningConfirmation + generateSignedDocument` | ~150 | `src/components/signing/SigningConfirmation.tsx` — post-signature audit trail, print/download. `src/lib/signing/generateSignedDocument.ts` — Certificate of Execution assembly. D-OPEN-22: HTML only, no PDF. |
| B2-6 | `feat: signing page branding + CompanySettings` | ~50 | Branding from vertical.config.ts (D-OPEN-9). Company details from DB with vertical.config.ts fallback. TLP logo, colors, legal name. |

#### File Mapping

| Source | Target |
|--------|--------|
| MOP `app/sign/[token]/page.tsx` (452 LOC) | `src/app/signing/[token]/page.tsx` (~200 LOC) |
| MOP `app/sign/[token]/hooks/useSigningForm.ts` (701 LOC) | `src/hooks/useSigningForm.ts` (~300 LOC — simplified) |
| MOP `app/sign/[token]/hooks/useDocumentLoader.ts` (96 LOC) | `src/hooks/useDocumentLoader.ts` (~60 LOC) |
| MOP `app/sign/[token]/hooks/useLivePreview.ts` (192 LOC) | `src/hooks/useLivePreview.ts` (~100 LOC) |
| MOP `components/signing/SigningForm.tsx` (638 LOC) | `src/components/signing/SigningForm.tsx` (~350 LOC) |
| MOP `components/signing/SigningSidebar.tsx` (285 LOC) | `src/components/signing/SigningSidebar.tsx` (~100 LOC) |
| MOP `components/signing/SigningConfirmation.tsx` (226 LOC) | `src/components/signing/SigningConfirmation.tsx` (~100 LOC) |
| MOP `components/signing/SigningPreview.tsx` (56 LOC) | `src/components/signing/SigningPreview.tsx` (~50 LOC) |
| MOP `components/signing/SigningOverlay.tsx` (61 LOC) | (merged into page.tsx) |
| MOP `app/sign/[token]/utils/generateSignedDocument.ts` (350 LOC) | `src/lib/signing/generateSignedDocument.ts` (~150 LOC) |
| MOP `app/api/sign/[token]/submit/route.ts` (495 LOC) | `src/app/api/signing/[token]/submit/route.ts` (~150 LOC) |
| MOP `app/api/sign/[token]/decline/route.ts` (115 LOC) | `src/app/api/signing/[token]/decline/route.ts` (~80 LOC) |
| MOP `app/api/sign/[token]/capture-view/route.ts` (71 LOC) | `src/app/api/signing/[token]/capture-view/route.ts` (~50 LOC) |

#### Data Rewiring
- MOP `signing_documents` → shared Supabase via @milo/contract-signing client
- Token resolution: MOP HMAC → @milo/contract-signing `resolveToken(token)`
- Entity lookup: `publishers`/`buyers` → `crm_counterparties` via @milo/crm
- CompanySettings: vertical.config.ts defaults + DB `company_settings` override (D-OPEN-9)
- Webhook: PPCRM endpoint → Milo-for-PPC webhook URL (D-OPEN-23: old rows historical)
- Template dispatch: D-OPEN-3 plugin interface — signing package lifecycle, vertical render functions

#### Config Changes
```typescript
signing: {
  baseUrl: process.env.NEXT_PUBLIC_APP_URL,
  tokenExpiryDays: 30,
  webhookUrl: `${process.env.NEXT_PUBLIC_APP_URL}/api/webhooks/signing`,
},
company: {
  name: "TLP Compliance",
  legalName: "The Lead Penguin LLC",
  ownerName: "Tiffani Kinkennon",
  ownerTitle: "Managing Member",
  address: "1500 N Grant St, STE R, Denver, CO 80203",
  brandingColor: "#dd72a6",
}
```

#### Display Config
- N/A — no analysis results on signing pages

#### Test Plan
- Valid token → document loads, 3-step form renders
- Expired token → "Token expired" message
- Already-signed token → "Already signed" confirmation with audit trail
- Submit signature → document status changes to 'signed', audit trail updated
- Decline → status changes to 'declined'
- Branding: TLP logo + colors rendered (not MOP)
- Tax form routing: W-9/W-8BEN/W-8BEN-E dispatch correctly based on country + entity

#### Risk: HIGH
- Most complex UI with most moving parts
- Template dispatch needs the plugin interface (D-OPEN-3) — if @milo/contract-signing v0.2 client SDK isn't ready, fallback to direct Supabase queries
- Tax form templates must be ported (D-OPEN-4: horizontal) — requires W-9/W-8BEN/W-8BEN-E HTML templates in @milo/contract-signing

---

### B3: Negotiation Admin — P2 (~800 LOC)

Operator creates and manages negotiations from analysis results.

#### Commits

| # | Commit | LOC | Description |
|---|--------|-----|-------------|
| B3-1 | `feat: negotiation list page` | ~200 | `src/app/negotiation/page.tsx` — list all negotiations with status filters (active/completed/cancelled/finalized). Table: counterparty, analysis link, round count, status, created. |
| B3-2 | `feat: negotiation detail page` | ~250 | `src/app/negotiation/[id]/page.tsx` — round history timeline, linked analysis results (using IssueCard from A1), current round form. |
| B3-3 | `feat: negotiation API routes` | ~200 | `src/app/api/negotiation/create/route.ts`, `[id]/send-round/route.ts`, `[id]/finalize/route.ts`, `[id]/generate-link/route.ts`, `[id]/download/route.ts`, `[id]/cancel/route.ts`. All rewired to @milo/contract-negotiation client. |
| B3-4 | `feat: negotiation download (HTML export)` | ~150 | HTML export of negotiation history (D-OPEN-22: no PDF). Full round history with decisions and suggested revisions. |

#### File Mapping

| Source | Target |
|--------|--------|
| MOP `app/negotiations/page.tsx` + `[id]/page.tsx` (~1,115 LOC) | `src/app/negotiation/page.tsx` + `[id]/page.tsx` (~450 LOC) |
| MOP `app/api/negotiations/create/route.ts` | `src/app/api/negotiation/create/route.ts` |
| MOP `app/api/negotiations/[id]/send-round/route.ts` | `src/app/api/negotiation/[id]/send-round/route.ts` |
| MOP `app/api/negotiations/[id]/finalize/route.ts` | `src/app/api/negotiation/[id]/finalize/route.ts` |
| MOP `app/api/negotiations/[id]/generate-link/route.ts` | `src/app/api/negotiation/[id]/generate-link/route.ts` |
| MOP `app/api/negotiations/[id]/download/route.ts` | `src/app/api/negotiation/[id]/download/route.ts` |
| MOP `app/api/negotiations/[id]/cancel/route.ts` | `src/app/api/negotiation/[id]/cancel/route.ts` |

#### Data Rewiring
- MOP `contract_negotiations` → shared Supabase `negotiation_records` via @milo/contract-negotiation client
- MOP `negotiation_rounds` → shared `negotiation_rounds`
- MOP `negotiation_links` → shared `negotiation_links`
- Linked analysis: `analysis_id` FK resolves in shared Supabase (same DB)
- Review link URLs: Milo-for-PPC domain (not MOP)

#### Display Config
- Linked analysis results: severity badges shown, no aggregate score
- Negotiation detail shows per-issue decisions (accept/reject/counter)

#### Test Plan
- Create negotiation from analysis results → negotiation_record created
- Submit round → round saved, counterparty notified
- Generate review link → token-based URL using Milo-for-PPC domain
- Finalize → status updates, audit trail recorded
- Cancel → status updates
- Download → HTML with full round history
- Verify with 2 migrated negotiation records (D56: 2 records, 7 rounds, 5 links)

#### Risk: MEDIUM
- Depends on Track A completing (creates negotiation from analysis)
- Only 2 existing negotiation records for testing (small dataset)

---

### B4: Negotiation Review (Public) — P3 (~400 LOC)

Public page where counterparties review and respond to negotiations.

#### Commits

| # | Commit | LOC | Description |
|---|--------|-----|-------------|
| B4-1 | `feat: negotiation review page` | ~250 | `src/app/negotiation/review/[token]/page.tsx` — token resolution via @milo/contract-negotiation. Displays current round with proposed changes. Accept/reject/counter per issue. Previous rounds read-only. No auth (public, token-validated). |
| B4-2 | `feat: negotiation review API routes` | ~150 | `src/app/api/negotiation/review/[token]/route.ts` (GET: load review data) + `submit/route.ts` (POST: submit counterparty decisions). View tracking (optional). |

#### File Mapping

| Source | Target |
|--------|--------|
| MOP `app/review/[token]/page.tsx` (455 LOC) | `src/app/negotiation/review/[token]/page.tsx` (~250 LOC) |
| MOP `app/api/review/[token]/route.ts` (133 LOC) | `src/app/api/negotiation/review/[token]/route.ts` (~80 LOC) |
| MOP `app/api/review/[token]/submit/route.ts` (132 LOC) | `src/app/api/negotiation/review/[token]/submit/route.ts` (~70 LOC) |

#### Data Rewiring
- Token resolution: @milo/contract-negotiation `resolveReviewLink(token)`
- All negotiation data from shared Supabase
- Linked analysis issues rendered with IssueCard (severity only, no score)

#### Display Config
- Per-issue severity on flagged clauses. No aggregate score.

#### Test Plan
- Valid token → review page loads with current round
- Accept all → decisions saved, notification sent to operator
- Counter with comments → new round created
- Expired link → 410 Gone (7-day default per D44 D9)
- Verify with 5 migrated negotiation_links

#### Risk: LOW

---

### B5: Rider Templates — P5 (~400 LOC)

CRUD for rider template text blocks referenced in analysis recommendations.

#### Commits

| # | Commit | LOC | Description |
|---|--------|-----|-------------|
| B5-1 | `feat: rider templates page` | ~250 | `src/app/contract-analysis/riders/page.tsx` — list all rider templates, create/edit/delete. Template name, category, body text. |
| B5-2 | `feat: rider templates API route` | ~150 | `src/app/api/contract-analysis/riders/route.ts` — CRUD for `rider_templates` table. Scoped to `tenant_id = 'tlp'`. |

#### File Mapping

| Source | Target |
|--------|--------|
| MOP `app/contract-analyzer/rider-templates/page.tsx` (508 LOC) | `src/app/contract-analysis/riders/page.tsx` (~250 LOC) |

#### Data Rewiring
- `rider_templates` table: stored in Milo-for-PPC's Supabase (not shared — vertical-owned data)
- Rider text referenced by `riderRef` field in analysis issue results

#### Test Plan
- Create/edit/delete rider templates
- Analysis results reference rider template text correctly
- Templates scoped to TLP tenant

#### Risk: LOW

---

## Dependency Graph

```
Track A ──────────────────────────────────────
│
A1: Shared components (SeverityBadge, IssueCard, etc.)
│
├── A2: Contract analysis upload (P4)
│   │
│   ├── A3: Export toolbar (R4)
│   │
│   └── A4: Analysis history (R1) [uses display name fallback chain]
│       │
│       └── A6: Settings/playbook (R2)
│
└── A5: Help/KB panel (R3) [independent, wire into A2/A4]

Track B ──────────────────────────────────────
│
B1: Signing dashboard (P6) [uses metadata.entity_name + counterparty_company]
│
├── B2: Signing page (P1) [public]
│
B3: Negotiation admin (P2) [depends on A2 for "create from analysis"]
│
├── B4: Negotiation review (P3) [public]
│
└── B5: Rider templates (P5)

POST: MOP URL redirects + dashboard wiring

[PRE-1 entity_id_mapping CANCELLED — Decision 67]
```

**Track A and Track B can run in parallel** except:
- B3 (negotiation admin) depends on A2 (analysis upload) for "create negotiation from analysis" flow
- B5 (rider templates) depends on A2 results display for rider text references

**Recommendation:** Start both tracks simultaneously. B1 + B2 (signing) have zero Track A dependency. B3 starts after A2 lands. B5 can start anytime.

---

## Post-Phase-2: MOP URL Redirect Setup

After all 10 UIs are live on Milo-for-PPC, set up MOP redirects for active counterparty links.

### Assets Requiring Redirect

| Asset | Count (at migration) | Current URL | Target URL | Natural Expiry |
|-------|---------------------|-------------|------------|---------------|
| Pending signing docs | 68 | `tlpmop.netlify.app/sign/[token]` | `tlp.justmilo.app/signing/[token]` | 30 days from creation |
| Active negotiation | 1 | `tlpmop.netlify.app/review/[token]` | `tlp.justmilo.app/negotiation/review/[token]` | Active until resolved |
| Historical signed docs | 109 | `tlpmop.netlify.app/sign/[token]` | `tlp.justmilo.app/signing/[token]` | Already completed |

### Redirect Implementation

| # | Commit | LOC | Where | Description |
|---|--------|-----|-------|-------------|
| POST-1 | `feat: signing redirect route` | ~25 | MOP repo | `app/sign/[token]/page.tsx` → 301 redirect to `tlp.justmilo.app/signing/[token]` |
| POST-2 | `feat: review redirect route` | ~25 | MOP repo | `app/review/[token]/page.tsx` → 301 redirect to `tlp.justmilo.app/negotiation/review/[token]` |

### Timeline
- Most pending signing docs will resolve naturally within 30 days
- The 1 active negotiation should be resolved before cutover
- Redirects stay in MOP indefinitely (MOP remains as TrackDrive microservice per D-OPEN-10)

---

## Phase 2 Exit Criteria

All must be TRUE before declaring Phase 2 complete:

| # | Criterion | Verification |
|---|-----------|-------------|
| 1 | All 10 UIs live on `tlp.justmilo.app` | Manual verification of each page |
| 2 | Contract analysis upload produces results | Upload test PDF, verify issue list |
| 3 | Signing flow end-to-end | Generate signing link, complete signature |
| 4 | Negotiation flow end-to-end | Create from analysis, send round, counterparty review |
| 5 | No aggregate scores visible anywhere | Audit all contract-displaying pages |
| 6 | MOP redirect routes deployed | Click old MOP URLs → arrive at Milo-for-PPC |
| 7 | Operators confirmed using Milo-for-PPC | Mark confirms operators have switched |
| 8 | `npm run build` passes clean | CI check |
| 9 | All 33 migrated analyses display correctly | Spot-check 5 in history page |
| 10 | All 109 migrated signing docs display correctly | Verify signing dashboard counts |

---

## Full Commit Sequence (Recommended Order)

| Order | ID | UI | Track | LOC | Risk |
|-------|-----|-----|-------|-----|------|
| ~~0~~ | ~~PRE-1~~ | ~~entity_id_mapping~~ | ~~Pre~~ | ~~~100~~ | ~~LOW~~ CANCELLED (Decision 67) |
| 1 | A1-1 | SeverityBadge + displayConfig | A | ~100 | LOW |
| 2 | A1-2 | IssueCard | A | ~250 | LOW |
| 3 | A1-3 | IssueSidebar + selectors | A | ~150 | LOW |
| 4 | A2-1 | Analysis API route | A | ~200 | MED |
| 5 | A2-2 | Analysis upload page | A | ~250 | MED |
| 6 | A2-3 | Analysis polling + results | A | ~150 | MED |
| 7 | B1-1 | Signing dashboard page | B | ~150 | LOW |
| 8 | B1-2 | Signing dashboard actions | B | ~50 | LOW |
| 9 | A3-1 | Export toolbar + DOCX | A | ~200 | LOW |
| 10 | A3-2 | Wire export into results | A | ~100 | LOW |
| 11 | B2-1 | Signing page shell + token | B | ~200 | HIGH |
| 12 | B2-2 | SigningForm port | B | ~350 | HIGH |
| 13 | B2-3 | SigningSidebar + Preview | B | ~150 | MED |
| 14 | B2-4 | Signing API routes | B | ~300 | HIGH |
| 15 | B2-5 | SigningConfirmation + signed doc | B | ~150 | MED |
| 16 | B2-6 | Signing branding + CompanySettings | B | ~50 | LOW |
| 17 | A4-1 | Analysis history list | A | ~200 | MED |
| 18 | A4-2 | History inline actions | A | ~100 | LOW |
| 19 | A4-3 | Revision comparison modal | A | ~200 | MED |
| 20 | A5-1 | Help KB content | A | ~200 | LOW |
| 21 | A5-2 | HelpPanel + HelpTooltip | A | ~150 | LOW |
| 22 | A5-3 | Wire help into pages | A | ~50 | LOW |
| 23 | B3-1 | Negotiation list page | B | ~200 | MED |
| 24 | B3-2 | Negotiation detail page | B | ~250 | MED |
| 25 | B3-3 | Negotiation API routes | B | ~200 | MED |
| 26 | B3-4 | Negotiation download | B | ~150 | LOW |
| 27 | B4-1 | Negotiation review page | B | ~250 | LOW |
| 28 | B4-2 | Negotiation review API | B | ~150 | LOW |
| 29 | B5-1 | Rider templates page | B | ~250 | LOW |
| 30 | B5-2 | Rider templates API | B | ~150 | LOW |
| 31 | A6-1 | Settings/profiles page | A | ~200 | LOW |
| 32 | A6-2 | Rule toggles + localStorage | A | ~150 | LOW |
| 33 | A6-3 | Wire settings into upload | A | ~50 | LOW |
| 34 | — | Dashboard display config wiring | — | ~100 | LOW |
| 35 | POST-1 | MOP signing redirect | Post | ~25 | LOW |
| 36 | POST-2 | MOP review redirect | Post | ~25 | LOW |
| | | **TOTAL** | | **~5,500** | |

---

## Coder Assignment Recommendation

**Parallel execution (Track A + Track B):**
- **Coder-1:** Track A (analysis UIs) — already familiar with @milo/contract-analysis internals from v0.2.0 implementation
- **Coder-2:** Track B (signing + negotiation UIs) — already familiar with @milo/contract-signing from consumer migration work

**PRE-1** (entity_id_mapping): either Coder, or Coder-4 as a quick ops task.

**POST-1/POST-2** (MOP redirects): Coder who has MOP deploy access.

**Estimated timeline:**
- Sequential (one Coder): ~3.5 weeks
- Parallel (two Coders): ~2 weeks
- B3 (negotiation admin) waits for A2 (analysis upload) — ~2 day lag if tracks start simultaneously

---

## Open Questions

**Zero new open questions.** All architectural decisions locked in D64 batch resolution. The 6 questions from the UI port plan (§12) are resolved:

| UI Port Plan Q | Resolution |
|----------------|-----------|
| Q1: Rider templates table location | Vertical-layer table in Milo-for-PPC (not shared Supabase) |
| Q2: Settings persistence | localStorage Phase 1 (D-OPEN-7) |
| Q3: Signing token format | Reuse @milo/contract-signing token scheme (data already migrated with same tokens per D54) |
| Q4: ContractAnalysisDisplay rewrite | Wire existing block to @milo/contract-analysis + display config (commit 34 above). Replace with IssueCard pattern if wiring proves too entangled. |
| Q5: MOP reference files | Don't copy — developers read MOP repo directly |
| Q6: Coder assignment | Parallel recommended (Coder-1 Track A, Coder-2 Track B) |
