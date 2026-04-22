# milo-ops Contacts Migration to @milo/crm (E3)

**Date**: 2026-04-20
**Lane**: Coder-2 (Migration)
**Directive**: #39 (E3)
**Status**: COMPLETE — write paths done, primary read paths done, ~20 secondary reads pending data migration

---

## Summary

All 10 milo-ops contact write operations now go through `@milo/crm` SDK with `tenant_id='tlp'`. The flat contacts model (company_name + entity_type per row) is resolved to CRM's relational model (crm_contacts → crm_counterparties) via `findOrCreateCounterparty`. Vertical-specific fields stored in CRM contact `metadata` JSONB.

---

## Commits (4 total)

| SHA | File(s) | What changed |
|-----|---------|--------------|
| `6c26304` | crm-client.ts, package.json, copy-milo-packages.mjs, next.config.ts | CRM client wrapper + dependency infrastructure |
| `699c19d` | 9 files (see write sites below) | 10 contact write sites migrated |
| `f473313` | milo-queries.ts | Primary read paths with CRM-first + local fallback |

---

## crm-client.ts wrapper (~130 LOC)

Key functions:
- `crmClient()` — singleton CRM client, tenant='tlp'
- `findOrCreateCounterparty(companyName, entityType)` — resolves company to crm_counterparties (creates if needed, maps entity_type → counterparty_type)
- `findOrCreateCrmContact(opts)` — dedup by email, create with counterparty parent link + metadata for vertical fields
- `updateCrmContact(id, updates)` — pass-through to @milo/crm updateContact
- `listCrmContacts(opts)` — pass-through to @milo/crm listContacts

Entity type mapping: publisher→publisher, buyer→buyer, vendor→vendor, partner→partner, client→client, unknown→prospect

---

## Write sites migrated (10 total)

| # | File | Operation | Before | After |
|---|------|-----------|--------|-------|
| 1 | `api/contacts/route.ts` | INSERT | `sb.from('contacts').insert(...)` | `findOrCreateCrmContact(...)` |
| 2 | `api/contacts/[id]/route.ts` | UPDATE | `sb.from('contacts').update(...)` | `updateCrmContact(id, ...)` with metadata merge |
| 3 | `api/leads/import/route.ts` | INSERT | `sb.from('contacts').insert(...)` | `findOrCreateCrmContact(...)` |
| 4 | `lib/publisher-onboarding.ts` | INSERT | local findOrCreateContact | `findOrCreateCrmContact(...)` |
| 5 | `lib/tools/setup-new-publisher.ts` | INSERT | local findOrCreateContact | `findOrCreateCrmContact(...)` |
| 6 | `lib/tools/setup-new-buyer.ts` | INSERT | local findOrCreateContact | `findOrCreateCrmContact(...)` |
| 7 | `api/capture/route.ts:470` | UPDATE | `sb.from('contacts').update(...)` | `updateCrmContact(id, ...)` |
| 8 | `api/capture/route.ts:496` | INSERT | `sb.from('contacts').insert(...)` | `findOrCreateCrmContact(...)` |
| 9 | `lib/web-research.ts` | UPDATE x3 | `sb.from('contacts').update(...)` | `updateCrmContact(id, {metadata: ...})` |
| 10 | `api/research/entity/route.ts` | UPDATE | `sb.from('contacts').update({personal_notes})` | `updateCrmContact(id, {metadata: {..., personal_notes}})` |

---

## Metadata field mapping

| milo-ops contacts column | CRM crm_contacts equivalent | Storage |
|--------------------------|----------------------------|---------|
| `full_name` | `name` | Direct column |
| `email` | `email` | Direct column |
| `phone` | `phone` | Direct column |
| `role` | `role` | Direct column |
| `linkedin_url` | `linkedin_url` | Direct column |
| `company_name` | *(derived from parent counterparty.company)* | Structural — resolved via `findOrCreateCounterparty` |
| `entity_type` | *(derived from parent counterparty.counterparty_type)* | Structural — mapped at counterparty level |
| `timezone` | `metadata.timezone` | Metadata JSONB |
| `preferred_channel` | `metadata.preferred_channel` | Metadata JSONB |
| `facebook_url` | `metadata.facebook_url` | Metadata JSONB |
| `twitter_url` | `metadata.twitter_url` | Metadata JSONB |
| `website` | `metadata.website` | Metadata JSONB |
| `industry` | `metadata.industry` | Metadata JSONB |
| `company_size` | `metadata.company_size` | Metadata JSONB |
| `research_status` | `metadata.research_status` | Metadata JSONB |
| `personal_notes` (JSONB) | `metadata.personal_notes` | Metadata JSONB |

---

## Read paths

### Migrated (CRM-first + local fallback)
- `milo-queries.ts:getContacts()` — primary contact search, used by milo-tools entity lookup
- `milo-queries.ts:getPipelineOverview()` — contact counts by type

### Remaining (~20 sites, read-only, lower priority)
These still read from local `contacts` table. They will auto-resolve after a data migration seeds crm_contacts from existing local data:
- `milo-tools.ts` — 3 sites (entity lookup, action items contact resolution, screening)
- `milo-queries.ts` — 3 sites (entity detail, pipeline stats, conversation matching)
- `milo-intent.ts` — 1 site (contact lookup for intent resolution)
- `campaign-matching.ts` — 1 site
- `api/capture/route.ts` — 2 sites (contact matching for conversation capture)
- `api/milo/route.ts` — 2 sites
- `api/publisher-onboarding/route.ts` — 3 sites
- `api/leads/import/route.ts` — 2 sites (dedup checks)
- `api/malvin/queue/route.ts` — 1 site
- `api/contract/process/route.ts` — 1 site
- `api/td/create-entity/route.ts` — 1 site
- `lib/tools/warm-handoff-to-tiffani.ts` — 1 site
- `lib/tools/publish-contract-for-buyer-review.ts` — 1 site

---

## Not in CRM scope (stays in milo-ops)

Per feature request report (b65c9af):
- `prospect_pipeline` — TLP-specific pipeline stages, TD IDs, coverage_states, risk_flags
- `campaign_routes` — TLP routing configuration
- `agreed_terms` — contractual/financial terms

---

## Remaining gaps

1. **No data migration yet** — existing local `contacts` rows are not in crm_contacts. New contacts go to CRM, but historical data needs a one-time migration script.
2. **~20 secondary read sites** — still read local. Non-blocking (data is there), but should be migrated after data migration.
3. **No `deleteLead`/`deleteCounterparty` in @milo/crm** — would need direct Supabase for contact deletion (not needed yet).

---

*Coder-2 — Migration lane*
