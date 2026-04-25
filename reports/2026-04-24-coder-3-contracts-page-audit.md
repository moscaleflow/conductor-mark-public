# Contracts Page Audit — Vet-drop Contract Readiness

**Coder-3 Research | 2026-04-24**
**Directive context:** Audit /contracts page in milo-for-ppc — confirm readiness for new Vet-drop contracts (Part 1 of 3: Coder-1 wiring @milo/contract-analysis into /ask Vet pill)

---

## Audit Point 1: Page Route — `/contracts` list page

**Status: ABSENT**

No `/contracts` list page exists anywhere in milo-for-ppc. The only contract page route is:

- `/contract-review/[id]/page.tsx` (1604 lines) — detail view for a single contract

Entry points to contract review are scattered across 3 surfaces:
1. **ConversationModal** (`src/components/ConversationModal.tsx:1099`) — navigates to `/contract-review/${documentId}` after contract processing
2. **EntityDetailPanel** (`src/components/shared/EntityDetailPanel.tsx:2781,2856`) — "Review" buttons on per-entity contract version cards
3. **Negotiate API** (`src/app/api/contract/negotiate/route.ts:107`) — generates internal review URLs for review_later tasks

There is no standalone `/contracts` page for operators to browse, search, or filter all contracts independent of a specific entity.

---

## Audit Point 2: Detail View — `/contract-review/[id]`

**Status: PRESENT**

Full-featured detail page at `src/app/contract-review/[id]/page.tsx` (1604 lines). Key capabilities:

| Feature | Implementation |
|---|---|
| Contract loading | Supabase browser client, fetches by UUID |
| Analysis polling | 3s interval, 310s max, auto-triggers `/api/contract/analyze-bg` |
| Clause display | Severity-grouped (critical/high/medium/low), expandable cards |
| Clause decisions | accept/reject/counter per clause, persisted to localStorage |
| Negotiation creation | POST to `/api/contract/negotiate`, returns review URL |
| Review Later | Creates action_item task via negotiate endpoint |
| Progress tracking | Sticky bottom bar with decision counts |
| Link generation | "Submit Response + Generate Link" creates negotiation + copy-paste link |
| Redline detection | `doc.status === 'redlined' || doc.status === 'countered'` |

**Fields consumed from `contract_documents`:**
id, contract_group_id, counterparty_name, document_type, version, title, status, risk_level, analysis_result, signing_token, created_at

**`analysis_result` shape consumed:**
- `flagged_clauses` / `clauses` array (FlaggedClause interface)
- `decisions` map
- `our_company`, `counterparty`, `our_role`, `counterparty_role`
- `summary.overall_risk`, `summary.total_issues`, `summary.favorable_clauses`
- `improvements` array
- `analysis_status` ('processing' | 'complete' | 'failed')
- `extracted_text`

**FlaggedClause interface (inline in page):**
```
id, section, title, originalText, changedText, riskLevel ('critical'|'high'|'medium'|'low'|'favorable'),
perspective, isSneakedIn, explanation, plainEnglish, impactToUs, recommendation,
suggestedRevision, legalReasoning, negotiationLeverage
```

---

## Audit Point 3: Component Inventory

**Status: PRESENT (two parallel systems)**

### System A: `@milo/contract-analysis` components (used by operator dashboard)

| Component | Path | Types | Notes |
|---|---|---|---|
| IssueCard | `src/components/contract/IssueCard.tsx` (220 lines) | `ContractIssue` from `@milo/contract-analysis` | Uses `verticalConfig.analysis.perIssueFields` for field display |
| IssueSidebar | `src/components/contract/IssueSidebar.tsx` (90 lines) | `RiskLevel` UPPERCASE enum | Severity-sorted nav sidebar |
| SeverityBadge | `src/components/contract/SeverityBadge.tsx` (38 lines) | `RiskLevel` from `@milo/contract-analysis` | Colors from `verticalConfig.analysis.severityColors` |
| RoleSelector | `src/components/contract/RoleSelector.tsx` (48 lines) | `AnalysisRole` from `@milo/contract-analysis` | Roles from `verticalConfig.analysis.roles` |
| JurisdictionSelector | `src/components/contract/JurisdictionSelector.tsx` (48 lines) | — | Two-party consent state picker |

### System B: Inline components (used by contract-review detail page + conversation)

| Component | Path | Types | Notes |
|---|---|---|---|
| ContractAnalysisDisplay | `src/components/shared/ContractAnalysisDisplay.tsx` (414 lines) | Own `FlaggedClause` interface (local) | accept/reject/counter + negotiation creation |
| ContractClauseList | `src/components/operator/ContractClauseList.tsx` (202 lines) | Own `ClauseItem` interface (snake_case) | Fetches from `/api/operator/clauses`, UPPERCASE severity |
| ClauseCard (inline) | Inside `contract-review/[id]/page.tsx` | Own `FlaggedClause` (camelCase) | Not extracted |
| DecisionBtn (inline) | Inside `contract-review/[id]/page.tsx` | — | Not extracted |

### Type schema divergence (critical finding):

Three separate type systems for the same data:

| System | Risk levels | Field casing | Key interface |
|---|---|---|---|
| `@milo/contract-analysis` | UPPERCASE: `CRITICAL\|HIGH\|MEDIUM\|LOW` | snake_case: `risk_level`, `clause_reference`, `plain_summary` | `ContractIssue` |
| `contract-review/[id]` page | lowercase: `critical\|high\|medium\|low\|favorable` | camelCase: `riskLevel`, `plainEnglish`, `impactToUs` | `FlaggedClause` (inline) |
| `ContractAnalysisDisplay` | lowercase: `high\|medium\|low` (no critical!) | camelCase: `riskLevel`, `changedText` | `FlaggedClause` (local) |

---

## Audit Point 4: Data Flow Gaps

**Status: PARTIAL — significant gaps**

### 4a. Two analysis pipelines

| Pipeline | Entry point | Model | Prompt source | Output schema |
|---|---|---|---|---|
| **Heavyweight (UI)** | `/api/contract/analyze-bg` | @milo/ai-client (judgment/Opus) | `contract-analysis-prompt.ts` (495 lines, 60+ rules, PUB/BUY checks) | `flagged_clauses[]` with camelCase FlaggedClause |
| **Tool (Milo-callable)** | `lib/tools/analyze-contract.ts` | @milo/ai-client (judgment/Opus) | Inline 30-line system prompt | `clauses_analyzed[]` with snake_case AnalyzedClause |

These two pipelines produce different schemas stored in the same `contract_documents.analysis_result` JSONB column. The detail page reads `flagged_clauses` or `clauses` — it handles both keys — but the field names and severity value casing differ.

### 4b. Contract ingestion flow

```
PDF/DOCX drop → /api/contract/process (extract text + metadata detection)
           → contract_documents row created (status: draft/redlined)
           → /api/contract/analyze-bg triggered (fire-and-forget)
           → analysis_result.analysis_status: processing → complete/failed
           → /contract-review/[id] polls until complete
```

### 4c. No blacklist/vet integration in contracts

The contract-review page has zero references to:
- Blacklist screening
- Vet pill
- `/ask` endpoint
- `@milo/blacklist`

The Vet pill (in `/ask`) currently has no awareness of contract data. Part 1 of the Vet-drop wiring needs to bridge this gap.

### 4d. Missing: contract list API

There is no API to list/search/filter contracts across all entities. The only query endpoint is:
- `GET /api/contract/documents?entity=<name>` — filters by counterparty name (ilike match)

No general-purpose list endpoint exists for a `/contracts` page.

---

## Audit Point 5: URL Routing Readiness

**Status: PARTIAL — one critical mismatch**

### Internal routes (working):

| Route | Purpose | Used by |
|---|---|---|
| `/contract-review/[id]` | Detail page (operator-facing) | ConversationModal, EntityDetailPanel, negotiate API |
| `/c/[slug]` | Short URL redirect | `contract_documents.short_slug` → `public_review_url` 302 redirect |
| `/api/contract/process` | PDF/DOCX ingestion | ConversationModal drop zone |
| `/api/contract/analyze-bg` | Heavyweight analysis | Detail page auto-trigger |
| `/api/contract/analyze-proxy` | MOP proxy | Legacy path |
| `/api/contract/documents` | Entity contract list | EntityDetailPanel |
| `/api/contract/negotiate` | Negotiation creation | Detail page submit |

### Critical URL mismatch:

`publish-contract-for-buyer-review.ts:130` generates:
```
const publicReviewUrl = `${baseUrl}/contracts/review/${row.id}`;
```

But the actual page route is `/contract-review/[id]`, not `/contracts/review/[id]`. Any published buyer review links will 404.

The `/c/[slug]` short URL route reads `public_review_url` from the DB and redirects to it — so short URLs also redirect to the wrong path.

### Missing public review page:

The `/contract-review/[id]` page uses the Supabase browser client (anon key), which means it could theoretically serve as a public review page. However, it currently:
- Loads ALL analysis data (not just buyer-relevant fields)
- Shows operator-facing decision controls (accept/reject/counter)
- Has no auth gating or role-based view

A proper public buyer review page does not exist.

---

## Audit Point 6: State Display — Contract Status Lifecycle

**Status: PRESENT (implicit, not documented)**

Observed statuses across all contract code:

| Status | Set by | Meaning |
|---|---|---|
| `draft` | `/api/contract/process` | Initial upload, pre-analysis |
| `redlined` | `/api/contract/process`, `ingest-returned-contract` | Buyer-returned or redline-detected upload |
| `sent` | `publish-contract-for-buyer-review` | Published for buyer review |
| `countered` | `/api/contract/negotiate` (submit action) | Operator submitted decisions |
| `processing` (in analysis_status) | `/api/contract/analyze-bg` | Analysis in progress |
| `complete` (in analysis_status) | `/api/contract/analyze-bg` | Analysis finished |
| `failed` (in analysis_status) | `/api/contract/analyze-bg` | Analysis errored |

Note: `status` (document lifecycle) and `analysis_result.analysis_status` (analysis pipeline) are separate fields. The detail page uses both.

No `signed`, `expired`, `archived`, or `rejected` statuses exist yet.

---

## READY-FOR-PART-1

**Partially ready.** Coder-1 can wire `@milo/contract-analysis` into the Vet pill for Part 1, but needs to account for:

1. **Type schema divergence** — the Vet pill will need to decide which schema to use. `@milo/contract-analysis` types (`ContractIssue`, UPPERCASE `RiskLevel`) are the canonical extraction, but the detail page consumes camelCase `FlaggedClause`. If Vet surfaces contract data in its response, it must match whichever component renders it.

2. **No blacklist→contract bridge exists** — the Vet pill currently screens entities via `@milo/blacklist` but has no path to query `contract_documents` for existing contract history. Part 1 must add this query (likely via the existing `/api/contract/documents?entity=` endpoint or a direct Supabase call).

3. **URL mismatch in publish tool** — `publish-contract-for-buyer-review.ts` generates `/contracts/review/${id}` but the page is at `/contract-review/${id}`. This is a pre-existing bug, not a Part 1 blocker, but should be fixed concurrently.

4. **Contract detail page has no Vet-drop UI** — no section exists for "vet findings" or "blacklist alerts" on the contract-review page. Part 1 likely surfaces Vet data in the `/ask` response rather than on the contract page itself.

---

## GAPS-FOR-PART-2

| Gap | Priority | Effort |
|---|---|---|
| No `/contracts` list page | HIGH | New page: filter by status/entity/type, link to detail |
| URL mismatch: `/contracts/review/` vs `/contract-review/` | HIGH | One-line fix in `publish-contract-for-buyer-review.ts` |
| Three divergent type schemas for contract issues | HIGH | Normalize to `@milo/contract-analysis` types, add camelCase adapter |
| No public buyer review page (distinct from operator view) | MEDIUM | New route with read-only clause display, no decision controls |
| `ContractAnalysisDisplay` missing `critical` risk level | MEDIUM | Add 'critical' to its riskLevel union type |
| No blacklist screening on contract counterparties | MEDIUM | Wire `@milo/blacklist` screenEntity into contract process pipeline |
| `analyze-contract` tool uses 30-line prompt vs 495-line prompt | MEDIUM | Consolidate or fork explicitly |
| No signed/expired/archived contract statuses | LOW | Add lifecycle states as needed |
| ClauseCard and DecisionBtn not extracted from detail page | LOW | Extract to `src/components/contract/` |
| Contract text stored in `notes` column (v1 hack) | LOW | `analyze-contract.ts:293` documents this as v1.1 follow-up |

---

## COMPONENT-REUSE-MAP

| Component | Can be reused for Vet-drop? | Notes |
|---|---|---|
| `@milo/contract-analysis` types | YES | Canonical types for ContractIssue, RiskLevel, AnalysisRole |
| `IssueCard` | YES | Already uses `verticalConfig` for field display; add Vet-drop fields via config |
| `IssueSidebar` | YES | Severity nav works as-is |
| `SeverityBadge` | YES | Uses `@milo/contract-analysis` RiskLevel + verticalConfig colors |
| `RoleSelector` | YES | `verticalConfig.analysis.roles` already has publisher/buyer |
| `JurisdictionSelector` | YES | Two-party consent state picker, useful for contract context |
| `ContractAnalysisDisplay` | PARTIAL | Missing `critical` risk level; type mismatch with @milo types |
| `ContractClauseList` | YES | Already uses UPPERCASE severity matching @milo types |
| `contract-analysis-prompt.ts` | YES | 60+ rules, PUB/BUY checks — this IS the analysis engine |
| `blacklist-client.ts` | YES | `screenEntity()` with tenant='tlp' — wire into contract process |
| `/api/contract/documents` | YES | Entity-scoped contract query — Vet pill can call this |
| `/c/[slug]` short URL | YES | Works once publish tool URL is fixed |
