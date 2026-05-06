# Data Integrity Audit: CRM + Pipeline Dedup Gaps

**Date:** 2026-05-06
**Author:** Coder-3 (Research)
**Repo:** milo-for-ppc
**Scope:** All code paths that CREATE or MODIFY `crm_counterparties`, `prospect_pipeline`, `verification_tokens`, and `vet_results`

---

## Executive Summary

The system has **11 distinct code paths** that create `prospect_pipeline` entries and **7 distinct code paths** that create `crm_counterparties` entries. Of these, **5 pipeline creation paths have NO dedup check** and **3 CRM creation paths have weak or missing dedup**. The core problem: multiple subsystems independently create records for the same entity without coordinating through a single dedup gateway.

The `findOrCreateCounterparty` function in `crm-client.ts` IS a dedup gateway for CRM counterparties, but not all paths use it, and its dedup is case-insensitive name match only (no fuzzy, no alias awareness). Pipeline entries have NO equivalent gateway -- each code path implements its own (or zero) dedup logic.

---

## 1. CRM Counterparty Creation Paths

### Path 1: `findOrCreateCounterparty()` (shared utility)
- **File:** `src/lib/crm-client.ts:49-90`
- **Dedup:** YES -- `ilike('company', companyName)` on `crm_counterparties`
- **Callers:**
  - `/api/ask/entity-add` (line 34)
  - `/api/ask/vet-action` handleAddCrm (line 94)
  - `/api/capture` via `findOrCreateCrmContact` (line 496)
  - `/api/leads/import` via `findOrCreateCrmContact` (line 351)
  - `tools/setup-new-publisher` via `findOrCreateCrmContact` (line 88)
  - `tools/setup-new-buyer` via `findOrCreateCrmContact` (line 80)

**GAP 1 -- ILIKE-only dedup:** The dedup query uses `ilike('company', companyName)` which is exact case-insensitive match. "AP Affiliates" will NOT match "AP Affiliates LLC" or "AP Affiliates, Inc." No fuzzy/containment logic. The discovery cron has fuzzy match but the CRM client does not.

### Path 2: `findOrCreateCrmContact()` (shared utility)
- **File:** `src/lib/crm-client.ts:92-143`
- **Dedup on contact:** YES -- `eq('email', opts.email)` on `crm_contacts` (only if email provided)
- **Dedup on counterparty:** Delegates to `findOrCreateCounterparty` (Path 1)

**GAP 2 -- No email means no contact dedup:** If `opts.email` is null/undefined, the function ALWAYS creates a new `crm_contacts` row. Two captures of the same person with name but no email = two contact rows. The function does not check by name.

### Path 3: `/api/capture` -- `updatePipelineAndCreateContacts()`
- **File:** `src/app/api/capture/route.ts:477-504`
- **Dedup on counterparty:** YES -- `ilike('company', company)` count check (line 480)
- **Creates via:** `findOrCreateCrmContact` (Path 2)
- **Correctly gates:** Only creates if `crmCount === 0`

### Path 4: `/api/outreach/record-manual-send` (UPDATE only)
- **File:** `src/app/api/outreach/record-manual-send/route.ts:77-79`
- **Creates:** NO -- only UPDATES `crm_counterparties.last_contact_at` via `ilike('company', entity_name)`
- Not a creation risk.

---

## 2. Pipeline Entry Creation Paths

### Path A: `/api/ask/vet-action` -- `handleAddCrm()`
- **File:** `src/app/api/ask/vet-action/route.ts:97-108`
- **Dedup:** NONE
- **Creates:** `prospect_pipeline.insert({ entity_name, entity_type, stage: opts.stage ?? 'vetted', source: 'milo-vet' })`
- **CRITICAL GAP 3:** No check for existing pipeline entry. If an operator clicks "Add to CRM" twice on the same vet card, or vets the same entity twice and adds both times, duplicate pipeline rows are created. The CRM counterparty won't duplicate (findOrCreateCounterparty handles that), but the pipeline entry WILL.

### Path B: `/api/cron/publisher-discovery` -- `createDiscoveryEntry()`
- **File:** `src/app/api/cron/publisher-discovery/route.ts:289-301`
- **Dedup:** YES (pre-check) -- `getExistingEntityNames()` (line 130-146) loads ALL names from both `crm_counterparties` AND `prospect_pipeline`, then `fuzzyMatch()` checks containment. Intra-batch dedup via `knownNames.add()` (line 539).
- **Creates:** `prospect_pipeline.insert({ entity_name, entity_type, stage: 'qualifying', source: 'milo_discovery' })`
- **This is the BEST dedup in the system.** However:
- **GAP 4 -- Timing race:** If two cron runs overlap (unlikely but possible with manual trigger + scheduled), the in-memory `knownNames` set won't see the other run's inserts.

### Path C: `/api/capture` -- `updatePipelineAndCreateContacts()`
- **File:** `src/app/api/capture/route.ts:507-510`
- **Dedup:** YES -- `ilike('entity_name', company)` on `prospect_pipeline` (line 488)
- **Creates:** Only if `!existingPipeline || existingPipeline.length === 0`
- **Correctly gates pipeline creation.** If a pipeline entry exists, links contact instead.

### Path D: `/api/publisher-onboarding` -- Path A (add_to_pipeline)
- **File:** `src/app/api/publisher-onboarding/route.ts:66-78`
- **Dedup:** YES -- `ilike('entity_name', businessName)` check, returns 409 if exists
- **Creates:** `prospect_pipeline.insert({ entity_type: 'publisher', entity_name, stage: 'qualifying' })`
- **Good dedup, returns error on duplicate.**

### Path E: `lib/publisher-onboarding.ts` -- `triggerPublisherOnboarding()` Path B (trusted shortcut)
- **File:** `src/lib/publisher-onboarding.ts:163-201`
- **Dedup:** NONE
- **Creates:** `prospect_pipeline.insert({ entity_type: 'publisher', entity_name, stage: 'onboarding' })`
- **GAP 5:** When `pipeline_entity_id` is NOT provided (trusted shortcut), the function creates a new pipeline row without checking if one already exists for the same company name. This is called by `tools/setup-new-publisher.ts`.

### Path F: `lib/publisher-onboarding.ts` -- `handleInboundPublisherSubmission()`
- **File:** `src/lib/publisher-onboarding.ts:319-325`
- **Dedup:** YES -- `ilike('entity_name', company_name)` check
- **Creates:** Only if no existing match. If exists and in early stage, advances to onboarding.
- **Good dedup + stage advancement logic.**

### Path G: `/api/onboard/publisher` (POST)
- **File:** `src/app/api/onboard/publisher/route.ts:25`
- **Dedup:** Delegates to `handleInboundPublisherSubmission()` (Path F) -- has dedup
- **Correctly gated.**

### Path H: `tools/setup-new-buyer` -- `execute()`
- **File:** `src/lib/tools/setup-new-buyer.ts:229-239`
- **Dedup:** NONE
- **Creates:** `prospect_pipeline.insert({ entity_type: 'buyer', entity_name, stage: 'onboarding'|'qualifying' })`
- **GAP 6:** No check for existing pipeline entry. Running `setup_new_buyer` twice for the same buyer creates two pipeline rows.

### Path I: `tools/setup-new-publisher` -- `execute()`
- **File:** `src/lib/tools/setup-new-publisher.ts:235-248`
- **Dedup:** Delegates to `triggerPublisherOnboarding()` (Path E)
- **Inherits GAP 5** -- no dedup on trusted shortcut path.

### Path J: `/api/leads/import` (CSV bulk import)
- **File:** `src/app/api/leads/import/route.ts:303-313`
- **Dedup:** YES -- checks both `contacts.company_name` and `prospect_pipeline.entity_name` via `ilike`
- **Creates:** Only if not duplicate
- **Good dedup, but uses exact ilike match (no fuzzy).**

### Path K: `/api/verify` PATCH handler (stage update, not creation)
- **File:** `src/app/api/verify/route.ts:311-317`
- **Creates:** NO -- only UPDATES existing pipeline entry stage to 'outreach'
- **Also creates contacts** (line 245) but with email-based dedup
- Not a creation risk for pipeline entries.

---

## 3. Verification Token Creation Paths

### Path 1: `/api/cron/publisher-discovery` -- `createDiscoveryEntry()`
- **File:** `src/app/api/cron/publisher-discovery/route.ts:310-319`
- **Dedup:** NONE -- always creates a new token per discovered entity
- **Expected behavior** -- tokens are meant to be unique per discovery event

### Path 2: `/api/verify` POST handler
- **File:** `src/app/api/verify/route.ts:117-126`
- **Dedup:** NONE -- always creates a new token
- **Expected behavior** -- manual token creation should always produce a fresh token

**No dedup gaps here.** Verification tokens are intentionally one-per-event.

---

## 4. Vet Results Creation Paths

### Path 1: `saveVetResult()` (shared utility)
- **File:** `src/lib/ask/vet-results.ts:48-74`
- **Dedup:** NONE -- always inserts
- **Callers:**
  - `/api/ask/entity-vet-inline/route.ts:86` (inline vet)
  - `/api/ask/route.ts` (main ask flow, via tool execution)

### Path 2: `/api/ask/entity-vet-inline` (inline vet)
- **File:** `src/app/api/ask/entity-vet-inline/route.ts:44-63`
- **Cache check:** YES -- `lookupExistingVet(normalized, 'tlp')` + `isVetFresh()` before creating new
- **Creates:** Only if no fresh cached result exists

**GAP 7 -- Race condition on parallel vets:** If two requests vet the same entity simultaneously, both may pass the "is fresh?" check before either inserts, resulting in duplicate vet_results rows. Not a high-frequency risk but possible with rapid UI clicks.

**GAP 8 -- No dedup on saveVetResult itself:** The `saveVetResult` function always inserts. The dedup is done by callers (inline vet checks cache; the main ask flow does not appear to check). Multiple `/ask` conversations about the same entity will create multiple vet_results rows.

---

## 5. Stage Transition Map

### Valid Stages (from `/api/pipeline/[id]/stage/route.ts:5-15`)
```
vetted -> outreach -> qualifying -> drip -> onboarding -> activation -> active -> dormant -> blacklisted
```

### How Entities Enter Each Stage

| Stage | Entry Path | File |
|-------|-----------|------|
| `qualifying` | Discovery cron | `publisher-discovery/route.ts:295` |
| `qualifying` | Capture (new entity) | `capture/route.ts:509` |
| `qualifying` | Publisher onboarding add_to_pipeline | `publisher-onboarding/route.ts:118` |
| `qualifying` | Setup new buyer (io_signed=false) | `setup-new-buyer.ts:237` |
| `vetted` | Vet-action add_crm (default) | `vet-action/route.ts:104` |
| `outreach` | Verify PATCH completion | `verify/route.ts:311-317` |
| `outreach` | Leads CSV import | `leads/import/route.ts:384` |
| `onboarding` | triggerPublisherOnboarding | `publisher-onboarding.ts:148,183` |
| `onboarding` | handleInboundPublisherSubmission (stage advance) | `publisher-onboarding.ts:338` |
| `onboarding` | Setup new buyer (io_signed=true) | `setup-new-buyer.ts:236` |
| `onboarding` | Setup new publisher | `setup-new-publisher.ts:235` |

### Stage Transitions

| From | To | Mechanism | File |
|------|----|-----------|------|
| `unverified`/`vetted` | `outreach` | Verify PATCH (auto) | `verify/route.ts:311-317` |
| `outreach`/`qualifying`/`drip` | `onboarding` | Inbound submission (auto) | `publisher-onboarding.ts:335-340` |
| Any existing | `onboarding` | triggerPublisherOnboarding Path A | `publisher-onboarding.ts:144-152` |
| Any | Any valid | Manual stage change | `pipeline/[id]/stage/route.ts:213-223` |

**GAP 9 -- Stage "unverified" used but not in VALID_STAGES:** The verify PATCH handler (line 317) moves entities from stages `['unverified', 'vetted']` to `outreach`, but `unverified` is NOT in the VALID_STAGES array in `pipeline/[id]/stage/route.ts`. This means:
- Entities can be placed at `unverified` via direct DB insert
- The manual stage-change endpoint cannot move entities TO `unverified` (it's not in the allowed list)
- But the verify handler can move entities FROM `unverified` (it checks via `.in('stage', ['unverified', 'vetted'])`)

**GAP 10 -- "trusted_shortcut" stage set but overwritten:** In `publisher-onboarding.ts:165`, the code sets `stage = 'trusted_shortcut'` locally but then inserts with `stage: 'onboarding'` (line 183). The `trusted_shortcut` value is only returned in the response object, never persisted. Misleading but not a data issue.

---

## 6. Complete Gap Summary and Recommended Fixes

### GAP 1: CRM counterparty dedup is ILIKE-exact only
- **Risk:** HIGH -- "Acme LLC" and "Acme" create two counterparties
- **Fix:** Add containment matching to `findOrCreateCounterparty()` similar to the discovery cron's `fuzzyMatch()`. Query with ILIKE `%companyName%` and also check if any existing company contains the input or vice versa. Consider normalizing company names (strip LLC, Inc, Corp suffixes) before comparison.

### GAP 2: CRM contact dedup requires email
- **Risk:** MEDIUM -- Contacts without email are always created fresh
- **Fix:** Add name + counterparty_id compound lookup as fallback when email is null. If a contact already exists for the same counterparty with the same name (case-insensitive), return it instead of creating a duplicate.

### GAP 3: vet-action `handleAddCrm()` has NO pipeline dedup
- **Risk:** CRITICAL -- Double-clicking "Add to CRM" creates duplicate pipeline rows
- **Fix:** Before inserting into `prospect_pipeline`, check `ilike('entity_name', opts.entityName)` and return existing row if found. This is a one-line fix that mirrors what capture and publisher-onboarding already do.

### GAP 4: Discovery cron timing race
- **Risk:** LOW -- Requires overlapping manual + scheduled runs
- **Fix:** Use a Supabase unique constraint on `(entity_name, entity_type)` or use `upsert` with `onConflict`. Alternatively, the day-gate already prevents same-day re-runs.

### GAP 5: `triggerPublisherOnboarding()` trusted shortcut has no pipeline dedup
- **Risk:** HIGH -- `setup_new_publisher` and direct API calls create duplicates
- **Fix:** Add `ilike('entity_name', input.business_name)` check before insert, same pattern as `handleInboundPublisherSubmission()` which already does this correctly.

### GAP 6: `setup_new_buyer` has no pipeline dedup
- **Risk:** HIGH -- Running the tool twice creates two pipeline rows
- **Fix:** Add dedup check before `prospect_pipeline.insert()` at line 229. Check `ilike('entity_name', input.buyer_name)`. If exists, use existing row ID and skip insert.

### GAP 7: Race condition on parallel inline vets
- **Risk:** LOW -- Requires concurrent requests for same entity
- **Fix:** Use `entity_name_normalized` as a unique constraint in the DB, or use `upsert` pattern. Alternatively, accept multiple vet_results as intentional (re-vet history).

### GAP 8: saveVetResult has no dedup
- **Risk:** MEDIUM -- Multiple /ask conversations create multiple vet rows for same entity
- **Fix:** This may be intentional (vet results represent point-in-time research). If not, add `upsert` on `(tenant_id, entity_name_normalized)`.

### GAP 9: "unverified" stage missing from VALID_STAGES
- **Risk:** LOW -- Inconsistency between verify handler and stage-change endpoint
- **Fix:** Add `'unverified'` to the VALID_STAGES array in `pipeline/[id]/stage/route.ts`, or remove the `unverified` check from the verify handler if it's never used as an entry stage.

### GAP 10: "trusted_shortcut" stage variable misleading
- **Risk:** NONE (cosmetic)
- **Fix:** Remove the `stage = 'trusted_shortcut'` assignment since it's never persisted.

---

## 7. Recommended Priority Order

1. **GAP 3** (vet-action pipeline dedup) -- CRITICAL, one-line fix
2. **GAP 5** (triggerPublisherOnboarding dedup) -- HIGH, mirrors existing pattern
3. **GAP 6** (setup_new_buyer dedup) -- HIGH, mirrors existing pattern
4. **GAP 1** (CRM counterparty fuzzy match) -- HIGH, prevents name-variant duplicates
5. **GAP 2** (CRM contact name-based fallback) -- MEDIUM
6. **GAP 9** (unverified stage consistency) -- LOW
7. **GAP 8** (vet_results dedup decision) -- MEDIUM, needs product decision on whether re-vets are intentional
8. **GAP 7** (parallel vet race) -- LOW
9. **GAP 4** (discovery cron race) -- LOW

---

## 8. Recommended Architectural Fix

All 11 pipeline creation paths should funnel through a single `findOrCreatePipelineEntry()` utility, similar to how CRM counterparties funnel through `findOrCreateCounterparty()`. This function would:

1. Normalize the entity name (strip suffixes, lowercase)
2. Check for existing entry by normalized name (fuzzy)
3. If found: return existing ID, optionally update stage if advancing
4. If not found: insert new row, return ID
5. Handle the `entity_type` mismatch case (same company is both publisher and buyer)

This eliminates the need to add dedup checks to each individual code path and prevents future paths from accidentally skipping dedup.

---

Report at: https://raw.githubusercontent.com/moscaleflow/conductor-mark-public/main/reports/data-integrity-audit-2026-05-06.md
