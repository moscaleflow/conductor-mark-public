---
directive: "milo-ops parallel contract system — three-way inventory"
lane: research
coder: Coder-3
started: 2026-04-19 ~22:00 MDT
completed: 2026-04-19 ~22:45 MDT
---

# PPC Contract System Three-Way Comparison — 2026-04-19

**Purpose:** Canonical reference for consolidating three contract analyzers (MOP, milo-ops, contract-checker) into one PPC vertical product.

## 1. Executive Summary

The three systems evolved from a common ancestor (contract-checker) into increasingly specialized forks. **contract-checker** is the UI-first standalone tool (Tagalog, settings, help docs, GPT-4o-mini). **MOP** has the deepest analytical engine (73-rule registry, scoring, post-processing, OCR). **milo-ops** has the richest prompt engineering (60+ inline rules, template comparison, MSA improvements, plain_english + impact_to_us output) and the best versioning (contract_group_id + parent chains). A consolidated system needs MOP's scoring/post-processing + milo-ops's prompt quality and versioning + contract-checker's UI and export features.

## 2. Systems Overview

| Dimension | contract-checker | MOP | milo-ops |
|-----------|-----------------|-----|----------|
| Repo | `~/contract-checker/` | `~/MOP/lib/contract-checker/` | `~/Miloops3.18/milo-ops/` |
| Total LOC | ~8,200 | ~6,370 | ~2,527 |
| AI model | GPT-4o-mini (OpenAI) | Claude Sonnet 4.5 (Anthropic) | Claude Opus 4.6 via @milo/ai-client |
| AI temp | 0.3 | varies (0.0-0.2) | 0.2 |
| Max tokens | 3,000 | 16,000 | 4,096-16,000 |

## 3. Feature Comparison Matrix

### 3a. Rule Engine

| Feature | contract-checker | MOP | milo-ops |
|---------|-----------------|-----|----------|
| Rule count | ~25 playbook rules | **73 rules** (43 PUB_*, 26 BUY_*, 2 legacy, 6 phantom) | ~60+ rules |
| Storage | Code arrays (PlaybookRule objects) | Centralized registry (RuleMeta objects) | **Inline in prompt** (natural language) |
| Rule ID format | Short slugs (`payment_terms`) | Structured (`PUB_FIN_FINALITY_MISSING`) | Same PUB_*/BUY_* as MOP |
| Rule metadata | id, name, description, category, severity, threshold, keywords, prompt_instruction | rule_id, title, category, defaultSeverity, baseWeight, brokerExplanation, riderRefs, tags, isExistential | None (prose only) |
| Buyer rules | Minimal (1 rule) | **Full** (26 BUY_* rules) | **Full** (embedded in prompt) |

### 3b. Scoring & Post-Processing

| Feature | contract-checker | MOP | milo-ops |
|---------|-----------------|-----|----------|
| Weighted scoring | No | **Yes** — `scoring.ts` (444 LOC) | No |
| Severity weights | N/A | LOW:5, MEDIUM:12, HIGH:20, CRITICAL:35 | N/A |
| Normalized score | N/A | 0-100 scale (max=200) | N/A |
| Risk buckets | AI-assigned HIGH/MEDIUM/LOW | Computed: CRITICAL>=75, HIGH>=55, MEDIUM>=30, LOW<30 | AI-assigned high/medium/low |
| Existential risk detection | No | **Yes** — `hasExistentialRisks()` | No |
| Post-processing | **No** | **Yes** — 9-step pipeline (1,112 LOC) | **No** |
| Deduplication | N/A | Concept-group merging + exact rule_id dedup | N/A |
| Reclassification | N/A | Liability cap detection, indemnity carve-out | N/A |
| Missed-rule injection | N/A | Injects 2 rules if AI missed them (12 regex patterns) | N/A |
| Revision fix-ups | N/A | Market-credible templates, broker-safe/buyer-safe | N/A |
| Role filtering | N/A | Drops wrong-role rules | N/A |

### 3c. Profiles & Presets

| Feature | contract-checker | MOP | milo-ops |
|---------|-----------------|-----|----------|
| Analysis profiles | No | **Yes** — TEST, SCALE, BROKER, BUYER (weight multipliers) | No |
| Playbook presets | **Yes** — full_audit, publisher_defensive, buyer_aggressive, quick_review | **Yes** (inherited) | No |
| Threshold profiles | **Yes** — strict, standard, relaxed | **Yes** (inherited) | No |

### 3d. Jurisdiction Detection

| Feature | contract-checker | MOP | milo-ops |
|---------|-----------------|-----|----------|
| Method | AI-powered (GPT-4o-mini call) | **Regex** — `jurisdiction.ts` (110 LOC) | Prompt-instructed (AI does it) |
| Confidence levels | No | **Yes** — HIGH/MEDIUM/LOW/UNKNOWN | No |
| Source tracking | No | **Yes** — governing_law, venue_clause, address_pattern, etc. | No |

### 3e. Two-Party Consent Handling

| Feature | contract-checker | MOP | milo-ops |
|---------|-----------------|-----|----------|
| State list | In playbook `jurisdiction_triggers` | `TWO_PARTY_CONSENT_STATES` Set | `TWO_PARTY_CONSENT_STATES` array |
| Escalation | Rule auto-enabled | **`applyTwoPartyConsentEscalation()`** — promotes recording rules to CRITICAL | Prompt instruction only |

### 3f. Revision Comparison

| Feature | contract-checker | MOP | milo-ops |
|---------|-----------------|-----|----------|
| Exists | **Yes** — 450 LOC | Merge only (157 LOC) | **Yes** — 334 LOC |
| Method | GPT-4o comparison | Fuzzy text replacement (4-tier matching) | Claude Opus diff |
| Sneaked-in detection | **Yes** — `unexpected_changes` array | No | **Yes** — `isSneakedIn` boolean |
| Template change detection | **Yes** — detects different templates | No | No |
| Revision numbering | Integer counter | No | Semantic versioning (1.0 -> 1.1) |

### 3g. Export Formats

| Feature | contract-checker | MOP | milo-ops |
|---------|-----------------|-----|----------|
| DOCX | **Yes** — `export.ts` (224 LOC) | **Yes** — 3 formats (1,132 LOC) | No |
| PDF | Referenced but not implemented | No | No |
| Redline text | **Yes** — plain text | **Yes** — track-changes markup | **Yes** — track-changes or clean |
| Rider generation | No | **Yes** — per-rule rider refs | No |
| Teams message | **Yes** | No | No |

### 3h. Translation & Help

| Feature | contract-checker | MOP | milo-ops |
|---------|-----------------|-----|----------|
| Translation | **Yes** — English + Tagalog (218 LOC) | No | No |
| Help/knowledge base | **Yes** — 316 LOC + glossary (EN+TL) | No | No |
| AI help chatbot | **Yes** — `/api/help` endpoint | No | No |
| Settings UI | **Yes** — playbook config (515 LOC) | No | No |

### 3i. Negotiation Workflow

| Feature | contract-checker | MOP | milo-ops |
|---------|-----------------|-----|----------|
| Multi-round negotiation | No | **Yes** — full state machine | Partial — version chain |
| Accept/reject per clause | **Yes** — pending/accepted/rejected/edited | **Yes** + counter status | **Yes** — in review UI |
| Counter-proposal | No | **Yes** | **Yes** |

### 3j. Version Tracking

| Feature | contract-checker | MOP | milo-ops |
|---------|-----------------|-----|----------|
| contract_group_id | No | No | **Yes** |
| Parent version chain | No | No | **Yes** — `parent_version_id` |
| Version numbering | Integer revision counter | No | **Semantic** (1.0, 1.1, 1.2) via `bumpVersion()` |
| Status tracking | N/A | NegotiationStage type | **Yes** — draft, uploaded, sent, redlined, countered, signed |

### 3k. AI Output Fields

| Feature | contract-checker | MOP | milo-ops |
|---------|-----------------|-----|----------|
| plain_english | No (uses `problem_explanation`) | Yes (`plain_summary`) | **Yes** — "1-2 sentences a non-lawyer can understand" |
| impact_to_us | No | No | **Yes** — "This affects {company} because..." |
| recommendation format | `negotiation_leverage` only | `suggested_revision` + `negotiation_leverage` | **Yes** — PUSH BACK / ACCEPT WITH MODIFICATION / ACCEPT AS-IS / REJECT |
| MSA improvements | No | No | **Yes** — 3-6 template improvement suggestions |
| Template comparison | Partial | No | **Yes** — CHANGED/REMOVED/ADDED/RISK |

### 3l. Infrastructure

| Feature | contract-checker | MOP | milo-ops |
|---------|-----------------|-----|----------|
| OCR | No | **Yes** — pngjs fallback when PDF text < 500 chars | No |
| Bot/tool integration | No | **Yes** — `bot-helpers.ts` (403 LOC) | **Yes** — Milo tool system (4 tools) |
| Short URL/publishing | No | No | **Yes** — 6-char hex slug, `/c/{slug}` |
| Field extraction (IO) | No | **Yes** — `field-mapper.ts` (189 LOC) | No |
| Document hash caching | Yes | **Yes** — `computeDocumentHash()` with version | No |

### 3m. UI

| Feature | contract-checker | MOP | milo-ops |
|---------|-----------------|-----|----------|
| Standalone app | **Yes** — full Next.js app | Integrated in MOP dashboard | Page in milo-ops |
| File upload | **Yes** — drag-drop (25MB max) | Via MOP file handling | N/A (text from DB) |
| Text paste | **Yes** | No | N/A |
| Issue cards | **Yes** — 350 LOC, expand/collapse | Integrated | Severity-grouped with inline decisions |
| AI assistant panel | **Yes** — 437 LOC per-issue AI chat | No | No |
| History page | **Yes** — 1,630 LOC with search | No | No |
| Progress indicators | **Yes** — animated, Tagalog messages | N/A | Polling with fun facts |
| AI persona | No (generic "expert attorney") | **Yes** — "Alex, Senior Analyst at Lead Penguin" | **Yes** — "Alex, Senior Analyst at {company}" |

### 3n. Database

| Feature | contract-checker | MOP | milo-ops |
|---------|-----------------|-----|----------|
| Primary table | `analyses` | `contract_analyses` (33 rows) | `contract_documents` |
| Key columns | document_hash, role, jurisdiction, raw_text, analysis_result | + scoring, profiles, negotiation_stage, parent chain | + contract_group_id, parent_version_id, version, status, short_slug, diff_from_previous, public_review_url |

## 4. Feature Coverage Heat Map

```
Feature                      contract-checker  MOP    milo-ops
─────────────────────────────────────────────────────────────
Rule registry (structured)      Partial        FULL    None
Scoring engine                  None           FULL    None
Post-processing pipeline        None           FULL    None
Analysis profiles               None           FULL    None
Playbook presets                Yes            Yes     None
Threshold profiles              Yes            Yes     None
Jurisdiction (with confidence)  AI-based       FULL    Prompt
Revision comparison             FULL           Merge   FULL
Sneaked-in detection            Yes            None    Yes
DOCX export                     Yes            FULL    None
Teams message format            Yes            None    None
Tagalog translation             Yes            None    None
Help / knowledge base           Yes            None    None
OCR                             None           Yes     None
Negotiation workflow            Partial        FULL    FULL
Version tracking                Partial        None    FULL
Short URL / publishing          None           None    Yes
Bot / tool integration          None           Yes     Yes
IO field extraction             None           Yes     None
Template comparison mode        Partial        None    FULL
plain_english output            None           Yes     Yes
impact_to_us output             None           None    Yes
MSA improvements                None           None    Yes
AI persona (Alex)               None           Yes     Yes
Two-party consent escalation    Partial        FULL    Prompt
```

## 5. What Each System Has That The Others Don't

### contract-checker only

1. **Tagalog translation** — full UI in English + Tagalog (218 LOC)
2. **Help/knowledge base** — 316 LOC with glossary (EN+TL), AI chatbot
3. **Settings UI** — playbook configuration page (515 LOC)
4. **History page** — 1,630 LOC with search, filtering, revision tracking
5. **Text paste input** — alternative to file upload
6. **Teams message format** — `generateTeamsMessage()` for notifications
7. **Template change detection** — detects if counterparty swapped templates entirely
8. **Per-rule numeric thresholds** — configurable per-rule (not just presets)

### MOP only

1. **73-rule structured registry** — machine-readable RuleMeta with baseWeight, riderRefs, tags, isExistential, isRecordingRelated
2. **Weighted scoring engine** — 444 LOC, normalized 0-100, risk buckets with decision guidance
3. **Post-processing pipeline** — 1,112 LOC, 9-step dedup/reclassify/inject/fix-up pipeline
4. **Analysis profiles** — TEST/SCALE/BROKER/BUYER with per-rule weight multipliers
5. **Two-party consent escalation function** — programmatic escalation to CRITICAL
6. **Existential risk detection** — `hasExistentialRisks()` flags deal-breakers
7. **OCR fallback** — pngjs-based OCR when PDF text extraction yields < 500 chars
8. **IO field extraction** — `field-mapper.ts` normalizes payout, pay terms, lead type, transfer type
9. **DOCX export** — 3 formats (standard, Channel Edge, analysis DOCX), 1,132 LOC
10. **Rider generation** — per-rule rider section references in registry

### milo-ops only

1. **Template comparison mode** — compares submitted doc against standard template, flags CHANGED/REMOVED/ADDED/RISK
2. **MSA improvement suggestions** — 3-6 suggestions for improving your own standard template
3. **impact_to_us output** — "This affects {company} because..."
4. **Semantic versioning** — `bumpVersion()` increments 1.0 -> 1.1
5. **contract_group_id** — groups all versions of same contract
6. **Parent version chain** — `parent_version_id` links revisions
7. **Contract publishing** — `publish-contract-for-buyer-review.ts` with short slugs
8. **6-status workflow** — draft, uploaded, sent, redlined, countered, signed
9. **Milo tool integration** — 4 contract tools in the Milo AI assistant system
10. **Structured recommendation format** — PUSH BACK / ACCEPT WITH MODIFICATION / ACCEPT AS-IS / REJECT

## 6. Consolidation Recommendations

### What the consolidated PPC vertical product should have

| Source | Feature to take |
|--------|----------------|
| MOP | Rule registry structure (machine-readable, not prompt-embedded) |
| MOP | Scoring engine (weighted, normalized, with risk buckets) |
| MOP | Post-processing pipeline (dedup, reclassify, missed-rule injection) |
| MOP | Analysis profiles (TEST/SCALE/BROKER/BUYER) |
| MOP | Two-party consent escalation (programmatic) |
| MOP | OCR fallback |
| MOP | DOCX export (3 formats) |
| MOP | Rider generation with per-rule refs |
| MOP | IO field extraction |
| milo-ops | Template comparison mode |
| milo-ops | MSA improvement suggestions |
| milo-ops | impact_to_us + plain_english output fields |
| milo-ops | Semantic versioning + contract_group_id + parent chain |
| milo-ops | 6-status workflow (draft -> ... -> signed) |
| milo-ops | PUSH BACK/ACCEPT/REJECT recommendation format |
| milo-ops | Short URL publishing |
| milo-ops | Milo tool integration |
| contract-checker | Tagalog translation |
| contract-checker | Help/knowledge base |
| contract-checker | Settings UI for playbook configuration |
| contract-checker | History page |
| contract-checker | Sneaked-in detection (both contract-checker and milo-ops have this) |
| contract-checker | Template change detection |

### What to discard

| Source | Feature to discard | Reason |
|--------|-------------------|--------|
| contract-checker | GPT-4o-mini as AI model | Switch to Claude (already done in MOP + milo-ops) |
| contract-checker | Standalone app architecture | Consolidate into milo-ops |
| milo-ops | Inline prompt-embedded rules | Move to structured registry (MOP pattern) |
| milo-ops | No scoring engine | Use MOP's scoring |
| milo-ops | No post-processing | Use MOP's pipeline |
| MOP | Dead code (phantom rule IDs, unused state machine) | Clean up |

### Suggested architecture

```
@milo/contracts (horizontal engine)
├── scoring engine (from MOP)
├── post-processing pipeline framework (from MOP)
├── jurisdiction detection (from MOP)
├── text merge engine (from MOP)
├── text extraction + OCR (from MOP)
└── export framework (from MOP)

Milo-for-PPC vertical (consumer)
├── 73-rule registry (from MOP, enhanced with milo-ops prompt quality)
├── system prompt (merge MOP's structured instructions + milo-ops's output fields)
├── analysis profiles (from MOP)
├── playbook presets + threshold profiles (from contract-checker/MOP)
├── template comparison mode (from milo-ops)
├── MSA improvements (from milo-ops)
├── version tracking (from milo-ops)
├── Tagalog translation (from contract-checker)
├── help/knowledge base (from contract-checker)
├── IO field extraction (from MOP)
├── DOCX export (from MOP)
├── rider generation (from MOP)
└── UI (merge contract-checker history/settings + milo-ops review)
```

## 7. Open Questions for Mark

1. **AI model for consolidated system?** contract-checker uses GPT-4o-mini, MOP uses Claude Sonnet 4.5, milo-ops uses Claude Opus 4.6. Opus produces highest quality but costs more. Is Sonnet 4.5 the right tier for analysis, with Opus reserved for judgment calls?

2. **Rules: registry or prompt?** MOP stores 73 rules as code objects. milo-ops embeds 60+ rules as prompt text. The prompt-embedded rules produce richer output (plain_english, impact_to_us, recommendations) but aren't machine-queryable. Recommendation: structured registry (MOP) with prompt-generation that includes milo-ops's output field instructions.

3. **contract-checker's future.** Once consolidated, does the standalone contract-checker app get archived? Its unique features (Tagalog, help, settings, history) should move to the consolidated system, and the GPT-4o-mini backend becomes obsolete.

4. **Version tracking schema.** milo-ops has `contract_group_id` + `parent_version_id` on `contract_documents`. MOP has `contract_group_id` + `parent_analysis_id` on `contract_analyses` (added later). Should the consolidated system use milo-ops's schema (which also includes `version`, `status`, `short_slug`, `diff_from_previous`)?

5. **Template comparison ownership.** Is template comparison a horizontal feature (any contract type could compare against a template) or vertical (only PPC contracts have standard templates)?

6. **Consolidation sequencing.** Should consolidation happen before or after `@milo/contracts` extraction? If before: consolidate milo-ops + MOP + contract-checker into one PPC system, then extract horizontal engine. If after: extract horizontal engine from MOP first, then consolidate the vertical layer.
