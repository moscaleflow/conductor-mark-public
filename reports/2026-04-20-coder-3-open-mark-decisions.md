---
directive: "Consolidate all open Mark-decisions into single doc"
lane: research
coder: Coder-3
directive-number: 51
started: 2026-04-20
completed: 2026-04-20
updated: 2026-04-20 (D52 resolved D-OPEN-1/2, D53 resolved D-OPEN-3 through D-OPEN-26)
---

# Open Mark Decisions (as of 2026-04-20)

> Extracted from 14 reports, TASKS.md, and TODO.md. Cross-referenced against DECISION-LOG 1-63.
> **ALL 26 DECISIONS RESOLVED.** Zero open questions remain. Phase 1 fully unblocked.

---

## All decisions resolved

### ~~D-OPEN-1: System prompt source for PPC contract analysis~~ RESOLVED

- **Answer:** Option A — port milo-ops hand-crafted prompt for Phase 1. Migrate to generated in Phase 2.
- **Resolved:** 2026-04-20 (Mark approved, Directive #52)

### ~~D-OPEN-2: AI model tier for PPC contract analysis~~ RESOLVED

- **Answer:** Sonnet default for analysis. Opus override for specific judgment paths.
- **Resolved:** 2026-04-20 (Mark approved, Directive #52)

### ~~D-OPEN-3: Signing template ownership~~ RESOLVED

- **Answer:** Option A — plugin interface. Signing package owns lifecycle, verticals provide render functions.
- **Resolved:** 2026-04-20 (Mark approved, Directive #53)

### ~~D-OPEN-4: Tax form templates~~ RESOLVED

- **Answer:** Option A — horizontal. IRS forms (W-9, W-8BEN, W-8BEN-E) ship with @milo/contract-signing.
- **Resolved:** 2026-04-20 (Mark approved, Directive #53)

### ~~D-OPEN-5: Compliance model~~ RESOLVED

- **Answer:** Option B — PPC-only. Formula too domain-specific to generalize. Each vertical defines its own.
- **Resolved:** 2026-04-20 (Mark approved, Directive #53)

### ~~D-OPEN-6: Profile selection UI~~ RESOLVED

- **Answer:** Option C — auto-detect from contract metadata with manual override.
- **Resolved:** 2026-04-20 (Mark approved, Directive #53)

### ~~D-OPEN-7: Settings UI scope~~ RESOLVED

- **Answer:** Option C — profiles only for Phase 1. Per-rule thresholds added when operators request.
- **Resolved:** 2026-04-20 (Mark approved, Directive #53)

### ~~D-OPEN-8: Version tracking schema~~ RESOLVED

- **Answer:** Option A — use milo-ops richer schema (version, status, short_slug, diff_from_previous).
- **Resolved:** 2026-04-20 (Mark approved, Directive #53)

### ~~D-OPEN-9: CompanySettings approach~~ RESOLVED

- **Answer:** Option A — vertical.config.ts for defaults (per Thesis 2.5). DB for per-tenant overrides. Remove hardcoded TLP fallback.
- **Resolved:** 2026-04-20 (Mark approved, Directive #53)

### ~~D-OPEN-10: MOP long-term fate~~ RESOLVED

- **Answer:** Option A — MOP stays alive as dedicated TrackDrive microservice indefinitely. Revisit if hosting cost matters.
- **Resolved:** 2026-04-20 (Mark approved, Directive #53)

### ~~D-OPEN-11: MOP dashboard / refresh-ping-agg cron~~ RESOLVED

- **Answer:** Operators don't use MOP or milo-ops dashboards today. Waiting for Milo-for-PPC (built on milo-ops UI foundation). Leave refresh-ping-agg DISABLED. Archive MOP dashboard code in Phase 4 retirement. milo-ops UI becomes Milo-for-PPC dashboard via Phase 1 fork, Phase 2 UI port.
- **Resolved:** 2026-04-20 (Mark confirmed, Directive #53)

### ~~D-OPEN-12: contract_issue_resolutions table~~ RESOLVED

- **Answer:** Option B — defer to v0.3 resolution tracking design.
- **Resolved:** 2026-04-20 (Mark approved, Directive #53)

### ~~D-OPEN-13: publisher_id/buyer_id reconciliation table~~ RESOLVED

- **Answer:** Build mapping table between Phase 1 and Phase 2.
- **Resolved:** 2026-04-20 (Mark approved, Directive #53)

### ~~D-OPEN-14: finalized_documents table~~ RESOLVED

- **Answer:** Skip confirmed. 0 rows, 0 files. Design fresh signing_attachments table if needed later.
- **Resolved:** 2026-04-20 (Mark approved, Directive #53)

### ~~D-OPEN-15: contract-checker future~~ RESOLVED

- **Answer:** Archive after Phase 2 absorption. Data is test/demo.
- **Resolved:** 2026-04-20 (Mark approved, Directive #53)

### ~~D-OPEN-16: Template comparison~~ RESOLVED

- **Answer:** Option B — PPC vertical for now. Promote to horizontal if 2nd vertical needs it.
- **Resolved:** 2026-04-20 (Mark approved, Directive #53)

### ~~D-OPEN-17: IO field extraction~~ RESOLVED

- **Answer:** PPC vertical PatternPlugin. Other verticals have different structured fields.
- **Resolved:** 2026-04-20 (Mark approved, Directive #53)

### ~~D-OPEN-18: Revision template system~~ RESOLVED

- **Answer:** Defer to Phase 2.
- **Resolved:** 2026-04-20 (Mark approved, Directive #53)

### ~~D-OPEN-19: Rule snapshot storage format~~ RESOLVED

- **Answer:** Overrides + delta format. Base version tracked by engine version number.
- **Resolved:** 2026-04-20 (Mark approved, Directive #53)

### ~~D-OPEN-20: Letter grade computation~~ RESOLVED

- **Answer:** Defer. Add when a vertical requests it.
- **Resolved:** 2026-04-20 (Mark approved, Directive #53)

### ~~D-OPEN-21: Base rule prompt_instruction sourcing~~ RESOLVED

- **Answer:** Copy from MOP registry.ts, edit to remove PPC jargon. Already battle-tested on 33 analyses.
- **Resolved:** 2026-04-20 (Mark approved, Directive #53)

### ~~D-OPEN-22: Signed HTML storage~~ RESOLVED

- **Answer:** Keep HTML for Phase 1. PDF generation is Phase 2+ enhancement.
- **Resolved:** 2026-04-20 (Mark approved, Directive #53)

### ~~D-OPEN-23: Signing webhook URL~~ RESOLVED

- **Answer:** Leave as historical reference. New documents use Milo-for-PPC webhook URL.
- **Resolved:** 2026-04-20 (Mark approved, Directive #53)

### ~~D-OPEN-24: VA Bees / @milo/time-tracking~~ RESOLVED

- **Answer:** Defer all 7 sub-questions. Not in scope for current phase.
- **Resolved:** 2026-04-20 (Mark approved, Directive #53)

### ~~D-OPEN-25: Operator capture audit~~ RESOLVED

- **Answer:** Defer all 5 sub-questions. Revisit when @milo/crm v0.3 scoped.
- **Resolved:** 2026-04-20 (Mark approved, Directive #53)

### ~~D-OPEN-26: MOP Realtime~~ RESOLVED

- **Answer:** Informational only. Polling is fine for dashboards.
- **Resolved:** 2026-04-20 (Mark approved, Directive #53)

---

## Recently resolved (this session)

Questions that were open in reports but have since been answered by DECISION-LOG entries or completed work:

| Question | Resolution | Source |
|----------|-----------|--------|
| D-OPEN-1: System prompt source | Port hand-crafted for Phase 1, generate in Phase 2 | Mark approved (D52) |
| D-OPEN-2: AI model tier | Sonnet default, Opus override for specific paths | Mark approved (D52) |
| D-OPEN-3: Signing templates | Plugin interface — vertical provides render functions | Mark approved (D53) |
| D-OPEN-4: Tax forms | Horizontal — ship with @milo/contract-signing | Mark approved (D53) |
| D-OPEN-5: Compliance model | PPC-only — each vertical defines own formula | Mark approved (D53) |
| D-OPEN-6: Profile selection | Auto-detect with manual override | Mark approved (D53) |
| D-OPEN-7: Settings UI | Profiles Phase 1, per-rule when requested | Mark approved (D53) |
| D-OPEN-8: Version tracking | milo-ops richer schema | Mark approved (D53) |
| D-OPEN-9: CompanySettings | vertical.config.ts defaults + DB overrides | Mark approved (D53) |
| D-OPEN-10: MOP fate | TrackDrive microservice indefinitely | Mark approved (D53) |
| D-OPEN-11: MOP dashboard | Operators not using either dashboard. refresh-ping-agg stays disabled. milo-ops UI becomes Milo-for-PPC. | Mark confirmed (D53) |
| D-OPEN-12: issue_resolutions | Defer to v0.3 | Mark approved (D53) |
| D-OPEN-13: ID reconciliation | Build mapping table between Phase 1-2 | Mark approved (D53) |
| D-OPEN-14: finalized_documents | Skip confirmed | Mark approved (D53) |
| D-OPEN-15: contract-checker | Archive after Phase 2 | Mark approved (D53) |
| D-OPEN-16: Template comparison | PPC for now, promote if needed | Mark approved (D53) |
| D-OPEN-17: IO field extraction | PPC PatternPlugin | Mark approved (D53) |
| D-OPEN-18: Revision templates | Defer to Phase 2 | Mark approved (D53) |
| D-OPEN-19: Rule snapshot | Overrides + delta | Mark approved (D53) |
| D-OPEN-20: Letter grade | Defer until requested | Mark approved (D53) |
| D-OPEN-21: Base rule prompts | Copy from MOP, remove PPC jargon | Mark approved (D53) |
| D-OPEN-22: Signed HTML | Keep HTML Phase 1, PDF Phase 2+ | Mark approved (D53) |
| D-OPEN-23: Webhook URL | Leave historical | Mark approved (D53) |
| D-OPEN-24: VA Bees | Defer all 7 | Mark approved (D53) |
| D-OPEN-25: Operator capture | Defer all 5, revisit @milo/crm v0.3 | Mark approved (D53) |
| D-OPEN-26: MOP Realtime | Informational only, polling is fine | Mark approved (D53) |
| Scaffolding Q1: Repo name | `moscaleflow/milo-for-ppc` | Scaffolding plan §10 (D28 locked) |
| Scaffolding Q2: Domain | `justmilo.app` + `justmilo.co` registered | D63 |
| Scaffolding Q3: TrackDrive data | Read-only cross-connection to MOP Supabase | Scaffolding plan §10 |
| Scaffolding Q4: Auth | Shared Supabase auth, same operator accounts | Scaffolding plan §10 |
| Scaffolding Q5: milo-ops fate | milo-ops BECOMES Milo-for-PPC via fork + rename | Scaffolding plan §10 |
| Scaffolding Q6: SDK timing | Phase 1 direct queries, Phase 2 SDKs | Scaffolding plan §10 |
| Scaffolding Q7: Rules authoring | Coder-3 mapping done (529d74f), Mark reviews, Coder-1 implements | Scaffolding plan §10 |
| Scaffolding Q8: Tagalog | Phase 2, not launch-blocking | Scaffolding plan §10 |
| Negotiation: separate package? | Yes — @milo/contract-negotiation (D44/D46) | D44 |
| Negotiation: dead state machine? | Activated during extraction | D44 D3 |
| Negotiation: webhook retry? | Shared via @milo/contract-signing fireWebhook | D44 D6 |
| Negotiation: link expiry? | 7-day default, configurable | D44 D9 |
| Short URL ownership? | @milo/contract-signing owns resolver, siblings register | D44 Q2 |
| HMAC bug fix? | Fixed during extraction — unified format | D43 |
| Multi-tenant from v0.1.0? | All packages ship with tenant_id NOT NULL | D42/D43/D44/D46 |
| Pattern injection extension point? | VerticalRulePack with PatternPlugin[] | D62 |
| MOP+milo-ops reconciliation strategy? | 45 base (MOP) + 32 PPC (milo-ops) = 77 total | Dedup mapping (529d74f) |
| Rule engine extraction boundary? | VerticalRulePack plugin interface | D62 |
| 6 phantom rule IDs? | Accounted for in dedup mapping (5 mapped to base, 1 dropped) | Dedup mapping (529d74f) |
| Contract version bump? | Straight to v0.2.0, no pending bug fixes | D62 (v0.2.0 shipped) |
| PPCRM end-of-life? | Frozen forever, permablocked | TASKS.md |
| MOP unfreeze? | Done (D45), selective contract freeze deployed (D52) | D45, D52 |
| Wave 1 authorization? | Complete (D57) | D57 |
| PPCRM onboardbot rewrite? | Deferred (PPCRM permablocked) | TASKS.md |
| MOP contract UI future? | Build fresh in Milo-for-PPC (UI port plan) | UI port plan (f0c2160) |
| milo-ops direct consumption? | Done — Wave 1 uses SDKs directly | D57 |
| @milo/blacklist scope expansion? | v0.2.0 shipped (ip_addresses, linkedin_urls) | D49 |
| Onboarding counterparty ID preservation? | UUIDs preserved (partner_id = counterparty_id) | D58 |
| Onboarding step ordering? | 15-step flow used in migration | D58 |
| Onboarding test data? | All 8 rows migrated | D58 |
| Onboarding non-required steps? | Bypassed steps marked 'skipped' | D58 |
| Signing partner_documents count? | 0 rows (signing_library empty) | D54 |
| Signing active sessions? | 68 pending migrated, links handled by redirect strategy | D54 |
| Signing generated_pdf_url? | 0 storage files (HTML-based signing) | D54 |
| Contract freeze deployment? | D52 deployed, used for all 3 migrations | D52/D53/D54/D56 |
| Route-level fence vs accept-and-reconcile? | Route-level fence deployed and used | D52 |
| Review page GET exemption? | Exempt (read-adjacent) | D52 |
| Financial crons re-enable? | Done via contract-writes deploy (618a6f7) | TASKS.md |
| Milo-for-PPC ownership? | Mark owns roadmap and ship decisions | D55 |
| Rule consolidation owner? | Coder-3 done, Mark reviews | Scaffolding Q7 |
| Consolidation sequencing? | Extraction first (done), consolidation Phase 2 | D42-D46 already shipped |
| Tagalog priority? | Phase 2 per Q8 | Scaffolding plan §10 |

---

## Summary

| Urgency | Count | Status |
|---------|-------|--------|
| ~~HIGH~~ | ~~2~~ | **RESOLVED (D52)** |
| ~~MEDIUM~~ | ~~9~~ | **RESOLVED (D53)** |
| ~~LOW~~ | ~~15~~ | **RESOLVED (D53)** |
| Recently resolved | 64 total (38 prior + 2 D52 + 24 D53) | -- |
| **Total open** | **0** | |

**Phase 1 fully unblocked.** All 3 Coder-1 gates clear: rule pack spec, Vercel host, open decisions.
