---
directive: "Milo-for-PPC vertical consolidation planning audit"
lane: ops
coder: Coder-4
started: 2026-04-20
completed: 2026-04-20
foundation: reports/2026-04-19-coder-3-ppc-contract-three-way.md (commit 40e2811)
---

# Milo-for-PPC Contract Consolidation Plan

## 1. Executive Summary

Three contract systems (MOP, milo-ops, contract-checker) need to consolidate into one PPC vertical product sitting on top of the @milo/contract-analysis, @milo/contract-signing, and @milo/contract-negotiation horizontal primitives. This plan maps every feature to its destination, proposes data reconciliation, identifies the winning UI, deduplicates 200+ rules, and sequences the rollout for Morgan.

The key architectural move: MOP's engine becomes the horizontal primitives (already shipped), milo-ops's prompt quality and versioning become the PPC vertical plugins, and contract-checker's UI features (Tagalog, help, settings, history) become Milo-for-PPC product features.

---

## 2. Feature → Destination Map

### (a) MOP features → @milo/contract-analysis plugins (PPC RuleDefinition[])

| MOP Feature | LOC | Destination | Notes |
|-------------|-----|-------------|-------|
| 73-rule structured registry | ~800 | PPC `RuleDefinition[]` plugin | 43 PUB_*, 26 BUY_*, 2 legacy, 6 phantom (discard phantoms) |
| Scoring weights (baseWeight per rule) | in registry | PPC `ScoringModifier[]` plugin | LOW:5, MEDIUM:12, HIGH:20, CRITICAL:35 |
| Analysis profiles (TEST/SCALE/BROKER/BUYER) | ~100 | PPC config layer | Weight multipliers per profile |
| Playbook presets (full_audit, publisher_defensive, etc.) | ~80 | PPC config layer | Preset = profile + threshold combo |
| Threshold profiles (strict/standard/relaxed) | ~60 | PPC config layer | Min severity filter |
| Two-party consent escalation | ~40 | PPC `PatternPlugin[]` | `applyTwoPartyConsentEscalation()` → PatternPlugin |
| Existential risk detection | ~30 | PPC `ScoringModifier[]` | `hasExistentialRisks()` post-scoring check |
| OCR fallback (pngjs) | ~80 | Stay in horizontal (already in @milo/contract-analysis `extractText()`) | Already extracted |
| DOCX export (3 formats) | 1,132 | PPC vertical layer | Channel Edge + standard + analysis formats |
| Rider generation (per-rule refs) | in registry | PPC vertical layer | `riderRefs` field on RuleMeta |
| IO field extraction | 189 | PPC vertical layer (`field-mapper.ts`) | Payout, pay terms, lead type, transfer type |
| Bot helpers | 403 | PPC vertical layer | Telegram/bot integration |
| Document hash caching | ~50 | Stay in horizontal (already extracted) | `computeDocumentHash()` |

### (b) milo-ops features → PPC vertical product

| milo-ops Feature | LOC | Destination | Notes |
|------------------|-----|-------------|-------|
| 60+ inline prompt rules | ~600 | Merge into PPC `RuleDefinition[]` (see rule dedup §6) | Move from prompt-embedded to structured registry |
| Template comparison mode | 334 | PPC vertical feature | CHANGED/REMOVED/ADDED/RISK per clause |
| MSA improvement suggestions | in prompt | PPC vertical feature | 3-6 suggestions for template improvement |
| `impact_to_us` output field | in prompt | PPC prompt config | "This affects {company} because..." |
| `plain_english` output field | in prompt | PPC prompt config | Already in @milo/contract-analysis as output field |
| PUSH BACK / ACCEPT / REJECT format | in prompt | PPC prompt config | Recommendation enum |
| Semantic versioning (`bumpVersion()`) | ~30 | PPC vertical layer | 1.0 → 1.1 → 2.0 |
| `contract_group_id` + parent chain | schema | PPC vertical layer (or @milo/contract-negotiation) | Already in negotiation_records |
| 6-status workflow | schema | Already in @milo/contract-signing (signing lifecycle) + @milo/contract-negotiation (negotiation lifecycle) | Mapped across two primitives |
| Short URL publishing | ~80 | Already in @milo/contract-signing (`resolveShortCode()`) | Already extracted |
| Milo tool integration (4 tools) | ~200 | PPC vertical layer | Tool definitions for Milo AI assistant |
| Sneaked-in detection (`isSneakedIn`) | ~50 | PPC `PatternPlugin[]` | Revision comparison flag |

### (c) contract-checker features → Milo-for-PPC product features

| contract-checker Feature | LOC | Destination | Notes |
|--------------------------|-----|-------------|-------|
| Tagalog translation | 218 | PPC product i18n layer | English + Tagalog UI strings |
| Help/knowledge base + glossary | 316 | PPC product help system | EN+TL glossary, AI chatbot |
| Settings UI (playbook config) | 515 | PPC product settings page | Profile/preset/threshold config |
| History page with search | 1,630 | PPC product history page | Search, filtering, revision tracking |
| Text paste input | ~50 | PPC product upload flow | Alternative to file upload |
| Teams message format | ~30 | PPC notification layer | `generateTeamsMessage()` |
| Template change detection | ~40 | PPC `PatternPlugin[]` | Detects if counterparty swapped templates |
| Per-rule numeric thresholds | ~60 | PPC config layer | Configurable per-rule (extends presets) |
| AI assistant panel (per-issue chat) | 437 | PPC product feature | Per-issue conversational AI |
| Progress indicators (Tagalog messages) | ~80 | PPC product UX | Animated progress with bilingual messages |
| File upload (drag-drop, 25MB) | ~100 | PPC product upload flow | Already standard pattern |

### Features to discard

| Feature | Source | Reason |
|---------|--------|--------|
| GPT-4o-mini as AI model | contract-checker | Anthropic-only policy (Decision 27) |
| Standalone app architecture | contract-checker | Consolidate into Milo-for-PPC |
| Inline prompt-embedded rules | milo-ops | Move to structured registry |
| Phantom rule IDs (6) | MOP | Dead code — never matched |
| Legacy rules (2) | MOP | Superseded by structured rules |

---

## 3. Data Reconciliation: Three contract_documents Tables

### Current state

| Repo | Table | Rows (est.) | Supabase project | Entity FK |
|------|-------|-------------|------------------|-----------|
| MOP | `contract_analyses` | 33 | wjxtfjaixkoifdqtfmqd (frozen) | `publisher_id`, `buyer_id` → partners |
| milo-ops | `contract_documents` | 5-10 | wjxtfjaixkoifdqtfmqd (shared MOP Supabase) | `prospect_id` → prospect_pipeline |
| contract-checker | `analyses` | unknown (standalone) | separate project | `user_id` → auth.users |

### Column comparison

| Column | MOP | milo-ops | contract-checker |
|--------|-----|----------|-----------------|
| document_name/title | `document_name` VARCHAR(255) | `title` TEXT | `document_name` TEXT |
| document_hash | `document_hash` VARCHAR(64) | `document_hash` TEXT | `document_hash` TEXT |
| role | `role` VARCHAR(20) | implicit in `document_type` | `role` TEXT |
| raw_text | `raw_text` TEXT | — (via `document_url`) | `raw_text` TEXT |
| analysis_result | `summary` + `issues` (separate JSONB) | `analysis_result` JSONB (combined) | `analysis_result` JSONB |
| counterparty | `publisher_id`/`buyer_id` UUID FKs | `counterparty_name` TEXT | — |
| jurisdiction | `counterparty_jurisdiction` VARCHAR | — | `counterparty_jurisdiction` TEXT |
| versioning | `archived_at` soft-delete | `contract_group_id` + `parent_version_id` + `version` | `archived_analyses` mirror table |
| signing | — | `signing_token`, `signed_at`, `signed_by`, `signature_data` | — |
| status | — | `status` (draft→signed) | — |
| risk_level | — | `risk_level` | — |

### Consolidation strategy

**MOP data (33 rows):** Already being migrated to `analysis_records` via @milo/contract-analysis migration (dry-run approved, ff7828c). This is the canonical path — entity FKs retarget to `crm_counterparties` (UUIDs preserved per Decision 32, verified today).

**milo-ops data (5-10 rows):** Lives on the same Supabase as MOP. Needs a separate migration script because:
- Different source table name (`contract_documents` vs `contract_analyses`)
- Different column names (`title` vs `document_name`, `analysis_result` vs `summary`+`issues`)
- Has `signing_token`/`signed_at` fields that map to `signing_documents`, not `analysis_records`
- `prospect_id` → need to resolve to `crm_counterparties` (prospect_pipeline is a milo-ops-specific table)

**contract-checker data:** Likely low value for migration (standalone tool, different user base). Recommend: archive the Supabase project, don't migrate. If specific analyses are needed, export JSON manually.

### Recommended approach

1. Complete MOP → `analysis_records` migration (already in flight)
2. Write `migrate-milo-ops-contracts.ts` for milo-ops → `analysis_records` + `signing_documents`
3. Skip contract-checker data migration (archive)
4. All future PPC contract work goes through @milo/* primitives

---

## 4. UI Consolidation: Which Wins?

### Three UIs compared

| Aspect | contract-checker | MOP (dashboard) | milo-ops (page) |
|--------|-----------------|-----------------|-----------------|
| Upload flow | Drag-drop + text paste (best) | Via MOP file handling | Text from DB only |
| Issue display | Cards with expand/collapse (350 LOC) | Integrated in dashboard | Severity-grouped with inline decisions |
| Per-issue AI chat | Yes (437 LOC) — unique | No | No |
| History/search | Full page (1,630 LOC) — unique | No | No |
| Settings/config | Playbook config page (515 LOC) — unique | No | No |
| Progress UX | Animated + Tagalog messages | N/A | Polling + fun facts |
| Bilingual | English + Tagalog — unique | English only | English only |
| Alex persona | No | Yes | Yes |

### Recommendation: contract-checker UI wins, with milo-ops enhancements

**Take from contract-checker:**
- Upload flow (drag-drop + text paste)
- Issue cards with expand/collapse
- Per-issue AI assistant panel
- History page with search
- Settings/playbook config page
- Tagalog translation layer
- Progress indicators

**Take from milo-ops:**
- Severity grouping with inline accept/reject/counter decisions
- Alex persona ("Alex, Senior Analyst at {company}")
- PUSH BACK / ACCEPT / REJECT recommendation UI
- Version chain display (contract_group_id browser)
- Short URL publishing UI

**Take from MOP:**
- DOCX export buttons (3 formats)
- Rider generation display

**Discard:**
- contract-checker standalone app shell (rebuild as Milo-for-PPC page)
- MOP dashboard integration (MOP is frozen/being decommissioned)
- milo-ops polling UX (use streaming instead)

---

## 5. Rule Consolidation: Dedup Map

### Rule inventory

| Source | Count | Format | ID Pattern |
|--------|-------|--------|------------|
| MOP | 73 (43 PUB_* + 26 BUY_* + 2 legacy + 6 phantom) | Structured `RuleMeta` | `PUB_FIN_FINALITY_MISSING`, `BUY_RATE_DISCOUNT_MISSING` |
| milo-ops | 60+ | Inline prompt text | Same PUB_*/BUY_* IDs as MOP |
| contract-checker | ~25 | `PlaybookRule` objects | Short slugs (`payment_terms`, `indemnification`) |

### Dedup strategy

**Step 1: MOP + milo-ops (same ID namespace)**

Both use `PUB_*/BUY_*` IDs. For each rule:
- If both have it: keep MOP's structured metadata + enhance with milo-ops's `plain_english`, `impact_to_us`, and recommendation fields
- If MOP only: keep as-is, add prompt enrichment later
- If milo-ops only: create structured `RuleMeta` entry from the prompt text

Expected outcome: ~80-85 unique rules (significant overlap, some milo-ops additions).

**Step 2: contract-checker rules → map to PUB_*/BUY_* IDs**

contract-checker uses short slugs. Create a mapping table:

| contract-checker slug | MOP/milo-ops ID | Notes |
|----------------------|-----------------|-------|
| `payment_terms` | `PUB_FIN_PAYMENT_TERMS` | Same concept |
| `indemnification` | `PUB_LGL_INDEMNITY_MISSING` | Same concept |
| `termination_clause` | `PUB_GEN_TERMINATION_ONESIDED` | Close match |
| `non_compete` | `PUB_OPS_NONCOMPETE_OVERLY_BROAD` | Close match |
| ... | ... | ~20 more mappings needed |

Expected: most contract-checker rules map to existing MOP rules. 3-5 may be unique (particularly the per-rule threshold configuration model).

**Step 3: Discard phantoms + legacy**

- 6 phantom rules (rule IDs in registry that never fire): drop
- 2 legacy rules (superseded): drop
- Net: ~80 consolidated rules

### Output format

Consolidated rules as PPC `RuleDefinition[]` plugin for @milo/contract-analysis:

```typescript
const ppcRules: RuleDefinition[] = [
  {
    rule_id: 'PUB_FIN_FINALITY_MISSING',
    title: 'Missing Payment Finality Clause',
    category: 'financial',
    defaultSeverity: 'HIGH',
    baseWeight: 20,
    plain_english: 'The contract doesn\'t guarantee when you get paid.',
    impact_to_us: 'Without finality, the buyer can dispute charges indefinitely.',
    recommendation_format: 'PUSH_BACK',
    tags: ['payment', 'publisher'],
    isExistential: false,
    riderRefs: ['§4.2 — Payment Terms'],
  },
  // ... 79 more
];
```

---

## 6. Recommended Rollout Sequence for Morgan

### Phase 1: Engine is ready (NOW)

@milo/contract-analysis, @milo/contract-signing, @milo/contract-negotiation are all shipped v0.1.0. The horizontal engine is available. Morgan can start building the PPC vertical layer today.

### Phase 2: PPC rule plugin (Week 1)

1. Create `ppc-contract-rules.ts` — consolidated `RuleDefinition[]` from MOP's 73 + milo-ops's 60+ (deduped to ~80)
2. Create `ppc-scoring-modifiers.ts` — `ScoringModifier[]` with MOP's weight system
3. Create `ppc-pattern-plugins.ts` — `PatternPlugin[]` for two-party consent, sneaked-in detection, template change detection
4. Wire into @milo/contract-analysis via `createAnalysisEngine({ rules, scoringModifiers, patternPlugins })`
5. Test: analyze a sample MSA through the engine with PPC rules

### Phase 3: PPC prompt config (Week 1-2)

1. Port milo-ops's output field instructions (`plain_english`, `impact_to_us`, recommendation format) into the PPC system prompt
2. Port Alex persona configuration
3. Port template comparison mode as a separate analysis mode
4. Port MSA improvement suggestion flow

### Phase 4: Data migration (Week 2)

1. MOP `contract_analyses` → `analysis_records` (already dry-run approved)
2. milo-ops `contract_documents` → `analysis_records` + `signing_documents`
3. Verify: all historical analyses accessible through the new engine

### Phase 5: UI consolidation (Week 2-3)

1. Build contract analysis page in Milo-for-PPC using contract-checker's UI patterns
2. Add milo-ops enhancements (severity grouping, inline decisions, version browser)
3. Port Tagalog translation, help system, settings page
4. Add DOCX export, rider generation display

### Phase 6: Feature completion (Week 3-4)

1. IO field extraction integration
2. Short URL publishing for counterparty review
3. Teams notification integration
4. Per-issue AI assistant panel
5. History page with search

### Phase 7: Decommission (Week 4+)

1. Archive contract-checker repo (standalone app no longer needed)
2. Remove contract analysis code from MOP (already frozen, eventual decommission)
3. Remove contract analysis from milo-ops (replaced by Milo-for-PPC)

---

## 7. Open Questions for Mark-Morgan Conversation

1. **AI model tier for analysis:** MOP uses Sonnet 4.5, milo-ops uses Opus 4.6. Opus produces higher-quality analysis but costs more per call. Should the PPC vertical use Opus for all analyses, or Sonnet for initial + Opus for review/negotiation?

2. **Tagalog priority:** contract-checker's Tagalog support serves the Filipino VA team. Is this day-1 for Milo-for-PPC or can it come in Phase 5?

3. **Settings UI scope:** contract-checker has per-rule threshold configuration. MOP has analysis profiles. Do operators need both (full flexibility) or just profiles (simpler)?

4. **contract-checker data:** Any analyses in the standalone contract-checker worth migrating? Or is it all test/demo data?

5. **prospect_pipeline mapping:** milo-ops's `contract_documents.prospect_id` references a milo-ops-specific table. When migrating these rows, should prospect_id be resolved to `crm_counterparties` (if the prospect was already migrated), or stored in metadata as `source_prospect_id`?

6. **Template comparison ownership:** Is template comparison a horizontal feature (add to @milo/contract-analysis v0.2.0) or a PPC-only feature? milo-ops uses it for comparing submitted MSAs against TLP's standard template — but other verticals might want the same capability.

7. **Rule consolidation owner:** Should Morgan do the MOP↔milo-ops rule dedup himself (he knows PPC nuances), or should Coder-3 produce the detailed mapping table first?

8. **IO field extraction scope:** MOP's `field-mapper.ts` extracts payout, pay terms, lead type, transfer type from IOs. Is this PPC-only, or should it become a @milo/contract-analysis `PatternPlugin`?
