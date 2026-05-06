# Synthetic Pill Response Test Suite — Results

**Date:** 2026-05-06
**Author:** Coder-3 (Research)
**Script:** `milo-for-ppc/scripts/synthetic-pill-responses.ts`
**Machine-readable results:** `milo-for-ppc/scripts/synthetic-pill-results.json`

## Summary

**108 total checks: 106 PASS / 2 FAIL**

The 2 FAILs are from the adversarial pricing fixture -- a deliberately invalid outreach draft that includes dollar amounts in copyable sections. The validator correctly detects this as an OUTREACH PRICING RULE violation. All production example JSONs from all four pills parse and validate correctly.

## Architecture Covered

### Source Files Read (read-only)
| File | Purpose |
|------|---------|
| `vendor/voice/src/pills/sales.ts` | Sales pill system prompt with 3 JSON examples |
| `vendor/voice/src/pills/buyer.ts` | Buyer pill system prompt with 3 JSON examples |
| `vendor/voice/src/pills/publisher.ts` | Publisher pill system prompt with 3 JSON examples |
| `vendor/voice/src/pills/vet.ts` | Vet pill system prompt (entity_vet + multi-entity shapes) |
| `vendor/voice/src/pills/vet-extraction.ts` | Per-entity vet prompt (extraction pipeline) |
| `vendor/voice/src/structured-response.ts` | `StructuredResponse` type, `extractStructuredResponse()`, `isStructuredResponse()` |
| `vendor/voice/src/vet-parsing.ts` | `EntityVetResult` type, `parseVetResponse()` |
| `src/components/ask/StructuredResponseCard.tsx` | Rendering component: `isPinnedSection()`, `isEntityDiscoverySection()`, `parseDiscoveredEntities()` |

### Parsing Pipeline Validated
1. Raw text -> `extractStructuredResponse()` -> JSON object
2. JSON object -> `isStructuredResponse()` -> type guard
3. StructuredResponse type field -> `TYPE_LABELS` / `TYPE_COLORS` lookup
4. research_list sections -> `isPinnedSection()` for collapse behavior
5. Entity discovery sections -> `isEntityDiscoverySection()` -> `parseDiscoveredEntities()` -> EntityCard rendering
6. Vet responses -> `isVetResult()` type guard -> EntityVetCard rendering

## Per-Pill Results

### Sales Pill (4 fixtures, all PASS)

| Fixture | Type | Checks | Result |
|---------|------|--------|--------|
| `sales_outreach_example` | outreach_draft | type guard, Subject/Hook/Body/CTA copyable, metadata, action_hint, pricing rule | PASS |
| `sales_research_example` | research_list | type guard, KEY TAKEAWAY, ACTIONABLE ITEMS, data_source on all, collapsed detail | PASS |
| `sales_general_example` | general | type guard, single Answer section | PASS |
| `sales_entity_discovery` | research_list | type guard, KEY TAKEAWAY, ACTIONABLE ITEMS, Publisher Target List -> 2 entities parsed | PASS |

### Buyer Pill (4 fixtures, all PASS)

| Fixture | Type | Checks | Result |
|---------|------|--------|--------|
| `buyer_outreach_example` | outreach_draft | type guard, Subject/Hook/Body/CTA copyable, metadata, action_hint, pricing rule | PASS |
| `buyer_research_example` | research_list | type guard, KEY TAKEAWAY (tlp_internal), ACTIONABLE ITEMS, data_source on all | PASS |
| `buyer_general_example` | general | type guard, single Answer section | PASS |
| `buyer_entity_discovery` | research_list | type guard, KEY TAKEAWAY, ACTIONABLE ITEMS, Publisher Target List -> 2 entities parsed | PASS |

### Publisher Pill (4 fixtures, all PASS)

| Fixture | Type | Checks | Result |
|---------|------|--------|--------|
| `publisher_outreach_example` | outreach_draft | type guard, Subject/Hook/Body/CTA copyable, metadata, action_hint, pricing rule | PASS |
| `publisher_research_example` | research_list | type guard, KEY TAKEAWAY, ACTIONABLE ITEMS, data_source on all | PASS |
| `publisher_general_example` | general | type guard, single Answer section | PASS |
| `publisher_entity_discovery` | research_list | type guard, KEY TAKEAWAY, ACTIONABLE ITEMS, Publisher Target List -> 2 entities parsed | PASS |

### Vet Pill (3 fixtures, all PASS)

| Fixture | Shape | Checks | Result |
|---------|-------|--------|--------|
| `vet_single_entity` | EntityVetResult | type guard, verdict, facts, flags, blacklist_match null | PASS |
| `vet_multi_entity` | MultiEntityVetResult | results array (2), per-result type guard, secondary_names (2) | PASS |
| `vet_blacklist_hit` | EntityVetResult | type guard, verdict=blacklisted, blacklist_match populated (95%) | PASS |

### Adversarial Tests (8 fixtures, all behave as expected)

| Fixture | Scenario | Result | Details |
|---------|----------|--------|---------|
| `adversarial_markdown_fenced` | JSON wrapped in \`\`\`json fences | PASS | extractStructuredResponse strips fences correctly |
| `adversarial_text_around` | Prose before and after JSON | PASS | extractStructuredResponse finds outermost {} correctly |
| `adversarial_empty_content` | Sections with empty string content | PASS | Type guard passes; renders empty sections (graceful degrade) |
| `adversarial_missing_metadata` | No metadata, no action_hint | PASS | Type guard passes; metadata grid and hint row do not render |
| `adversarial_h2_entity_discovery` | `##` instead of `###` in entity list | PASS | parseDiscoveredEntities returns null -- falls back to markdown rendering |
| `adversarial_special_chars` | Entity names with & < > " ' | PASS | Parsed 2 entities; & preserved; < absorbed by markdown parser |
| `adversarial_unknown_type` | type="campaign_analysis" | PASS | Type guard passes (only checks string); falls back to "RESPONSE" label |
| `adversarial_pricing_in_outreach` | Dollar amounts in copyable sections | EXPECTED FAIL | Correctly caught 2 pricing violations (Hook, CTA) |
| `adversarial_buyer_target_list` | "Buyer Target List" label | PASS | isEntityDiscoverySection detects "TARGET LIST" substring; entities parse correctly |

## Findings

### Finding 1: All prompt example JSONs are valid
Every JSON example embedded in the sales, buyer, and publisher pill prompts parses correctly through the full StructuredResponseCard pipeline. No structural mismatches between what the prompts teach Claude to output and what the rendering code expects.

### Finding 2: Vet pill uses a completely separate type system
The vet pill does NOT output StructuredResponse JSON -- it outputs EntityVetResult (single) or MultiEntityVetResult (multi). This is handled by a separate EntityVetCard component, not StructuredResponseCard. The test validates both shapes. The per-entity extraction prompt (`buildPerEntityPrompt`) includes `associated_entities` field not present in the main VET_PROMPT, but both share the core verdict/entity/facts/flags shape.

### Finding 3: extractStructuredResponse handles common Claude formatting errors
The JSON extractor correctly handles:
- Markdown code fences (\`\`\`json ... \`\`\`) -- strips them
- Prose before/after JSON -- finds outermost { }
- Pure JSON (no wrapping) -- passes through

It does NOT handle:
- Multiple JSON objects in one response (takes outermost braces, which may span both)
- Truncated JSON (missing closing brace) -- returns null

### Finding 4: isEntityDiscoverySection uses substring match
The function checks for "TARGET LIST" substring (case-insensitive). This means:
- "Publisher Target List" -> detected
- "Buyer Target List" -> detected
- "Target List" -> detected
- "PUBLISHER TARGET LIST" -> detected

This is more permissive than the prompt instructions (which specify exact labels), but this is correct defensive behavior -- Claude may vary casing.

### Finding 5: parseDiscoveredEntities only matches `### N.` format
The regex `/^### (\d+)\.\s+(.+?)$/m` requires exactly three hash marks. Claude using `##` or `####` will NOT trigger entity card rendering -- it falls back to markdown. This is correct behavior (the prompt specifies `###`), but if Claude deviates, entities render as plain markdown instead of interactive cards.

### Finding 6: Empty content sections pass validation
A research_list with empty KEY TAKEAWAY and ACTIONABLE ITEMS content strings passes the type guard and renders without error. The UI shows the section headers with empty content below them. This is a graceful degradation but represents a low-quality response from Claude.

### Finding 7: The pricing rule detector works on raw string matching
The `$\d` regex check catches dollar amounts in copyable outreach sections. It correctly flags `$45 CPL` and `$45/180` patterns. The Rationale section (non-copyable) is not checked, which is correct -- rationale is internal-facing.

### Finding 8: Unknown response types get fallback rendering
A response with type "campaign_analysis" passes `isStructuredResponse()` (which only checks for string type + array sections). The card renders with the generic "RESPONSE" label and white color scheme. No crash, but the type badge may confuse operators. In practice, the pill prompts constrain Claude to known types, so this is an edge case safety net.

### Finding 9: Buyer Target List is correctly detected alongside Publisher Target List
The `isEntityDiscoverySection` function's substring match on "TARGET LIST" means both buyer-side and publisher-side entity discovery labels are detected. Entities parse identically regardless of whether the list is buyer or publisher focused.

### Finding 10: No cross-contamination between vet and StructuredResponse pipelines
The vet pill outputs a fundamentally different JSON shape (verdict/entity/facts/flags vs. type/sections/metadata). The ask page routes these through separate code paths -- VetResultData for vet, StructuredResponse for sales/buyer/publisher. There is no risk of a vet result being accidentally parsed as a StructuredResponse or vice versa, because `isStructuredResponse()` checks for `.type` + `.sections` which vet results lack, and vet parsing checks for `.verdict` + `.entity` which StructuredResponses lack.

## Recommendations

1. **No immediate action needed** -- all prompt examples produce valid rendering output.
2. **Consider adding `data_source` to general type examples** -- currently the general examples in all three pills omit `data_source`, which is only required for research_list. This is correct per spec.
3. **Consider adding `schema_version` field to prompt examples** -- the StructuredResponse interface supports it, the card displays it, but no pill prompt includes it in examples. Adding `"schema_version": 1` would future-proof versioned parsing.
4. **Entity name XSS is handled by React** -- special characters (& < > " ') in entity names are escaped by React's default rendering. The `parseDiscoveredEntities` function preserves them as-is, which is correct.

---

Report at: https://raw.githubusercontent.com/moscaleflow/conductor-mark-public/main/reports/pill-response-test-2026-05-06.md
