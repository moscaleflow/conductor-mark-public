# Coder-2 Report: milo-ops @milo/blacklist Consumer Integration

**Date**: 2026-04-20
**Lane**: Coder-2 (Migration)
**Task**: Integrate @milo/blacklist into milo-ops counterparty/lead flows
**Status**: Coverage gaps fixed (3 commits). Package migration blocked on feature parity.

---

## Key Finding: Local Implementation is a Superset

milo-ops already has `src/lib/blacklist-screening.ts` (237 LOC) — a **richer** screening engine than `@milo/blacklist`. Comparison:

| Capability | Local `screenEntity()` | `@milo/blacklist` `check()` |
|---|---|---|
| exact_name | Yes | Yes |
| fuzzy_name (normalized) | Yes | Yes |
| fuzzy_name (Levenshtein) | Yes (distance < 3) | No |
| alias | Yes | Yes |
| contact_name | Yes (with Levenshtein) | Yes |
| email_domain | Yes | Yes |
| ip_address | Yes | **No** |
| linkedin_url | Yes | **No** |
| contact_names[] input | Yes | **No** (only company_name + email) |
| Tenant isolation | No (cross-tenant) | Yes |
| Pure function for testing | No | Yes (`screenAgainst`) |

**Migration would regress**: Loses ip_address, linkedin_url signals, contact_names multi-input, and Levenshtein fuzzy matching on contact names.

## Existing Coverage (already integrated)

| Flow | File | Screening |
|---|---|---|
| CSV lead import | `src/app/api/leads/import/route.ts` | `screenEntity()` at line 263 |
| Conversation capture | `src/app/api/capture/route.ts` | `screenEntity()` at line 319 |
| Publisher onboarding (trigger) | `src/lib/publisher-onboarding.ts` | `screenEntity()` at line 106 |
| Prospect research tool | `src/lib/tools/research-prospect.ts` | Via `/api/blacklist/screen` |
| Blacklist screen API | `src/app/api/blacklist/screen/route.ts` | Direct `screenEntity()` |

## Coverage Gaps Fixed (3 commits)

### Gap 1: Buyer setup tool — no screening

**SHA**: `50516b7` — feat: add blacklist screening gate to buyer setup tool

- File: `src/lib/tools/setup-new-buyer.ts`
- Added `screenEntity()` call as step 1, before IO check and pipeline creation
- If blocked: logs `buyer_setup_blocked` to `ai_action_log`, throws error
- Renumbered steps 1-8 → 1-9

### Gap 2: Publisher add_to_pipeline mode — no screening

**SHA**: `fdefdf0` — feat: add blacklist screening to add_to_pipeline mode

- File: `src/app/api/publisher-onboarding/route.ts`
- Added `screenEntity()` after input validation, before duplicate check
- If blocked: logs `pipeline_add_blocked` to `ai_action_log`, returns 403

### Gap 3: Inbound MOP webhook — no screening

**SHA**: `16566cd` — feat: add blacklist screening to inbound MOP webhook handler

- File: `src/lib/publisher-onboarding.ts` (`handleInboundMOPSubmission`)
- Added `screenEntity()` after company_name validation, before pipeline creation
- If blocked: logs `inbound_mop_blocked` to `ai_action_log`, returns error

## Build verification

`npm run build` passes clean after all 3 commits.

## Flows explicitly left alone

| Flow | File | Reason |
|---|---|---|
| Outreach drafting | `src/app/api/outreach/draft/route.ts` | Downstream of research_prospect which screens |
| First outreach writer | `src/lib/tools/draft-first-outreach.ts` | Downstream of research_prospect |
| Contact creation | `src/app/api/contacts/route.ts` | Low-level CRUD; screening belongs upstream |
| Entity verification | `src/lib/entity-verification.ts` | Disambiguation tool, not a gate point |
| Research entity | `src/app/api/research/entity/route.ts` | Enrichment, not creation |

## @milo/blacklist Migration — Blocked

**Status**: Blocked on feature parity. Three feature requests for Coder-1:

1. **Add `ip_address` signal** — milo-ops screens IP addresses against blacklist entries. `@milo/blacklist` does not support this.
2. **Add `linkedin_url` signal** — milo-ops screens LinkedIn profile URLs. `@milo/blacklist` does not support this.
3. **Add `contact_names[]` input** — milo-ops passes multiple contact names for screening. `@milo/blacklist` `check()` only accepts `company_name` + `email`.

Optional (architectural decision for Mark):
4. **Cross-tenant vs tenant-isolated screening** — Local impl queries ALL blacklist entries regardless of tenant. `@milo/blacklist` filters by tenant. If a scammer is blacklisted by mysecretary but not by TLP, should TLP's screening catch them? Current behavior: yes. After migration: no.

**Recommended path**: Once Coder-1 adds features 1-3 to `@milo/blacklist`, migrate `blacklist-screening.ts` to use the package. Until then, the local implementation provides strictly more coverage.

## Files changed (milo-ops)

| File | Change |
|---|---|
| `src/lib/tools/setup-new-buyer.ts` | Added screening gate (step 1) |
| `src/app/api/publisher-onboarding/route.ts` | Added screening to add_to_pipeline mode |
| `src/lib/publisher-onboarding.ts` | Added screening to handleInboundMOPSubmission |

---

*Coder-2 — Migration lane*
