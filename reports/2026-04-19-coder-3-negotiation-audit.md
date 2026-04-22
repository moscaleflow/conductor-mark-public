---
directive: "@milo/contract-negotiation pre-extraction audit"
lane: research
coder: Coder-3
started: 2026-04-19 ~22:00 MDT
completed: 2026-04-19 ~22:45 MDT
---

# @milo/contract-negotiation Pre-Extraction Audit — 2026-04-19

## 1. Executive Summary

MOP's contract negotiation system is ~4,127 LOC spanning 14 API routes, a state machine module (defined but unused), a generic text merge engine, webhook dispatch, 3 UI pages, and 3 database tables. The system is **overwhelmingly horizontal** — the negotiation lifecycle, round management, tokenized review links, and merge algorithm are all domain-agnostic. Vertical coupling is limited to hardcoded brand strings ("TLP Compliance", "The Lead Penguin", `#dd72a6`). Critical finding: **~1,000 LOC is duplicated** between internal and bot download routes, and the **state machine is dead code** — routes implement ad-hoc guards instead of using `isTransitionAllowed()`. Estimated `@milo/contract-negotiation` v0.1.0: ~1,800 LOC.

## 2. API Route Inventory

### Internal Routes (`app/api/negotiations/`)

| Route | Method | LOC | Purpose | H/V |
|-------|--------|-----|---------|-----|
| `/create` | POST | 91 | Create negotiation + Round 1 with broker decisions | Mixed (hardcodes "TLP Compliance") |
| `/[id]/finalize` | POST | 82 | Status -> finalized, fires webhook | **Horizontal** |
| `/[id]/cancel` | POST | 81 | Status -> cancelled, fires webhook | **Horizontal** |
| `/[id]/generate-link` | POST | 104 | Create tokenized review link + short code | Mixed (hardcodes `tlpmop.netlify.app`) |
| `/[id]/send-round` | POST | 149 | New broker round + generate link | Mixed (hardcodes "TLP Compliance", `tlpmop.netlify.app`) |
| `/[id]/download` | GET | 503 | Generate merged agreement as DOCX/HTML | Mixed (hardcodes TLP branding in output) |
| **Subtotal** | | **1,010** | | |

### Bot Routes (`app/api/bot/negotiations/`)

| Route | Method | LOC | Purpose | H/V |
|-------|--------|-----|---------|-----|
| `/create` | POST | 106 | Same as internal + auth + webhook_url + fires webhook | Mixed |
| `/[id]` | GET | 97 | Full negotiation state (no internal equivalent) | **Horizontal** |
| `/[id]/finalize` | POST | 78 | Same as internal + auth | **Horizontal** |
| `/[id]/generate-link` | POST | 96 | Same but different token gen, no short_code | Mixed |
| `/[id]/download` | GET | 499 | Nearly identical to internal download | Mixed |
| **Subtotal** | | **876** | | |

### Review Routes (public, token-authenticated)

| Route | Method | LOC | Purpose | H/V |
|-------|--------|-----|---------|-----|
| `/api/review/[token]` | GET | 123 | Load negotiation for counterparty review | **Horizontal** |
| `/api/review/[token]/submit` | POST | 144 | Submit counterparty round, fire webhook | **Horizontal** |
| `/r/[code]` | GET | 33 | Short-link redirect to /review/[token] | **Horizontal** |
| **Subtotal** | | **300** | | |

## 3. Duplication Analysis: Internal vs Bot

| Feature | Internal LOC | Bot LOC | Overlap | Notes |
|---------|-------------|---------|---------|-------|
| Create | 91 | 106 | ~70% | Bot adds auth + webhook_url + webhook fire |
| Finalize | 82 | 78 | ~85% | Bot uses chained `.select().single()` |
| Generate-link | 104 | 96 | ~60% | Different token gen, no short_code in bot |
| Download | 503 | 499 | ~90% | Bot lacks issue title lookup, simpler outcome logic |
| Cancel | 81 | N/A | — | No bot equivalent |
| Send-round | 149 | N/A | — | No bot equivalent |
| GET detail | N/A | 97 | — | No internal equivalent |

**~1,002 LOC duplicated.** The download routes alone duplicate ~490 LOC of DOCX/HTML generation.

### Download route drift (subtle bugs)

| Aspect | Internal | Bot |
|--------|----------|-----|
| Issue label | Looks up `issue.title` from analysis | Uses `Issue ${state.issue_id}` |
| `edited` handling | Counts `edited` as accepted | Does not handle `edited` |
| `rejectedCount` | Explicit filter for `rejected` | `finalState.length - acceptedCount` |
| Params to generateHTML/DOCX | 10 (includes issues) | 9 (no issues) |

### Extraction opportunity

Extract `generateDOCX()` and `generateHTML()` into a shared `lib/negotiation-export.ts` module. Both routes call the same function with the same data shape. Eliminates ~490 LOC duplication and prevents future drift.

## 4. State Machine — Dead Code

**File:** `lib/negotiation-transitions.ts` (38 LOC)

```
broker_review -> send_round       -> counterparty_review
broker_review -> finalize         -> finalized
broker_review -> cancel           -> cancelled
counterparty_review -> counterparty_submit -> broker_review
counterparty_review -> cancel     -> cancelled
finalized -> (terminal)
cancelled -> (terminal)
```

`isTransitionAllowed()` returns `{ allowed, nextStatus, reason }`.

**Usage search result:** Only appears in:
1. `lib/negotiation-transitions.ts` (definition)
2. `tests/core/critical-paths.test.ts` (test coverage)

**Zero usage in any API route.** Routes implement ad-hoc guards:
- `finalize`: checks `status !== 'broker_review'`
- `cancel`: checks `status === 'finalized' || status === 'cancelled'`
- `send-round`: checks `status !== 'broker_review'`
- `submit`: checks `status !== 'counterparty_review'`

These are equivalent to the state machine's logic but hardcoded per-route. The state machine is well-designed but dead.

**Note:** `draft` is in the DB CHECK constraint but NOT in the state machine type. The create route skips `draft`, inserting directly as `broker_review`.

### Extraction recommendation

Wire up `isTransitionAllowed()` in extracted routes. Centralizes guard logic, eliminates per-route ad-hoc checks, makes the lifecycle explicit.

## 5. Database Schema

### `contract_negotiations`

| Column | Type | Notes |
|--------|------|-------|
| id | UUID PK | gen_random_uuid() |
| analysis_id | UUID FK | -> contract_analyses(id) ON DELETE CASCADE |
| status | TEXT | CHECK: draft, broker_review, counterparty_review, finalized, cancelled |
| document_type | TEXT | CHECK: msa, io, rider, amendment |
| counterparty_name | TEXT NOT NULL | |
| counterparty_email | TEXT | nullable |
| broker_decisions | JSONB | default '[]' |
| current_round | INTEGER | default 0 |
| created_by | TEXT | default 'mop' |
| webhook_url | TEXT | nullable |
| created_at | TIMESTAMPTZ | default NOW() |
| updated_at | TIMESTAMPTZ | default NOW() |
| finalized_at | TIMESTAMPTZ | nullable |

Indexes: analysis_id, status, created_at DESC, counterparty_email
Trigger: auto-update `updated_at`
RLS: Enabled, permissive USING(true)
Bot grants: SELECT, INSERT, UPDATE (no DELETE)

### `negotiation_rounds`

| Column | Type | Notes |
|--------|------|-------|
| id | UUID PK | |
| negotiation_id | UUID FK | -> contract_negotiations(id) ON DELETE CASCADE |
| round_number | INTEGER | |
| actor_type | TEXT | CHECK: broker, counterparty |
| actor_name | TEXT | nullable |
| actor_ip | TEXT | nullable |
| actor_user_agent | TEXT | nullable |
| decisions | JSONB | default '[]' |
| overall_notes | TEXT | nullable |
| submitted_at | TIMESTAMPTZ | default NOW() |

Constraint: UNIQUE(negotiation_id, round_number)
Indexes: negotiation_id, actor_type, submitted_at DESC

### `negotiation_links`

| Column | Type | Notes |
|--------|------|-------|
| id | UUID PK | |
| negotiation_id | UUID FK | -> contract_negotiations(id) ON DELETE CASCADE |
| token | TEXT UNIQUE NOT NULL | for /review/[token] URL |
| short_code | TEXT UNIQUE | added by migration 20260227 |
| link_type | TEXT | CHECK: review, final_approval |
| round_number | INTEGER | default 1 |
| expires_at | TIMESTAMPTZ | nullable |
| first_viewed_at | TIMESTAMPTZ | nullable |
| last_viewed_at | TIMESTAMPTZ | nullable |
| view_count | INTEGER | default 0 |
| created_at | TIMESTAMPTZ | default NOW() |

### Multi-tenant readiness

Migration `20300101` adds `tenant_id UUID REFERENCES tenants(id)` to all three tables. Drops open policies, creates tenant-scoped `USING (tenant_id = (auth.jwt()->>'tenant_id')::uuid)`. NOT NULL constraint prepared but commented out.

## 6. Merge Logic (`lib/contract-checker/merge.ts`, 158 LOC) — **Fully Horizontal**

### `findTextPosition(haystack, needle)` — 4-tier fuzzy matching

1. **Exact** — `indexOf`
2. **Normalized whitespace** — collapse whitespace, compare
3. **Ellipsis-trimmed** — strip leading/trailing `...`
4. **Sliding window** — 70% of needle length

### `mergeFinalDocument(rawText, issues, negotiationState)`

- Only applies revisions where BOTH broker AND counterparty accepted (or edited)
- Collects replacements with text positions
- Sorts by position descending (back-to-front to avoid offset shift)
- Does substring replacement

### `buildFinalNegotiationState(issues, brokerDecisions, counterpartyDecisions)`

- Maps each issue to final state with `broker_status`, `counterparty_status`, `final_text`
- Resolution logic:
  - Default: `issue.suggested_revision || issue.recommendation`
  - Counterparty countered + broker accepted -> use counterparty's text
  - Broker edited + counterparty accepted -> use broker's text

**Classification: 100% horizontal.** Generic text replacement + two-party decision resolution. Zero domain coupling.

## 7. Webhook System (`lib/negotiation-webhooks.ts`, 58 LOC)

Events: `negotiation.created`, `negotiation.round_submitted`, `negotiation.finalized`, `negotiation.cancelled`

URL resolution: `negotiation.webhook_url || process.env.PPCRM_WEBHOOK_URL`

Payload: `{ event, negotiation_id, document_type, counterparty_name, status, current_round, timestamp, ...extraData }`

Fire-and-forget via `.catch()`. No retry logic (unlike `lib/webhooks.ts` which has 3-attempt retry).

**Vertical coupling:** `PPCRM_WEBHOOK_URL` env var name. Trivially parameterizable.

## 8. Types (`types/negotiations.ts`, 82 LOC)

| Type | Fields |
|------|--------|
| `NegotiationDecision` | issue_id, status (accepted/rejected/edited/counter/pending), edited_text?, notes? |
| `ContractNegotiation` | mirrors DB table |
| `NegotiationRound` | mirrors DB table |
| `NegotiationLink` | mirrors DB table |
| `CreateNegotiationPayload` | analysis_id, document_type, counterparty_name/email, decisions, webhook_url? |
| `SubmitRoundPayload` | decisions, overall_notes?, actor_name? |
| `NegotiationWithRounds` | extends ContractNegotiation with rounds[], links[] |

**Note:** `CreateNegotiationPayload` and `SubmitRoundPayload` are defined but not imported in any API route. Routes manually destructure request bodies.

## 9. Negotiation UI (~1,572 LOC)

### `/app/negotiations/page.tsx` (230 LOC) — List page

- Client component, uses `supabaseUntyped` (anon key)
- Table: counterparty, doc type, status, round, dates
- Status badges: yellow (broker_review), blue (counterparty_review), green (finalized), gray (cancelled)
- Dark mode design system
- **Vertical:** TLP brand colors

### `/app/negotiations/[id]/page.tsx` (887 LOC) — Broker detail page

- Full broker workflow UI
- Issue cards: accept/reject/edit buttons per issue
- Counterparty response display
- Round history
- Action modals: Send Another Round, Finalize, Cancel
- Download links for finalized negotiations
- **Vertical:** "TLP Compliance", "TLP team" strings

### `/app/review/[token]/page.tsx` (455 LOC) — Counterparty review page

- Public-facing (no auth, token-validated)
- Per-issue: Accept, Reject, Counter with notes
- Pre-fills previous decisions for Round 2+
- **Vertical:** "The Lead Penguin", "TLP Compliance", "TLP team" strings

## 10. Horizontal vs Vertical Classification

### Horizontal (extractable)

| Component | LOC | Notes |
|-----------|-----|-------|
| State machine | 38 | Wire up in extraction (currently dead code) |
| Merge engine | 158 | Fully generic text replacement |
| Webhook dispatch | 58 | Parameterize env var name |
| Types | 82 | Parameterize document_type enum |
| Review routes (token-based) | 300 | Fully generic |
| Negotiation CRUD (deduplicated) | 500 | Merge internal + bot into shared handlers |
| Download/export (deduplicated) | 500 | Extract generateDOCX/generateHTML |
| Short-link redirect | 33 | Generic |
| DB migrations | 170 | Schema is generic |
| **Subtotal** | **~1,839** | |

### Vertical (stays in PPC)

| Component | LOC | Notes |
|-----------|-----|-------|
| Hardcoded brand strings | — | "TLP Compliance", "The Lead Penguin", `#dd72a6` |
| `document_type` enum values | — | msa, io, rider, amendment |
| `actor_name` defaults | — | "TLP Compliance", "TLP Compliance (API)" |
| Base URL | — | `tlpmop.netlify.app` |
| UI pages (with branding) | ~1,572 | Brand strings need parameterization |
| Bot role name | — | `mop_bot_reader` |

## 11. Bugs / Issues Found

1. **State machine is dead code** — well-tested but not called. Routes implement ad-hoc guards.
2. **Download route drift** — bot version doesn't handle `edited` status, uses simpler outcome logic, lacks issue title lookup.
3. **Bot generate-link lacks short_code** — feature drift from internal version.
4. **Review link tokens never expire by default** — `expires_at: null`. No automatic link revocation.
5. **`draft` status exists in DB CHECK but not in state machine** — create skips draft entirely.
6. **Unused type definitions** — `CreateNegotiationPayload` and `SubmitRoundPayload` defined but never imported.
7. **No negotiation webhook retry** — `negotiation-webhooks.ts` uses fire-and-forget, unlike `lib/webhooks.ts` which has 3-attempt retry.

## 12. Estimated @milo/contract-negotiation v0.1.0

| Component | LOC |
|-----------|-----|
| State machine (wired up) | 50 |
| Merge engine | 158 |
| Webhook dispatch (parameterized) | 65 |
| Types | 82 |
| Negotiation CRUD (deduplicated, shared handlers) | 500 |
| Download/export (deduplicated) | 500 |
| Review routes (token-based) | 300 |
| Short-link redirect | 33 |
| DB migrations | 170 |
| **Total** | **~1,858** |

UI pages (~1,572 LOC) are excluded from v0.1.0 — they need branding parameterization and are consumer-facing.

## 13. Open Questions

1. **Should `@milo/contract-negotiation` be a separate package or a module within `@milo/contracts`?** The negotiation system depends on `contract_analyses` (FK on analysis_id) and `merge.ts` (which reads issues from analyses). This coupling suggests it might be better as a submodule of `@milo/contracts` rather than standalone.

2. **Should the dead state machine be activated before extraction?** It's tested, correct, and centralizes all guard logic. Wiring it up is ~20 lines of changes across 6 routes.

3. **Webhook retry parity.** `lib/webhooks.ts` (signing) has 3-attempt retry. `lib/negotiation-webhooks.ts` does not. Should `@milo/contract-negotiation` share the same webhook helper with retry?

4. **Review link expiry.** Should extracted negotiation links have a configurable default expiry (e.g., 7 days)?
