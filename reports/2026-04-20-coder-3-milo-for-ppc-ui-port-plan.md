---
directive: "Milo-for-PPC UI port plan (Phase 2)"
lane: research
coder: Coder-3
started: 2026-04-20 ~15:30 MDT
completed: 2026-04-20 ~16:30 MDT
supplements:
  - reports/2026-04-20-coder-3-milo-for-ppc-scaffolding-plan.md (b5e376f)
  - reports/2026-04-20-coder-3-ppc-rules-dedup-mapping.md (254794c)
  - reports/2026-04-20-coder-3-contract-analysis-v02-spec.md (a338922)
  - reports/2026-04-20-coder-3-consumer-wave-plan.md (a69fb83)
---

# Milo-for-PPC UI Port Plan — Phase 2

## 1. Executive Summary

| Metric | Value |
|--------|-------|
| UIs to port (adapt from MOP) | 6 pages |
| UIs to rebuild (from contract-checker patterns) | 4 pages |
| UIs that stay as-is (milo-ops base) | 14 pages |
| UIs to reference only (stay in MOP forever) | 11 pages |
| Total new/modified LOC (Phase 2) | ~4,800 |
| Estimated effort | ~3-4 weeks |

**Strategy:** milo-ops is the base — its 14 operator pages carry over unchanged. MOP contributes 6 public-facing and contract-management UIs that get ported (rewired from MOP Supabase to shared Supabase via @milo/* clients). contract-checker contributes 4 UIs with better operator UX patterns that get rebuilt fresh using its design as the reference.

**Display config (locked, Directive #35):** All contract-related UIs render per-issue severity only. No aggregate scores, letter grades, or risk bucket labels anywhere in Milo-for-PPC.

---

## 2. UIs That Stay As-Is (milo-ops Base)

These pages carry over from the milo-ops fork with zero UI changes. Data layer already points at milo-ops Supabase (tmzsmdkfqvjjjwkukysg) or shared Supabase via @milo/* clients.

| Page | Path | LOC | Data Source | Notes |
|------|------|-----|-------------|-------|
| Dashboard v2 | `app/dashboard-v2/` | ~2,237 | Multiple block APIs | 34 role-based blocks, core operator view |
| Operator landing | `app/operator/` | ~521 | Briefing + pills API | Morning briefing, action pills, search |
| Pipeline | `app/pipeline/` | ~1,485 | `prospect_pipeline`, contacts | Kanban board + entity detail |
| Pipeline board | `app/pipeline-board/` | ~1,251 | `prospect_pipeline`, contacts | Alternative kanban layout |
| Partners | `app/partners/` | ~599 | PPCRM API → @milo/onboarding | Partner onboarding status |
| Campaigns | `app/campaigns/` | ~424 | `campaigns`, `campaign_routes` | Campaign management |
| Disputes | `app/disputes/` | ~1,526 | `disputes`, `call_records` | Dispute resolution workflow |
| Invoices | `app/invoices/` | ~701 | `invoices`, TrackDrive reconciliation | AR/AP billing |
| EOD reports | `app/eod-reports/` | ~580 | `eod_reports` | End-of-day summaries |
| Reports | `app/reports/` | ~992 | `call_records`, `campaigns` | Multi-tab reporting |
| Ping/post | `app/ping-post/` | ~685 | `ping_posts`, TrackDrive | Lead routing analytics |
| Pings detail | `app/pings/` | ~431 | `ping_posts`, `call_records` | Individual ping trace |
| Support | `app/support/` | ~491 | `support_tickets` | Support queue |
| Observatory | `app/observatory/` | ~305 | System health API | Uptime, sync status |

**Phase 1 rewiring needed (already in scaffolding plan):**
- `app/partners/` — PPCRM proxy calls → @milo/onboarding direct
- Dashboard ContractAnalysisDisplay block — already displays contract results; will consume @milo/contract-analysis after Phase 2 wiring

**Total carried LOC:** ~12,228 (unchanged)

---

## 3. UIs to Port from MOP

These UIs exist in MOP today, serving public-facing and contract-management workflows. They get ported to Milo-for-PPC with data layer rewired from MOP's local Supabase (wjxtfjaixkoifdqtfmqd) to shared Supabase (tappyckcteqgryjniwjg) via @milo/* primitives.

### 3a. Signing Page (Public)

| Field | Value |
|-------|-------|
| **Source** | MOP `app/sign/[token]/page.tsx` + `hooks/` + `components/` + `app/api/sign/` |
| **Source LOC** | ~4,086 (page 452, hooks 989, components 1,264, API routes 831, utilities 550) |
| **Target** | `app/signing/[token]/page.tsx` |
| **Target LOC estimate** | ~1,200 (simplified — strip MOP cruft, use @milo/contract-signing client) |
| **Primitives consumed** | @milo/contract-signing |
| **Data source change** | `signing_documents` on MOP Supabase → `signing_documents` on shared Supabase via `@milo/contract-signing` client |
| **Auth model** | **None** — public route, token-authenticated (counterparty clicks email link). Token resolution moves from MOP HMAC to @milo/contract-signing `resolveToken()`. |
| **Visual change** | Minimal — same document viewer + signature capture. Strip MOP navigation chrome, apply Milo-for-PPC branding (VERTICAL_CONFIG colors, logo). |
| **Display config** | No score/grade shown on signing pages. Document metadata + signature fields only. |

**Key components to port:**
- `SigningForm.tsx` (638 LOC) — signature capture, field validation
- `SigningSidebar.tsx` (285 LOC) — document metadata display
- `SigningConfirmation.tsx` (226 LOC) — post-signature confirmation
- `useSigningForm.ts` (701 LOC) — form state management, validation, submission
- `useDocumentLoader.ts` (96 LOC) — token → document resolution
- `generateSignedDocument.ts` (350 LOC) — PDF assembly post-signature

**API routes to port:**
- `POST /api/signing/[token]/submit` — submit signed document → @milo/contract-signing `submitSignature()`
- `POST /api/signing/[token]/decline` — decline to sign → @milo/contract-signing `declineDocument()`
- `POST /api/signing/[token]/capture-view` — view analytics (optional, low priority)

**What changes:**
1. Supabase client: MOP local → shared via `@milo/contract-signing` client SDK
2. Token resolution: MOP's custom HMAC → `@milo/contract-signing.resolveToken(token)`
3. Entity lookup: `publishers`/`buyers` → `crm_counterparties` via `@milo/crm`
4. Tenant filter: add `tenant_id = 'tlp'` to all queries
5. Branding: MOP logo/colors → VERTICAL_CONFIG

---

### 3b. Negotiation Admin (Operator)

| Field | Value |
|-------|-------|
| **Source** | MOP `app/negotiations/page.tsx` + `app/negotiations/[id]/page.tsx` + `app/api/negotiations/` |
| **Source LOC** | ~2,319 (pages 1,115, API routes 1,004, download 502) |
| **Target** | `app/negotiation/page.tsx` + `app/negotiation/[id]/page.tsx` |
| **Target LOC estimate** | ~800 |
| **Primitives consumed** | @milo/contract-negotiation, @milo/contract-analysis |
| **Data source change** | `contract_negotiations` + `negotiation_rounds` on MOP → shared Supabase via @milo/contract-negotiation client |
| **Auth model** | Shared Supabase auth (operator session). RLS + tenant_id. |
| **Visual change** | Apply milo-ops design system (dark theme, inline styles + Tailwind). Strip MOP's navigation, use Milo-for-PPC layout. |
| **Display config** | Per-issue severity on linked analysis results. No aggregate score. |

**Key features to port:**
- Negotiation list with status filters (active/completed/cancelled)
- Negotiation detail: round history, linked analysis, round submission
- Create negotiation from analysis results
- Generate shareable review link (token-based)
- Download negotiation as DOCX/HTML
- Cancel/finalize negotiation

**API routes to port:**
- `POST /api/negotiation/create` → @milo/contract-negotiation `createNegotiation()`
- `POST /api/negotiation/[id]/send-round` → `submitRound()`
- `POST /api/negotiation/[id]/finalize` → `finalizeNegotiation()`
- `POST /api/negotiation/[id]/generate-link` → `createReviewLink()`
- `GET /api/negotiation/[id]/download` → `exportNegotiation()`
- `POST /api/negotiation/[id]/cancel` → `cancelNegotiation()`

---

### 3c. Negotiation Review (Public)

| Field | Value |
|-------|-------|
| **Source** | MOP `app/review/[token]/page.tsx` + `app/api/review/` |
| **Source LOC** | ~720 (page 455, API routes 265) |
| **Target** | `app/negotiation/review/[token]/page.tsx` |
| **Target LOC estimate** | ~400 |
| **Primitives consumed** | @milo/contract-negotiation |
| **Data source change** | `negotiation_links` on MOP → shared Supabase via @milo/contract-negotiation |
| **Auth model** | **None** — public route, token-authenticated. Counterparty reviews + approves/rejects. |
| **Visual change** | Minimal — apply Milo-for-PPC branding. Same approve/reject/counter UI. |
| **Display config** | Per-issue severity on flagged clauses. No aggregate score. |

**Key features:**
- Token → negotiation resolution
- Display current round with proposed changes
- Accept / reject / counter with comments
- Previous rounds history (read-only)

---

### 3d. Contract Analyzer Upload

| Field | Value |
|-------|-------|
| **Source** | MOP `app/contract-analyzer/page.tsx` (1,465 LOC) — upload + analyze workflow |
| **Target** | `app/contract-analysis/page.tsx` |
| **Target LOC estimate** | ~600 |
| **Primitives consumed** | @milo/contract-analysis (with PPC rules plugin) |
| **Data source change** | `contract_analyses` on MOP → `analysis_records` on shared Supabase |
| **Auth model** | Shared Supabase auth (operator session) |
| **Visual change** | Use contract-checker's issue card design (expandable, original/suggested/legal/negotiation sections). No MOP score wheel. Apply no-score display config. |
| **Display config** | `displayAggregateScore: false`, `displayLetterGrade: false`, `displayRiskBucket: false`. Issue list ordered by severity (CRITICAL first). Per-issue fields: severity badge, plain English explanation, recommendation, rider text reference. |

**Upload flow (adapted from MOP):**
1. Drag-drop or file picker (PDF, DOCX, TXT) — keep MOP's file handling
2. Role selector: Publisher / Buyer — keep from contract-checker
3. Playbook preset: Full Audit / Publisher Defensive / Buyer Aggressive / Quick Review — from contract-checker
4. Jurisdiction selector — from contract-checker (two-party consent escalation)
5. Submit → @milo/contract-analysis `createAnalysis()` with PPC rules + vertical config
6. Display results as severity-ordered issue list (contract-checker card design)

**Key difference from MOP:** MOP shows a score wheel + letter grade + risk bucket at the top of results. Milo-for-PPC shows ONLY the issue list with per-issue severity badges. Engine still computes scores internally for sorting.

---

### 3e. Rider Templates

| Field | Value |
|-------|-------|
| **Source** | MOP `app/contract-analyzer/rider-templates/page.tsx` (508 LOC) |
| **Target** | `app/contract-analysis/riders/page.tsx` |
| **Target LOC estimate** | ~400 |
| **Primitives consumed** | @milo/contract-analysis (or vertical-layer `rider_templates` table) |
| **Data source change** | `rider_templates` on MOP → shared or vertical Supabase (decision pending — see scaffolding plan Section 7c) |
| **Auth model** | Shared Supabase auth (operator session) |
| **Visual change** | Apply milo-ops design system |

**Features:** CRUD for rider template text blocks that get referenced in analysis recommendations.

---

### 3f. Signing Documents Dashboard (Operator)

| Field | Value |
|-------|-------|
| **Source** | MOP `app/signing/page.tsx` (200 LOC) |
| **Target** | `app/signing/page.tsx` |
| **Target LOC estimate** | ~200 |
| **Primitives consumed** | @milo/contract-signing |
| **Data source change** | `signing_documents` on MOP → shared Supabase |
| **Auth model** | Shared Supabase auth (operator session) |
| **Visual change** | Apply milo-ops design system |

**Features:** List all signing documents with status filters (pending/signed/declined/expired). Operator overview of document pipeline.

---

### Port Summary Table

| # | UI | Source | Target Path | LOC (new) | Primitives | Public? |
|---|-----|--------|-------------|-----------|-----------|---------|
| P1 | Signing page | MOP `app/sign/[token]/` | `app/signing/[token]/` | ~1,200 | @milo/contract-signing | Yes (token) |
| P2 | Negotiation admin | MOP `app/negotiations/` | `app/negotiation/` | ~800 | @milo/contract-negotiation | No |
| P3 | Negotiation review | MOP `app/review/[token]/` | `app/negotiation/review/[token]/` | ~400 | @milo/contract-negotiation | Yes (token) |
| P4 | Contract analyzer | MOP `app/contract-analyzer/` | `app/contract-analysis/` | ~600 | @milo/contract-analysis | No |
| P5 | Rider templates | MOP `app/contract-analyzer/rider-templates/` | `app/contract-analysis/riders/` | ~400 | @milo/contract-analysis | No |
| P6 | Signing dashboard | MOP `app/signing/` | `app/signing/` | ~200 | @milo/contract-signing | No |
| | **Total port LOC** | | | **~3,600** | | |

---

## 4. UIs to Rebuild (contract-checker Patterns)

These UIs don't exist in MOP or milo-ops in a suitable form. contract-checker has proven UX patterns worth adopting. Built fresh in Milo-for-PPC using contract-checker as design reference.

### 4a. Analysis History + Revision Comparison

| Field | Value |
|-------|-------|
| **Reference** | contract-checker `app/history/page.tsx` (1,630 LOC) |
| **Target** | `app/contract-analysis/history/page.tsx` |
| **Target LOC estimate** | ~500 |
| **Primitives consumed** | @milo/contract-analysis |
| **Data source** | `analysis_records` + `analysis_jobs` on shared Supabase |

**Features to inherit from contract-checker:**
- Search + filter by document name
- Active / archived tabs with counts
- Inline rename
- Archive / restore / permanent delete
- Revision comparison modal (upload revised contract, diff visualization)
- 4-column comparison summary: Implemented / Partially / Not Done / Unexpected
- Teams message generator for follow-up (contract-checker already has this)
- Auto-cleanup stats (90d → archive, 365d → delete)

**What MOP has instead:** `app/contract-analyzer/history/page.tsx` (507 LOC) — simpler list view without revision comparison, archiving, or Teams integration. contract-checker's version is significantly better for operator workflow.

---

### 4b. Settings / Playbook Configuration

| Field | Value |
|-------|-------|
| **Reference** | contract-checker `app/settings/page.tsx` (515 LOC) + `lib/playbook-rules.ts` (746 LOC) |
| **Target** | `app/settings/page.tsx` (or `app/contract-analysis/settings/`) |
| **Target LOC estimate** | ~400 |
| **Primitives consumed** | PPC rules plugin (77 rules from `src/lib/ppc/rules.ts`) |
| **Data source** | localStorage (per-operator) or `company_settings` table |

**Features to inherit from contract-checker:**
- Rule list grouped by category (FINANCIAL, TERMINATION, LIABILITY, OPERATIONAL, COMPLIANCE, DISPUTE)
- Per-rule: enable/disable toggle, severity badge, applies-to badge (publisher/buyer/both)
- Threshold editor for rules with configurable triggers (payment days, holdback %, etc.)
- Preset selector: Full Audit / Publisher Defensive / Buyer Aggressive / Quick Review
- Threshold profile selector: Strict / Standard / Relaxed
- Stats display: "X of Y active", "Z critical"
- Save to localStorage (fast) with optional Supabase persistence

**Adaptation for PPC:**
- 77 rules instead of contract-checker's ~30
- PPC_SEVERITY_OVERRIDES and PPC_THRESHOLD_OVERRIDES as defaults
- PPC_MANDATORY_CHECKS shown as locked-on (cannot disable)

---

### 4c. Help / Knowledge Base Panel

| Field | Value |
|-------|-------|
| **Reference** | contract-checker `components/help/help-panel.tsx` (269 LOC) + `help-tooltip.tsx` (72 LOC) + `lib/help-knowledge.ts` (316 LOC) |
| **Target** | `src/components/help/help-panel.tsx` + `help-tooltip.tsx` + `src/lib/help-knowledge.ts` |
| **Target LOC estimate** | ~400 |
| **Primitives consumed** | @milo/ai-client |

**Features to inherit:**
- AI chatbot sidebar (slide-in sheet)
- Suggested questions (6 defaults, PPC-specific)
- Context injection: full KB as system message
- Per-issue "AI Assist" button with quick actions:
  - "Make less aggressive"
  - "Make more specific"
  - "Explain simpler"
  - "Show alternatives"
- Glossary tooltips on key terms (30+ terms: net_30, clawback, holdback, TCPA, etc.)

**PPC adaptations:**
- KB content rewritten for PPC domain (pay-per-call terms, TCPA specifics, lead quality)
- Suggested questions: PPC-focused ("What does a clawback clause mean for my call revenue?")
- AI backend: @milo/ai-client instead of contract-checker's direct Anthropic calls
- Tagalog support deferred to Phase 2 per Q8 answer

---

### 4d. Export Toolbar

| Field | Value |
|-------|-------|
| **Reference** | contract-checker `components/results/export-toolbar.tsx` (199 LOC) + `lib/export.ts` (224 LOC) + MOP `lib/contract-checker/docx-*.ts` |
| **Target** | `src/components/contract/export-toolbar.tsx` + `src/lib/export.ts` |
| **Target LOC estimate** | ~300 |

**Features:**
- Export to DOCX (using `docx` package — contract-checker pattern)
- Export to PDF
- Export to JSON (API consumption)
- Per-analysis export with issue list, original text, suggested revisions

**PPC adaptation:** Export format excludes aggregate score (consistent with display config). Issue list ordered by severity.

---

### Rebuild Summary Table

| # | UI | Reference | Target Path | LOC (new) |
|---|-----|-----------|-------------|-----------|
| R1 | Analysis history | CC `app/history/` | `app/contract-analysis/history/` | ~500 |
| R2 | Settings/playbook | CC `app/settings/` | `app/settings/` | ~400 |
| R3 | Help/KB panel | CC `components/help/` | `src/components/help/` | ~400 |
| R4 | Export toolbar | CC `components/results/export-toolbar` + MOP docx | `src/components/contract/` | ~300 |
| | **Total rebuild LOC** | | | **~1,600** |

---

## 5. UIs to Reference Only (Stay in MOP Forever)

These MOP pages are NOT ported. They either serve TrackDrive-specific workflows, are superseded by milo-ops equivalents, or contain legacy patterns.

### 5a. TrackDrive-Specific (Permanent MOP Residents)

| Page | MOP Path | LOC | Why Not Port |
|------|----------|-----|-------------|
| Call log browser | `app/calls/page.tsx` | 1,861 | TrackDrive call data display. milo-ops already has its own call reporting via `app/reports/` and `CallDetailModal` (19K LOC). |
| Campaign list | `app/campaigns/page.tsx` | 193 | MOP campaign ≠ milo-ops campaign. Different data model. milo-ops has its own. |
| Campaign detail | `app/campaigns/[id]/page.tsx` | 404 | Same — MOP's campaign model is TrackDrive-native. |
| Ping-post stats | `app/ping-post/page.tsx` | 1,886 | milo-ops already has `app/ping-post/` (685 LOC) with equivalent functionality. |
| Outreach queue | `app/outreach/page.tsx` | 748 | milo-ops handles outreach differently (drip campaigns, conversation captures). |
| ConvoQC coverage | `app/convoqc-coverage/page.tsx` | 127 | milo-ops integrates ConvoQC via dashboard blocks (QualityBlock, FlaggedCallsBlock). |

### 5b. Entity Management (Superseded by milo-ops)

| Page | MOP Path | LOC | Why Not Port |
|------|----------|-----|-------------|
| Publisher list | `app/publishers/page.tsx` | 1,545 | milo-ops has `app/pipeline/` with EntityDetailPanel (196K LOC). Far richer. |
| Publisher detail | `app/publishers/[id]/page.tsx` | 1,357 | Same — milo-ops entity detail subsumes MOP's per-entity pages. |
| Buyer list | `app/buyers/page.tsx` | 1,519 | Same pattern. |
| Buyer detail | `app/buyers/[id]/page.tsx` | 1,234 | Same pattern. |
| End-buyers | `app/end-buyers/page.tsx` | 634 | milo-ops dashboard has EndBuyersWidget. |

### 5c. Other MOP Pages Not Porting

| Page | MOP Path | LOC | Why Not Port |
|------|----------|-----|-------------|
| Main dashboard | `app/page.tsx` | 1,683 | milo-ops Dashboard v2 is superior (34 blocks, role-based). |
| Routes manager | `app/routes/page.tsx` | 1,078 | TrackDrive route management — no milo-for-PPC equivalent needed. |
| DID management | `app/dids/page.tsx` | 616 | milo-ops has DIDTrackerBlock (8,222 LOC). |
| Settings | `app/settings/page.tsx` | 1,289 | MOP settings are MOP-specific (verticals, lead types, synonyms). |
| Master blacklist | `app/master-blacklist/page.tsx` | 407 | @milo/blacklist consumed directly. |
| Invoices | `app/invoices/page.tsx` | 1,543 | milo-ops has its own invoices page (701 LOC). |
| Offers | `app/offers/page.tsx` | 699 | MOP-specific offer management. |
| Reports | `app/reports/page.tsx` | 1,333 | milo-ops has its own reports (992 LOC). |
| Public invoice | `app/invoice/[token]/page.tsx` | 179 | Keep in MOP — invoicing stays MOP-side for now. |
| Onboard link | `app/onboard/[token]/page.tsx` | 230 | milo-ops has onboard flow. |

### 5d. MOP Shared Components — Selectively Inherit

MOP has 65 components (~15,632 LOC). Most are NOT needed — milo-ops has its own component library. Selectively reference:

| MOP Component | LOC | Inherit? | Reason |
|---------------|-----|----------|--------|
| `SigningForm.tsx` | 638 | **YES** — port | Core signing UI |
| `SigningSidebar.tsx` | 285 | **YES** — port | Document metadata |
| `SigningConfirmation.tsx` | 226 | **YES** — port | Post-signature flow |
| `DocumentUploadModal.tsx` | 290 | **MAYBE** | milo-ops may have equivalent |
| `DocumentList.tsx` | 242 | **MAYBE** | For signing dashboard |
| `BlacklistModal.tsx` | 287 | **NO** | @milo/blacklist handles this |
| `MatchSuggestionsModal.tsx` | 196 | **NO** | milo-ops has AddProspectModal |
| All widget components | ~1,408 | **NO** | milo-ops Dashboard v2 blocks replace these |
| `FilterBar.tsx` | 358 | **NO** | milo-ops has its own filter patterns |
| `GlobalSearch.tsx` | 312 | **NO** | milo-ops has its own search |

---

## 6. contract-checker Features — Deferred vs. Inherited

| Feature | CC LOC | Inherit Now? | Where | Notes |
|---------|--------|-------------|-------|-------|
| Issue card (expandable) | 350 | **YES** (R4) | Analysis results display | Better than MOP's flat list — shows original/suggested/legal/negotiation |
| Summary card (risk) | 127 | **NO** | — | Shows aggregate score. Conflicts with no-score directive. |
| Issue sidebar (jump-to) | 176 | **YES** | Analysis results | Quick navigation for long issue lists |
| AI assistant panel | 437 | **YES** (R3) | Help panel | Per-issue "AI Assist" with quick actions |
| Revision comparison | (in history) | **YES** (R1) | Analysis history | Diff visualization + Teams follow-up |
| Tagalog translations | 218 + 36 | **DEFERRED** | Phase 2 per Q8 | Filipino VAs work in English today |
| Playbook rules config | 515 + 746 | **YES** (R2) | Settings page | Rule enable/disable, thresholds, presets |
| DOCX export | 224 | **YES** (R4) | Export toolbar | `docx` package, structured output |
| Role selector | (in upload) | **YES** | Analysis upload | Publisher / Buyer role selection |
| Jurisdiction selector | (in upload) | **YES** | Analysis upload | Two-party consent state trigger |
| Zustand store | 246 | **NO** | — | milo-ops uses different state patterns |
| shadcn/ui components | 12 | **NO** | — | milo-ops uses inline styles + Tailwind (no shadcn) |

---

## 7. Display Config Per Ported UI

Per Directive #35 (no-score architectural directive), locked in `VERTICAL_CONFIG.analysis`:

| UI | Score | Grade | Risk Bucket | Severity Badges | Issue List | Notes |
|----|-------|-------|-------------|-----------------|------------|-------|
| Contract analysis results (P4) | **Hidden** | **Hidden** | **Hidden** | **Shown** (CRITICAL/HIGH/MEDIUM/LOW) | **Shown** (ordered severity-desc) | Primary display — operators see per-issue triage |
| Signing page (P1) | N/A | N/A | N/A | N/A | N/A | No analysis results on signing pages |
| Negotiation admin (P2) | **Hidden** | **Hidden** | **Hidden** | **Shown** | **Shown** | Linked analysis shown on negotiation detail |
| Negotiation review (P3) | **Hidden** | **Hidden** | **Hidden** | **Shown** | **Shown** | Counterparty sees per-issue flagged clauses |
| Analysis history (R1) | **Hidden** | **Hidden** | **Hidden** | **Shown** (summary badge) | Per-item severity count | History list shows issue count + top severity |
| Settings/playbook (R2) | N/A | N/A | N/A | **Shown** (per-rule default) | N/A | Rule severity as reference, not analysis result |
| Dashboard ContractAnalysisDisplay | **Hidden** | **Hidden** | **Hidden** | **Shown** | **Shown** (truncated) | Existing milo-ops block — needs display config wiring |
| Export (R4) | **Hidden** | **Hidden** | **Hidden** | **Shown** | **Shown** | DOCX/PDF exports exclude score section |

**Implementation:** All contract-displaying components read `VERTICAL_CONFIG.analysis` to determine which fields to render. The engine's response includes scores (for internal sorting), but the UI layer strips them before rendering when `displayAggregateScore: false`.

```typescript
// src/lib/display-config.ts
import { VERTICAL_CONFIG } from './vertical.config';

export function filterAnalysisForDisplay(analysis: AnalysisResult) {
  const config = VERTICAL_CONFIG.analysis;
  return {
    ...analysis,
    score: config.displayAggregateScore ? analysis.score : undefined,
    grade: config.displayLetterGrade ? analysis.grade : undefined,
    riskBucket: config.displayRiskBucket ? analysis.riskBucket : undefined,
    issues: analysis.issues
      .sort((a, b) => severityOrder(b.severity) - severityOrder(a.severity))
      .map(issue => ({
        ...issue,
        displayFields: config.perIssueFields,
      })),
  };
}
```

---

## 8. Active Counterparty Link Migration Strategy

### Current State

| Asset | Count | Current URL Pattern | Expiry |
|-------|-------|-------------------|--------|
| Pending signing documents | 68 | `mop-domain/sign/[token]` | 30 days from creation |
| Active negotiation reviews | 1 | `mop-domain/review/[token]` | No expiry (active until resolved) |
| Historical signed documents | 109 | `mop-domain/sign/[token]` | Already completed — links show "already signed" |

### The Problem

Counterparties (publishers/buyers) have received email links pointing to MOP's domain. When Milo-for-PPC goes live and MOP's contract features are decommissioned, those links break.

### Recommended Strategy: Dual-Host Then Redirect

**Phase 2 (Port):** Build signing/review pages in Milo-for-PPC. MOP keeps serving existing tokens. New documents generated from Milo-for-PPC use Milo-for-PPC URLs.

**Phase 3 (Operator Switch):** Both systems live. Existing MOP tokens still work. New tokens come from Milo-for-PPC.

**Phase 4 (MOP Retirement):**
1. Add redirect routes to MOP: `app/sign/[token]` → `milo-for-ppc-domain/signing/[token]`
2. Add redirect routes to MOP: `app/review/[token]` → `milo-for-ppc-domain/negotiation/review/[token]`
3. Since data has been migrated to shared Supabase, Milo-for-PPC can resolve the same tokens.
4. MOP's redirect routes stay alive as long as MOP runs (TrackDrive microservice).

**Per-counterparty UX impact:**

| Counterparty Scenario | Action | UX Impact |
|----------------------|--------|-----------|
| 68 pending signing docs — counterparty hasn't clicked yet | Counterparty clicks MOP link → redirect to Milo-for-PPC → signs there | Seamless (redirect is transparent) |
| 68 pending signing docs — counterparty clicked, started filling | If form was not submitted, redirected link starts fresh | Minor — counterparty re-enters signature (no saved state in MOP's signing flow) |
| 1 active negotiation — counterparty has review link | Redirect to Milo-for-PPC review page | Seamless |
| Historical signed docs — counterparty clicks old confirmation link | Redirect → "Document already signed" confirmation page | Seamless |

**Timeline for link migration:**
- Most pending signing docs will resolve naturally (signed or expired) within 30 days
- The 1 active negotiation should be resolved before Phase 4 cutover
- If any pending docs remain at cutover, redirect handles them

**Alternative considered and rejected:** Send fresh links from Milo-for-PPC to all 68 pending counterparties. Rejected because: (a) confusing to receive a second signing invitation, (b) unnecessary if redirect handles it, (c) risk of double-signing if counterparty uses both links.

---

## 9. TrackDrive Data Display in Milo-for-PPC

### Data Access (Per Q3 Decision)

Read-only cross-connection from MOP Supabase (wjxtfjaixkoifdqtfmqd). Milo-for-PPC keeps `MOP_SUPABASE_URL` + `MOP_SUPABASE_ANON_KEY` env vars for reading `call_logs` and `ping_post` tables.

### Which UIs Show TrackDrive Data

These milo-ops pages already display TrackDrive data and carry over unchanged:

| Page | Data Shown | Source Table (MOP Supabase) | How milo-ops Reads It |
|------|-----------|---------------------------|----------------------|
| Reports | Call volume, revenue, quality, publisher/buyer health | `call_records` (synced from TD) | milo-ops local `call_records` table (synced via cron) |
| Ping/post | Accepted/rejected breakdown by buyer | `ping_posts` (synced from TD) | milo-ops local `ping_posts` table |
| Pings detail | Individual ping trace | `ping_posts` + `call_records` | Same |
| Dashboard v2 — multiple blocks | Call volume, revenue charts, flagged calls, campaign health, publisher/buyer monitoring | `call_records`, `campaigns` | Same |
| Disputes | Call-level evidence for dispute resolution | `call_records` | Same |

**Key insight:** milo-ops does NOT read from MOP Supabase directly. It has its OWN local tables (`call_records`, `ping_posts`, `campaigns`) synced via cron jobs from TrackDrive API. The TrackDrive sync code (`src/lib/trackdrive.ts`, `src/app/api/sync/calls/route.ts`) pulls data from TrackDrive directly, not from MOP.

**Implication for Milo-for-PPC:** The TrackDrive cross-connection (Q3 answer) is only needed if Milo-for-PPC wants to access MOP's historical call data. For ongoing data, the TrackDrive sync cron (already in milo-ops) continues pulling fresh data into Milo-for-PPC's own Supabase tables.

### Operator Dashboard Call Display

milo-ops Dashboard v2 has these TrackDrive-displaying blocks:

| Block | LOC | Data | Port Decision |
|-------|-----|------|--------------|
| `CallVolumeWidget` / equivalent | — | `call_records` aggregates | Carries over as-is |
| `FlaggedCallsBlock` | 3,355 | `call_records` + ConvoQC API | Carries over as-is |
| `CampaignHealthBlock` | 11,382 | `campaigns`, `call_records` | Carries over as-is |
| `PingBlock` | 1,268 | `ping_posts` | Carries over as-is |
| `CallDetailModal` | 19,415 | `call_records`, ConvoQC | Carries over as-is |
| `BuyerMonitoringBlock` | 10,400 | `call_records`, `disputes` | Carries over as-is |
| `PublisherMonitoringBlock` | 400 | `call_records`, blacklist | Carries over as-is |

**Decision: Port from milo-ops, not rebuild from call_logs reads.** milo-ops already has mature TrackDrive display UIs with ConvoQC integration, dispute correlation, and campaign health scoring. Rebuilding from raw `call_logs` reads would replicate 40K+ LOC of proven operator UX.

---

## 10. Phase 2 Execution Sequence

### Prerequisites (from Phase 1)

- [ ] Milo-for-PPC repo exists (forked from milo-ops)
- [ ] @milo/* packages wired (or direct Supabase fallback)
- [ ] PPC rules plugin created (`src/lib/ppc/rules.ts`, 77 rules)
- [ ] `npm run build` passes clean
- [ ] Auth works on staging URL

### Execution Order

Phase 2 is ordered by operator value and dependency chain:

| Step | UI | Depends On | LOC | Why This Order |
|------|-----|-----------|-----|---------------|
| 2.1 | Contract analysis upload (P4) | @milo/contract-analysis + PPC rules | ~600 | Core feature — operators need to analyze contracts. Unblocks everything else. |
| 2.2 | Issue card component | P4 results | ~350 | Shared component used by analysis, negotiation, and history pages. Build once. |
| 2.3 | Export toolbar (R4) | P4 results | ~300 | Operators need to export analysis results immediately after analyzing. |
| 2.4 | Analysis history (R1) | P4 + @milo/contract-analysis | ~500 | Operators need to find past analyses. Revision comparison for follow-ups. |
| 2.5 | Help/KB panel (R3) | @milo/ai-client | ~400 | Available on all contract pages. Build as shared component early. |
| 2.6 | Negotiation admin (P2) | P4 (create negotiation from analysis) | ~800 | Operator workflow: analyze → negotiate. |
| 2.7 | Negotiation review (P3) | P2 (generates review links) | ~400 | Public counterparty page — must exist before operators send review links. |
| 2.8 | Signing dashboard (P6) | @milo/contract-signing | ~200 | Operator document pipeline overview. |
| 2.9 | Signing page (P1) | P6 + @milo/contract-signing | ~1,200 | Public signing page — highest LOC, most complex. Build after signing dashboard validates data layer. |
| 2.10 | Rider templates (P5) | @milo/contract-analysis | ~400 | Nice-to-have for analysis recommendations. Not blocking. |
| 2.11 | Settings/playbook (R2) | PPC rules plugin | ~400 | Operators customize rule sensitivity. Not blocking — defaults work. |
| 2.12 | Dashboard display config wiring | All above | ~100 | Wire `VERTICAL_CONFIG.analysis` into existing ContractAnalysisDisplay block. |

**Total Phase 2:** ~5,250 LOC across 12 steps

### Parallelization Opportunities

| Parallel Track A (Contract Analysis) | Parallel Track B (Signing/Negotiation) |
|--------------------------------------|---------------------------------------|
| 2.1 Contract analysis upload | 2.8 Signing dashboard |
| 2.2 Issue card component | 2.9 Signing page |
| 2.3 Export toolbar | |
| 2.4 Analysis history | |
| 2.5 Help/KB panel | |

Track A and Track B can run in parallel after Phase 1 completes. Track A depends on @milo/contract-analysis; Track B depends on @milo/contract-signing. Negotiation (2.6, 2.7) depends on Track A completing.

---

## 11. Shared Components to Create

New shared components needed across multiple Phase 2 pages:

| Component | Used By | LOC | Source Reference |
|-----------|---------|-----|-----------------|
| `IssueCard` | P4 analysis, P2 negotiation, R1 history | ~350 | contract-checker `issue-card.tsx` |
| `IssueSidebar` | P4 analysis, P2 negotiation | ~176 | contract-checker `issue-sidebar.tsx` |
| `HelpPanel` | All contract pages | ~270 | contract-checker `help-panel.tsx` |
| `HelpTooltip` | All contract pages | ~72 | contract-checker `help-tooltip.tsx` |
| `ExportToolbar` | P4 analysis, R1 history | ~200 | contract-checker `export-toolbar.tsx` |
| `RoleSelector` | P4 upload | ~100 | contract-checker `role-selector.tsx` |
| `JurisdictionSelector` | P4 upload | ~100 | contract-checker `jurisdiction-selector.tsx` |
| `SeverityBadge` | All contract UIs | ~50 | New (simple — CRITICAL/HIGH/MEDIUM/LOW color badges) |
| `displayConfig` filter | All contract UIs | ~50 | New (Section 7 pattern) |

**Total shared component LOC:** ~1,368

These live in `src/components/contract/` (contract-specific shared components, distinct from milo-ops's existing `src/components/dashboard-v2/`).

---

## 12. Open Questions

| # | Question | Options | Urgency | Blocks |
|---|----------|---------|---------|--------|
| 1 | **Rider templates table location** | (a) Shared Supabase as @milo/contract-analysis extension (b) Vertical-layer table in Milo-for-PPC | LOW | P5 only |
| 2 | **Settings persistence** | (a) localStorage only (fast, per-browser) (b) `company_settings` table (shared across operators) (c) Both (localStorage cache + Supabase sync) | LOW | R2 only — defaults work without settings |
| 3 | **Signing token format** | (a) Reuse MOP's HMAC token scheme (backward compatible with 68 pending docs) (b) New @milo/contract-signing token scheme (cleaner, but breaks existing links without redirect) | MEDIUM | P1, link migration |
| 4 | **ContractAnalysisDisplay block rewrite** | (a) Wire existing milo-ops 14K LOC block to @milo/contract-analysis + display config (b) Replace with new lighter component using IssueCard pattern | MEDIUM | Dashboard integration |
| 5 | **MOP reference files in `_reference/`** | (a) Copy at Phase 2 start, delete after port complete (b) Keep permanently as documentation (c) Don't copy — developers read MOP repo directly | LOW | Process only |
| 6 | **Coder assignment for Phase 2** | (a) Coder-1 builds all (sequential, ~3-4 weeks) (b) Coder-1 Track A + Coder-2 Track B (parallel, ~2 weeks) (c) Wait for Coder-1 to finish v0.2 specs first | HIGH | Timeline |

---

## 13. Risk Matrix

| Risk | Probability | Impact | Mitigation |
|------|-------------|--------|------------|
| @milo/contract-signing client SDK not ready | HIGH | MEDIUM | Direct Supabase queries as fallback (same pattern as Phase 1) |
| 68 pending signing tokens break during cutover | LOW | HIGH | Dual-host + redirect (Section 8). Most will expire naturally within 30 days. |
| Issue card design mismatch between contract-checker and milo-ops design system | MEDIUM | LOW | contract-checker uses shadcn; milo-ops uses inline + Tailwind. Rebuild cards using milo-ops patterns, contract-checker layout as reference. |
| ContractAnalysisDisplay block (14K LOC) too entangled to wire display config | MEDIUM | MEDIUM | Replace with lighter IssueCard-based component (Option 4b above) |
| Negotiation round state diverges between MOP and shared Supabase | LOW | MEDIUM | Data already migrated (Decision 56). Only 1 active negotiation — resolve before cutover. |
| Playbook settings saved in localStorage lost on device change | MEDIUM | LOW | Acceptable for Phase 2. Phase 3 adds Supabase persistence. |

---

## Summary

| Category | Pages | New LOC | Source |
|----------|-------|---------|--------|
| Stay as-is (milo-ops base) | 14 | 0 | milo-ops fork |
| Port from MOP | 6 | ~3,600 | MOP → @milo/* rewire |
| Rebuild from contract-checker | 4 | ~1,600 | contract-checker UX patterns |
| Shared components | 9 | ~1,370 | Mixed sources |
| Reference only (MOP forever) | 11 | 0 | Not ported |
| **Phase 2 total** | **10 new pages** | **~5,250** | |

**Key decisions locked:**
- No aggregate scores anywhere in Milo-for-PPC (Directive #35)
- contract-checker's issue card design > MOP's flat list (better operator workflow)
- Dual-host + redirect for active counterparty links (no re-sending)
- TrackDrive display carries over from milo-ops as-is (no rebuild from raw call_logs)
- Analysis upload inherits contract-checker's role/jurisdiction/preset selectors
