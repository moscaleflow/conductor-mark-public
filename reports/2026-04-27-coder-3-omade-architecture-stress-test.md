# OMADe Outreach Architecture Stress-Test — Coder-3 Report

**Coder-3 Research | 2026-04-28**
**Directive:** #74
**Precondition:** Directive #73 (OMADe engine audit) complete at `beb38c3`.

---

## Section 1: Capability Deltas

### Option A — Extend milo-outreach (add channel abstraction)

| Capability | What exists today | What needs build | Where it lives |
|---|---|---|---|
| Lead identity (email vs handle vs both) | `Lead.email`, `Lead.linkedin_url`. `LeadChannel = 'linkedin' \| 'email' \| 'both'` | Add `instagram_handle`, `tiktok_handle` fields to `leads` table. Extend `LeadChannel` to include `'instagram' \| 'tiktok' \| 'dm-queue'`. | milo-outreach `lib/types.ts:17`, `00001_initial_schema.sql:64-71` |
| Discovery (web_search vs Apify vs hybrid) | `research-agent.ts` uses `@milo/ai-client` + `web_search` tool. Directory-rotation config per tenant. | Replace/augment `web_search` with Apify scraper results import. Add `discovery_method` to `ResearchConfig`. New Apify adapter module. | milo-outreach `lib/research-agent.ts` |
| Draft generation (text vs text+post-reference) | `draft-generator.ts` builds text from Lead fields + VoiceConfig + OutreachTemplate. LinkedIn (<300 chars) and email supported. | Add post-reference context to `buildUserPrompt()`. Extend `Lead` with `recent_post_url`, `recent_post_text`. Adapt prompt for creator-specific tone (not B2B cold email). | milo-outreach `lib/draft-generator.ts:37-60` |
| Send channel (email vs manual DM queue vs API DM) | Email via Resend with three-gate safety. LinkedIn is "draft only, human sends" (locked architecture per ARCHITECTURE.md:766). | Add `dm-queue` as a send_log channel. Instagram/TikTok DM sending: nothing exists, would be new modules alongside `email-sender.ts`. | milo-outreach `lib/email-sender.ts` |
| Reply detection | Nothing automated. Replies are manually noted in `response_notes` + `reply_classification` fields. | Inbound email webhook (Resend). DM reply detection for social channels. | Green-field |
| Reply classification | `reply-classifier.ts`: 7 categories, Claude-powered, tenant-scoped. | Extend categories for creator context (e.g., "interested_collab", "rate_request", "already_partnered"). Adapt VoiceConfig prompts. | milo-outreach `lib/reply-classifier.ts` |
| Three-gate safety | 3 gates: `MILO_OUTREACH_ALLOW_REAL_SENDS` env + `tenant.real_sends_enabled` + `tenant.send_mode`. Full audit in `send_log.gate_state`. | Extend to DM channels. Gate semantics differ for DMs (no CAN-SPAM, but platform ToS risk). | milo-outreach `lib/email-sender.ts:24-223` |
| CAN-SPAM compliance | Unsubscribe headers, physical address footer, opt-out table with tenant/domain/global scoping. | Reusable as-is for email channel. Not applicable to DMs. | milo-outreach `lib/email-sender.ts:28-41`, `00001_initial_schema.sql:108-122` |
| Platform ToS compliance (DMs) | LinkedIn locked as "draft only, human sends" (ARCHITECTURE.md:766). No Instagram/TikTok ToS handling. | New: rate limiting per platform, human-in-loop enforcement for automated DMs, ToS violation detection. | Green-field |
| Multi-tenant isolation | All tables have `tenant_id`. Tenant config in `tenants` table. `requireTenant()` on every route. | Add 'omade' tenant row. OMADe-specific `VoiceConfig` and `ResearchConfig`. | milo-outreach `00001_initial_schema.sql:3-31`, `lib/tenant.ts` |
| Post-detection loop | Nothing in place. | New: Apify or platform API scraper, post matching logic, attribution to outreach. | Green-field |
| Sales attribution (Shopify codes) | Nothing in place. No Shopify integration anywhere. | New: Shopify SDK, affiliate code CRUD, order webhook receiver, attribution linking. | Green-field |
| Affiliate code lifecycle | Nothing in place. | New: code generation, assignment to creators, expiry rules, dashboard. | Green-field |
| Day-60 retainer ranking | Nothing in place. | New: scoring model, retention metric tracking, re-engagement trigger. | Green-field |

### Option B — Extract @milo/outreach primitive into milo-engine

| Capability | What exists today | What needs build | Where it lives |
|---|---|---|---|
| Lead identity | Same as Option A (milo-outreach `Lead` interface) | Extract `Lead` interface into `@milo/outreach`. Make fields extensible via `metadata: JSONB`. Add handle fields to base type. | New package: `milo-engine/packages/outreach/` |
| Discovery | `research-agent.ts` (220+ lines) with `ResearchConfig` interface already defined | Extract research loop (create run -> call Claude -> parse -> filter -> insert) as generic. Per-consumer config provides directory rotation, exclusions, prompt builder. Apify adapter as optional discovery source. | `@milo/outreach/research.ts` (extracted), `@milo/outreach/adapters/apify.ts` (new) |
| Draft generation | `draft-generator.ts` (120+ lines) with template + VoiceConfig | Extract core: prompt builder, template resolver, channel-aware length constraints. Consumer provides VoiceConfig + custom prompt sections. | `@milo/outreach/drafts.ts` (extracted) |
| Send channel | `email-sender.ts` (280+ lines) with three-gate safety | Extract gate evaluation engine + send_log pattern. Email remains default sender. DM channels as pluggable adapters (`SendAdapter` interface). | `@milo/outreach/send.ts` (extracted), `@milo/outreach/adapters/resend.ts`, `@milo/outreach/adapters/dm-queue.ts` (new) |
| Reply detection | Nothing | Same as Option A — green-field. | Green-field |
| Reply classification | `reply-classifier.ts` (160+ lines) | Extract classifier core. Consumer provides category list + VoiceConfig. 7 B2B categories as default set. | `@milo/outreach/classify.ts` (extracted) |
| Three-gate safety | Fully implemented in `email-sender.ts` | Extract as `evaluateGates(config, channel)` function. Gate definitions per channel type. | `@milo/outreach/gates.ts` (extracted) |
| CAN-SPAM | Implemented in email-sender.ts + opt_outs table | Extract opt-out management + unsubscribe header builder. | `@milo/outreach/compliance.ts` (extracted) |
| Platform ToS compliance | Nothing | Same as Option A — green-field. | Green-field |
| Multi-tenant isolation | tenant_id on every table + requireTenant() | Extract `TenantConfig` interface. Consumer provides tenant resolver. Tables use tenant_id FK. | `@milo/outreach/tenant.ts` (extracted) |
| Post-detection loop | Nothing | Same as Option A. | Green-field |
| Sales attribution | Nothing | Same as Option A. | Green-field |
| Affiliate code lifecycle | Nothing | Same as Option A. | Green-field |
| Day-60 retainer ranking | Nothing | Same as Option A. | Green-field |

### Option C — Greenfield @milo/creator-outreach

| Capability | What exists today | What needs build | Where it lives |
|---|---|---|---|
| Lead identity | Nothing creator-specific. B2B `Lead` in milo-outreach. | New `Creator` interface: handle-centric (instagram_handle, tiktok_handle primary; email secondary). Follower count, engagement rate, content niche, audience demo. | New: `milo-engine/packages/creator-outreach/types.ts` |
| Discovery | milo-outreach `research-agent.ts` is B2B directory-based | New: Apify scraper integration (TikTok creator search, Instagram hashtag search). No directory rotation — hashtag/niche-based discovery. | New: `creator-outreach/discovery.ts` |
| Draft generation | milo-outreach `draft-generator.ts` is B2B cold-email | New: Creator-specific prompts (reference their content, propose collab terms, attach product details). Multi-format: DM, email, comment. | New: `creator-outreach/drafts.ts` |
| Send channel | Email via Resend in milo-outreach | New: DM queue manager (human-sends for Instagram/TikTok). Email via Resend (can import `ResendEmailProvider` from @milo/contract-signing or copy Resend pattern). | New: `creator-outreach/send.ts` |
| Reply detection | Nothing | Same — green-field everywhere. | Green-field |
| Reply classification | milo-outreach has 7 B2B categories | New: Creator-specific categories (interested_collab, rate_request, already_partnered, not_interested, auto_responder). | New: `creator-outreach/classify.ts` |
| Three-gate safety | milo-outreach has it for email | Re-implement for creator channels. Simpler gates (env + manual confirm only; no CAN-SPAM for DMs). | New: `creator-outreach/gates.ts` |
| CAN-SPAM | milo-outreach has it | Only needed if email channel used. Copy pattern from milo-outreach or import from @milo/outreach if B also happens. | Copied or imported |
| Platform ToS compliance | Nothing | Same as all options — green-field. | Green-field |
| Multi-tenant isolation | milo-outreach tenant model | New tenant model or reuse `tenants` table from milo-outreach (if same Supabase). Needs 'omade' tenant row. | New or shared |
| Post-detection loop | Nothing | New. Same scope regardless of option. | Green-field |
| Sales attribution | Nothing | New. Same scope. | Green-field |
| Affiliate code lifecycle | Nothing | New. Same scope. | Green-field |
| Day-60 retainer ranking | Nothing | New. Same scope. | Green-field |

### Option D — Hybrid: Add 'omade' tenant to milo-outreach now; extract @milo/outreach only when forced

| Capability | What exists today | What needs build | Where it lives |
|---|---|---|---|
| Lead identity | milo-outreach `Lead` interface | Add `instagram_handle`, `tiktok_handle`, `follower_count`, `engagement_rate`, `content_niche` columns to `leads` table. New migration. | milo-outreach `00005_creator_fields.sql` |
| Discovery | `research-agent.ts` with web_search | New Apify adapter. Wire into research-agent as alternative discovery source. `ResearchConfig` gets `discovery_method: 'web_search' \| 'apify' \| 'hybrid'`. | milo-outreach `lib/apify-adapter.ts` (new), `lib/research-agent.ts` (modified) |
| Draft generation | B2B prompts in `draft-generator.ts` | Add creator-specific prompt path: if `tenant.outreach_type === 'creator'`, use creator prompts (reference post, propose collab). Template channel adds `'instagram-dm'`. | milo-outreach `lib/draft-generator.ts` (extended) |
| Send channel | Email + LinkedIn draft-only | Add `dm-queue` channel to `send_log`. Instagram/TikTok DMs are draft-only + human-sends (same pattern as LinkedIn). Email stays gated. | milo-outreach `lib/email-sender.ts` (extended), new `lib/dm-queue.ts` |
| Reply detection | Nothing | Green-field. | Same as all options |
| Reply classification | 7 B2B categories | Extend to 10 categories: add `interested_collab`, `rate_request`, `already_partnered`. Category set per tenant via `VoiceConfig`. | milo-outreach `lib/reply-classifier.ts` (extended) |
| Three-gate safety | 3-gate for email | Extend to cover DM-queue channel. DM gate simpler: env + manual only. | milo-outreach `lib/email-sender.ts` → refactored to `lib/send-gates.ts` |
| CAN-SPAM | Implemented for email | No change. DMs exempt. | As-is |
| Platform ToS compliance | LinkedIn locked as draft-only | Extend same pattern: Instagram/TikTok DMs are "draft only, human sends." | milo-outreach ARCHITECTURE.md (document decision) |
| Multi-tenant isolation | `tenant_id` everywhere | `INSERT INTO tenants VALUES ('omade', ...)` migration. OMADe-specific VoiceConfig + ResearchConfig. | milo-outreach `00005_omade_tenant.sql` |
| Post-detection loop | Nothing | Green-field. | New module |
| Sales attribution | Nothing | Green-field. Separate from outreach core. | New module (possibly in milo-for-ppc or new service) |
| Affiliate code lifecycle | Nothing | Green-field. | New module |
| Day-60 retainer ranking | Nothing | Green-field. | New module |

---

## Section 2: Costs

### Option A — Extend milo-outreach

| Dimension | Size | Notes |
|---|---|---|
| Ship OMADe v1 | M | Modify 6-8 existing files, add 2-3 new modules (Apify adapter, DM queue, creator prompts). Migration for new lead columns + OMADe tenant. Risk: B2B assumptions baked into prompts/types require careful surgery. |
| Ship TLP creator-style later | S | Already done — same codebase. |
| Extend to 2nd venture | S | Add another tenant row. Same code. |
| Maintenance | Medium risk — **god-app** | milo-outreach grows to serve B2B cold email + creator DM outreach + future channels. `if (tenant.outreach_type === 'creator')` branches multiply. Single deploy breaks all tenants. |

### Option B — Extract @milo/outreach primitive

| Dimension | Size | Notes |
|---|---|---|
| Ship OMADe v1 | XL | Extraction (6-8 files -> package), migration of milo-outreach to consume its own primitive, THEN build OMADe on top. Two large tasks before OMADe writes a single creator DM. |
| Ship TLP creator-style later | S | Primitive already extracted; wire up. |
| Extend to 2nd venture | S | New consumer of extracted primitive. |
| Maintenance | Low risk — **clean architecture** | But Pattern 12 violation: extraction cost paid upfront for speculative future consumers. Currently 1 consumer (mysecretary via milo-outreach). OMADe would be 2nd. |

### Option C — Greenfield @milo/creator-outreach

| Dimension | Size | Notes |
|---|---|---|
| Ship OMADe v1 | L | Build from scratch: types, discovery, drafts, send, classify, gates, tenant, migrations. No existing code to modify. But also no existing code to fight. |
| Ship TLP creator-style later | L | TLP would need to adopt creator-outreach OR keep milo-outreach for B2B + add creator-outreach alongside. Two outreach systems. |
| Extend to 2nd venture | M | If next venture is creator-style: S. If next venture is B2B: no reuse, back to milo-outreach. Bet on creator being the future. |
| Maintenance | Medium risk — **primitive sprawl** | Two outreach packages with duplicated patterns (gates, send_log, opt-outs, tenant resolution). Drift between them over time. |

### Option D — Hybrid: tenant-first, extract-later

| Dimension | Size | Notes |
|---|---|---|
| Ship OMADe v1 | M | Same as Option A in scope. Modify existing milo-outreach, add OMADe tenant + creator fields. Cheaper than B, more focused than C. |
| Ship TLP creator-style later | S | Already in the codebase. |
| Extend to 2nd venture | S-M | If milo-outreach gets too complex, extract THEN. Extraction trigger is concrete: 3rd tenant, or `if (tenant.outreach_type)` branches exceed 5. |
| Maintenance | Medium risk initially, but **extraction escape hatch** | Accepts god-app risk short-term. D2/D7 already document the extraction trigger: "after 3rd vertical with different outreach needs." OMADe IS the distant vertical D7 predicted. Extraction deferred until complexity proves it's needed. |

---

## Section 3: Risks

### Option A — Extend milo-outreach

1. **B2B assumptions hardcoded in prompts and types.** `draft-generator.ts` system prompt says "like a human founder, not a marketing bot" with B2B cold email structure. Creator outreach needs completely different tone and format. Extending will fork the prompt path but the function signatures stay B2B-shaped.
2. **Lead interface bloat.** `Lead` has 35+ fields already. Adding `instagram_handle`, `tiktok_handle`, `follower_count`, `engagement_rate`, `content_niche`, `recent_post_url` pushes it to 40+. Every query pays for all fields regardless of tenant type.
3. **Deploy coupling.** A bad OMADe migration or prompt change breaks mysecretary outreach. Single Vercel project, single deploy.

### Option B — Extract @milo/outreach primitive

1. **Pattern 12 violation.** D2 and D7 explicitly locked outreach extraction as DEFERRED until 3rd vertical proves it. OMADe is the 2nd consumer (or 3rd if TLP counts). Extracting now breaks the "wait for proof" gate without the proof.
2. **Extraction scope creep.** milo-outreach has 15 lib files + 6 migration files + 3 API route dirs. Extracting the primitive means splitting each file into "generic engine" and "mysecretary consumer." Past extractions (@milo/contract-signing: ~1,565 LOC; @milo/contract-negotiation: ~1,003 LOC) took full Coder-1 directives each. This is larger.
3. **Two-step delay.** OMADe v1 is blocked until extraction completes AND milo-outreach is migrated to consume the primitive. Mark wants a 30-creator seed campaign; this adds a full extraction sprint before creator work starts.

### Option C — Greenfield @milo/creator-outreach

1. **Duplicated safety infrastructure.** Three-gate safety, send_log audit trail, opt-out management, tenant resolution — all must be rebuilt or copied. If copy-pasted, future safety fixes need to be applied in two places.
2. **Pattern 12 direct violation.** Zero current consumers for @milo/creator-outreach at time of creation. It's speculative infrastructure for a venture that hasn't generated revenue. This is the exact failure mode D2 warns against.
3. **Divergent lead models.** `Lead` in milo-outreach vs `Creator` in creator-outreach will diverge. When TLP needs creator-style outreach later (publisher recruitment via social), which model wins? Forces a reconciliation that Option A/D avoids.

### Option D — Hybrid (tenant-first, extract-later)

1. **God-app accumulation.** milo-outreach serves mysecretary B2B + OMADe creator + TLP outreach. Conditional paths grow: `if (outreach_type === 'creator')` in draft-generator, reply-classifier, research-agent. Without discipline, complexity spirals.
2. **Deferred extraction debt.** "Extract later" often means "never extract." If OMADe ships and works, motivation to extract drops. Need a hard trigger (D7's "3rd vertical" gate or measurable complexity threshold).
3. **Creator-specific features squeezed into B2B schema.** Instagram/TikTok handle fields on a `leads` table designed for B2B contacts. Semantic mismatch (a "lead" is not a "creator"), though the pipeline mechanics (research -> draft -> send -> classify reply) are identical.

---

## Section 4: Tenant + Table Prefix Collision Check

### Option A / D (extend milo-outreach)

- **Tenant ID:** `omade`
- **Collision check:** Only existing tenant is `mysecretary` (seeded in `00001_initial_schema.sql:163`). No collision.
- **Table prefix needed:** None new. OMADe data goes into existing milo-outreach tables (`leads`, `outreach_templates`, `research_runs`, `opt_outs`, `send_log`) scoped by `tenant_id = 'omade'`.
- **New columns on `leads`:** `instagram_handle TEXT`, `tiktok_handle TEXT`, `follower_count INT`, `engagement_rate NUMERIC`, `content_niche TEXT`, `recent_post_url TEXT`
- **New migration:** `milo-outreach/supabase/migrations/00005_creator_fields.sql`
- **Collision with milo-for-ppc tables:** milo-outreach uses its own tables in shared Supabase. `leads` table exists in BOTH milo-outreach (outreach leads) AND milo-for-ppc (confirmed 200 on REST probe from audit). **Potential collision** — verify these are separate tables or share the same table with different tenant scoping.
- **Schema amendment:** D38 requires SCHEMA.md update before implementation. New columns + OMADe tenant definition.

### Option B (extract @milo/outreach)

- **Tenant ID:** `omade`
- **Table prefix:** `outreach_*` for extracted primitive tables (outreach_leads, outreach_templates, outreach_send_log, outreach_opt_outs, outreach_research_runs). Current milo-outreach tables are unprefixed (`leads`, `send_log`, etc.) — extraction would either rename or alias.
- **Collision check against #73 prefix list:** `outreach_*` prefix does NOT exist in shared Supabase. No collision.
- **Migration complexity:** Rename existing unprefixed tables to `outreach_*` prefix during extraction. All milo-outreach code updates to new table names. High churn.

### Option C (greenfield @milo/creator-outreach)

- **Tenant ID:** `omade`
- **Table prefix:** `creator_*` (creator_leads, creator_templates, creator_send_log, creator_opt_outs, creator_discovery_runs)
- **Collision check:** `creator_*` prefix does NOT exist in shared Supabase. No collision. `crm_*` is taken (D34 confirmed). No overlap.
- **New tables (all green-field):** creator_leads, creator_templates, creator_send_log, creator_opt_outs, creator_discovery_runs, creator_post_detections, creator_affiliate_codes
- **Isolation:** Fully isolated from milo-outreach. No migration risk to existing data.

### Option D (hybrid)

- Same as Option A. Tenant `omade`, no new prefix, new columns on `leads`, new migration. The `leads` table collision note applies.

---

## Section 5: Recommendation

**Option D — Hybrid: add 'omade' tenant to milo-outreach now; extract only when complexity forces it.**

This is the right call because OMADe is exactly the "distant vertical" D7 predicted — but D7 said to *build it first*, then decide whether to extract. The trigger was: "If leads.ts needs 40%+ changes, the Lead interface needs generics." We don't know the answer yet. Building OMADe as a tenant in milo-outreach gives us that answer empirically.

Option A loses because it's Option D without the extraction escape hatch articulated. Option D is A with discipline: document a hard extraction trigger (Pattern 12 compliant) so "later" has a concrete gate.

Option B loses because it inverts the sequence. D2 and D7 locked extraction as deferred. Extracting before building OMADe is speculative — we'd be guessing which interfaces to generalize before we have the consumer. Past @milo/* extractions (contract-signing at 1,565 LOC, contract-negotiation at 1,003 LOC) succeeded because they extracted FROM working vertical code WITH concrete consumers. Option B extracts before the 2nd consumer exists.

Option C loses on Pattern 12. A greenfield package with zero consumers at creation time is the exact anti-pattern D2 warns against. It also duplicates safety infrastructure (three-gate, opt-outs, send_log) that already works in milo-outreach, creating two divergent implementations of the same safety-critical code.

**Hard extraction trigger (document as a decision):** Extract @milo/outreach when ANY of:
- `if (tenant.outreach_type)` conditional branches exceed 5 across milo-outreach lib/
- A 3rd tenant with different outreach needs is added
- A deploy-breaking regression in one tenant's outreach code affects another tenant

**First shippable directive after Mark approves:** "Add 'omade' tenant to milo-outreach: tenant row migration, VoiceConfig for creator outreach, ResearchConfig for Apify-based discovery, creator lead columns." Pure data/config work, no new modules. Gets OMADe registered in the system so subsequent directives can target it.

---

## Section 6: Open Questions for Mark

**Q1: Tenant naming.** Is `omade` the right tenant ID? Or should it be `omade-protein`, `scaleflow-omade`, or match a future domain?

**Q2: DM platforms in v1.** Instagram DMs and TikTok DMs are both mentioned. Which ship in v1 for the 30-creator seed? Or are both "draft only, human sends" like LinkedIn? If both are manual, the send infrastructure is identical (dm-queue + human copy-paste).

**Q3: Apify account and scraper selection.** Discovery via Apify requires: an Apify account, selection of specific scrapers (TikTok creator search, Instagram hashtag search), and API key provisioning. Is this already set up or is it part of the first directive?

**Q4: Shopify integration timing.** Affiliate codes + sales attribution require Shopify API access. Is this v1 (30-creator seed) or deferred until the engine proves creators convert? If deferred, the outreach engine can ship without it.

**Q5: Verified sender domain for OMADe.** Email outreach to creators needs a verified Resend domain. Is it `omade.com`, `omadeprotein.com`, or another domain? Needs DNS setup before first email send.

**Q6: D7 extraction trigger — does OMADe count as the "distant vertical"?** D7 says "revisit after 3rd vertical." OMADe is the 2nd consumer of milo-outreach (mysecretary is 1st). Should we reset D7's gate to "after OMADe ships" (i.e., after empirical data about leads.ts change rate), or does OMADe alone trigger extraction now?

**Q7: Post-detection scope in v1.** Post-detection (monitoring whether creators actually post about OMADe after receiving product) is listed in context. Is this v1 or a Phase 2 capability? It's the largest green-field item and is independent of outreach send mechanics.

**Q8: Lead table collision.** The #73 audit confirmed a `leads` table exists in BOTH milo-for-ppc (confirmed 200 REST probe) AND milo-outreach (migration 00001). Are these the same physical table or separate tables? If separate, no action needed. If shared, OMADe data in the same table needs careful scoping verification.

---

Cross-references: Directive #73 (reports/2026-04-27-coder-3-omade-engine-audit.md), D2 (no horizontal extraction yet), D7 (outreach extraction deferred), D14 (milo-outreach v0 scope locked), ARCHITECTURE.md:766 (LinkedIn draft-only locked), Pattern 12 (no speculative infrastructure).
