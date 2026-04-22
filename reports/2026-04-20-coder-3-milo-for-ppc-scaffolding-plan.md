---
directive: "Milo-for-PPC scaffolding plan"
lane: research
coder: Coder-3
started: 2026-04-20 ~11:00 MDT
completed: 2026-04-20 ~11:45 MDT
supplements:
  - reports/2026-04-20-coder-4-milo-for-ppc-consolidation-plan.md (71a29b4)
  - reports/2026-04-19-coder-3-ppc-contract-three-way.md (40e2811)
  - reports/2026-04-20-coder-3-consumer-wave-plan.md (a69fb83)
---

# Milo-for-PPC Scaffolding Plan — 2026-04-20

## 1. Executive Summary

| Decision | Recommendation |
|----------|---------------|
| Repo strategy | **Fresh repo from milo-ops fork** (not milo-starter, not rename) |
| Starting point | milo-ops @ current HEAD, pruned to PPC-relevant pages |
| MOP code | Copy reference files into `_reference/` dir, never run live |
| Deployment | Vercel (same as milo-ops today) |
| Supabase | Shared instance (tappyckcteqgryjniwjg) — tenant='tlp' |
| Auth | Magic-link (per locked decision) via shared Supabase auth |
| Estimated scaffold LOC | ~500 new/modified (wiring + config), ~8,000 carried from milo-ops |

**Why fork milo-ops, not milo-starter?** milo-starter is designed for greenfield verticals with fresh domains, Stripe billing, and simple CRUD. Milo-for-PPC needs the existing operator workflows (pipeline, onboarding, contract review, Milo AI assistant, campaign matching, TrackDrive integration, EOD reports) that milo-ops already has. Stripping milo-starter's scaffold and rebuilding 8,000+ LOC of PPC-specific operator tooling would take weeks. A surgical fork of milo-ops + rewiring proxy calls to direct primitive consumption is faster and preserves proven UX.

---

## 2. Repo Structure

### (a) Recommendation: New Repo, Forked from milo-ops

```
moscaleflow/milo-for-ppc
├── src/
│   ├── app/                          # Next.js 14 App Router (from milo-ops)
│   │   ├── api/                      # Server routes (rewired to @milo/* direct)
│   │   ├── contract-review/          # Public signing/review pages (from MOP ref)
│   │   ├── dashboard-v2/             # Operator dashboard (from milo-ops)
│   │   ├── onboard/                  # Onboarding flow (from milo-ops)
│   │   ├── partners/                 # Partner management (from milo-ops)
│   │   ├── pipeline/                 # Pipeline board (from milo-ops)
│   │   ├── campaigns/                # Campaign management (from milo-ops)
│   │   ├── disputes/                 # Dispute handling (from milo-ops)
│   │   ├── invoices/                 # Billing (from milo-ops)
│   │   ├── contract-analysis/        # NEW: Analysis UI (from contract-checker ref)
│   │   ├── negotiation/              # NEW: Negotiation management (from MOP ref)
│   │   ├── settings/                 # NEW: Playbook/rule config (from contract-checker)
│   │   └── layout.tsx
│   ├── components/                   # UI components (from milo-ops)
│   ├── lib/                          # Server-side logic
│   │   ├── primitives/               # NEW: @milo/* SDK wrappers
│   │   │   ├── crm.ts               # Direct @milo/crm calls
│   │   │   ├── onboarding.ts        # Direct @milo/onboarding calls
│   │   │   ├── contract-analysis.ts  # Direct @milo/contract-analysis calls
│   │   │   ├── contract-signing.ts   # Direct @milo/contract-signing calls
│   │   │   └── contract-negotiation.ts
│   │   ├── ppc/                      # PPC vertical logic
│   │   │   ├── rules.ts             # ~80 RuleDefinition[] (consolidated)
│   │   │   ├── scoring-modifiers.ts  # ScoringModifier[] (MOP weights)
│   │   │   ├── pattern-plugins.ts    # PatternPlugin[] (2-party consent, etc.)
│   │   │   ├── ppc-config.ts        # Profiles, presets, thresholds
│   │   │   └── field-mapper.ts      # IO field extraction (from MOP)
│   │   ├── vertical.config.ts       # PPC brand/tenant settings
│   │   ├── supabase.ts              # Shared Supabase client
│   │   ├── supabase-admin.ts        # Service role client
│   │   └── teams-client.ts          # Teams notification (direct, not via PPCRM)
│   └── middleware.ts                 # Auth + public route exclusions
├── _reference/                       # MOP files for UI reference (not live code)
│   ├── signing-ui/                   # app/sign/[token]/ pages
│   ├── negotiation-ui/               # app/negotiations/, app/review/[token]/
│   ├── analysis-ui/                  # app/contract-analyzer/
│   └── README.md                     # "Reference only. Do not import."
├── vertical.config.ts                # Root-level config (re-export)
├── package.json
├── next.config.ts
├── tailwind.config.ts
└── tsconfig.json
```

### (b) Why NOT the Alternatives

| Option | Reason to reject |
|--------|------------------|
| Rename milo-ops | milo-ops serves TLP operations TODAY. Renaming disrupts live tooling while Milo-for-PPC is being built. |
| Extend milo-ops (same repo) | Feature flags + conditional routing = complexity. Clean separation is better — milo-ops stays stable for operators while Milo-for-PPC evolves fast. |
| milo-starter template | Designed for simple SaaS verticals (Stripe, magic-link, CRUD). Missing 8,000+ LOC of PPC operator tooling. Would require rebuilding everything milo-ops already has. |

### (c) Fork Strategy

```bash
# One-time fork (preserves git history for blame/reference)
gh repo create moscaleflow/milo-for-ppc --private
git clone ~/Miloops3.18/milo-ops milo-for-ppc
cd milo-for-ppc
git remote set-url origin git@github.com:moscaleflow/milo-for-ppc.git
git push -u origin main
```

---

## 3. Starting Point: What Carries Over from milo-ops

### (a) Pages to KEEP (PPC operator workflow)

| Page | Path | LOC (est.) | Why |
|------|------|-----------|-----|
| Dashboard v2 | `app/dashboard-v2/` | ~2,000 | Core operator view — stats, campaigns, health |
| Pipeline board | `app/pipeline/`, `app/pipeline-board/` | ~800 | Publisher/buyer pipeline stages |
| Partners | `app/partners/` | ~500 | Partner management, onboarding status |
| Onboarding | `app/onboard/` | ~300 | Publisher onboarding trigger |
| Contract review | `app/contract-review/` | ~400 | Public signing/review (already public routes) |
| Campaigns | `app/campaigns/` | ~600 | Campaign matching + management |
| Disputes | `app/disputes/` | ~400 | Dispute resolution workflow |
| Invoices | `app/invoices/` | ~500 | Billing/invoicing |
| EOD reports | `app/eod-reports/` | ~300 | End-of-day summary |
| Ping/post | `app/ping-post/`, `app/pings/` | ~400 | TrackDrive ping-post management |
| Operator views | `app/operator/` | ~600 | Per-operator dashboards |
| Milo AI | `app/jen/`, `app/malvin/`, `app/tiffani/` | ~800 | AI assistant personas |
| Observatory | `app/observatory/` | ~300 | Health monitoring |
| Support | `app/support/` | ~200 | Support interface |

### (b) Pages to REMOVE (non-PPC or superseded)

| Page | Path | Reason |
|------|------|--------|
| Feedback | `app/feedback/` | Internal tool, not PPC product |
| Setup | `app/setup/` | One-time setup, already done |
| Welcome | `app/welcome/` | Claim links — rethink for Milo-for-PPC auth |

### (c) Lib files to KEEP

| File | Purpose | Migration needed? |
|------|---------|-------------------|
| `anthropic.ts` | AI client wrapper | Already uses @milo/ai-client |
| `blacklist-client.ts` | Blacklist screening | Already uses @milo/blacklist |
| `publisher-onboarding.ts` | Onboarding convergence point | Rewire: PPCRM proxy → @milo/onboarding direct |
| `mop-client.ts` | MOP bot API proxy | **REMOVE** — replace with direct @milo/* calls |
| `ppcrm-client.ts` | PPCRM proxy | **REMOVE** — replace with direct @milo/* calls |
| `teams-client.ts` | Teams messaging | Rewire: direct webhook (not via PPCRM relay) |
| `contract-analysis-prompt.ts` | Analysis prompt | Keep — PPC prompt config layer |
| `milo-*.ts` (agent, tools, etc.) | Milo AI assistant | Keep — core feature |
| `supabase*.ts` | DB clients | Keep — retarget URL to shared instance |
| `trackdrive.ts`, `td-costs.ts` | TrackDrive integration | Keep — PPC data source |
| `billing-cycles.ts`, `invoice-generate.ts` | Billing | Keep |
| `campaign-matching.ts` | Campaign engine | Keep |
| `dispute-*.ts` | Dispute handling | Keep |

### (d) Lib files to REMOVE

| File | Reason |
|------|--------|
| `mop-client.ts` | MOP being decommissioned. Replace with `lib/primitives/` direct calls. |
| `ppcrm-client.ts` | PPCRM frozen forever. Replace with `lib/primitives/` direct calls. |
| `synthetic-tester.ts` | Dev-only tooling, not product |

---

## 4. MOP Reference Files to Copy

These go into `_reference/` for UI pattern extraction. NOT imported, NOT deployed.

### (a) Signing Page UI

| MOP Path | Purpose | Lines (approx.) |
|----------|---------|-----------------|
| `app/sign/[token]/page.tsx` | Public signing page render | ~300 |
| `app/sign/[token]/components/` | Signature capture, document viewer | ~500 |
| `app/api/sign/[token]/submit/route.ts` | Signature submission handler | ~200 |
| `app/api/sign/[token]/capture-view/route.ts` | View tracking | ~50 |
| `lib/signing/` | Token resolution, HMAC, templates | ~400 |

**Use for:** Building `app/contract-review/[token]/` page that renders documents from @milo/contract-signing.

### (b) Negotiation Review UI

| MOP Path | Purpose | Lines (approx.) |
|----------|---------|-----------------|
| `app/review/[token]/page.tsx` | Public negotiation review page | ~250 |
| `app/negotiations/` | Admin negotiation management | ~300 |
| `app/api/bot/negotiations/` | Negotiation CRUD endpoints | ~200 |

**Use for:** Building `app/negotiation/` pages that consume @milo/contract-negotiation.

### (c) Contract Analysis Admin

| MOP Path | Purpose | Lines (approx.) |
|----------|---------|-----------------|
| `app/contract-analyzer/` | Analysis submission + results UI | ~400 |
| `app/history/` | Analysis history browser | ~150 |
| `lib/contract-checker/` | Rule registry, scoring, post-processing | ~6,370 |

**Use for:** Building `app/contract-analysis/` page. Note: contract-checker standalone (Tagalog, settings, help, history) has better UI per consolidation plan — use that as primary reference, MOP for engine wiring.

---

## 5. Package Dependencies

### (a) @milo/* Primitives

| Package | Version | Ready for Milo-for-PPC? | Gap for v0.2.0 |
|---------|---------|------------------------|----------------|
| `@milo/ai-client` | v0.1.1 | **YES** — already consumed by milo-ops | None |
| `@milo/crm` | v0.1.1 | **YES** (reads) / **NO** (writes) | v0.2.0 needs: create/update counterparties, lead management, contact CRUD |
| `@milo/blacklist` | v0.1.0 | **YES** — already consumed by milo-ops | None |
| `@milo/contract-analysis` | v0.1.0 | **YES** — plugin system ready for PPC rules | v0.2.0 nice-to-have: template comparison mode |
| `@milo/contract-signing` | v0.1.0 | **PARTIAL** — SDK needs higher-level client API | v0.2.0 needs: `createDocument()`, `getSigningUrl()`, `voidDocument()` client functions |
| `@milo/contract-negotiation` | v0.1.0 | **PARTIAL** — same as signing | v0.2.0 needs: `createNegotiation()`, `submitRound()`, `getReviewUrl()` client functions |
| `@milo/onboarding` | v0.1.0 | **PARTIAL** — needs client SDK | v0.2.0 needs: `startOnboarding()`, `advanceStep()`, `getProgress()` client functions |

### (b) External Dependencies (carry from milo-ops)

```json
{
  "@ai-sdk/anthropic": "^3.x",
  "@supabase/ssr": "^0.9.x",
  "@supabase/supabase-js": "^2.99.x",
  "ai": "^6.x",
  "googleapis": "^171.x",
  "lucide-react": "^0.577.x",
  "next": "16.x",
  "react": "19.x",
  "recharts": "^3.x",
  "xlsx": "^0.18.x",
  "@hello-pangea/dnd": "^18.x"
}
```

### (c) New Dependencies (contract-checker features)

```json
{
  "docx": "^8.x"          // DOCX export (replaces MOP's custom implementation)
}
```

### (d) Primitive SDK Gap Assessment

The @milo/* packages v0.1.0 expose **engine internals** (types, transforms, DB schemas). For Milo-for-PPC to consume them directly (bypassing MOP/PPCRM proxy), each needs a **client SDK layer**:

```typescript
// What v0.1.0 has:
import { SigningDocument } from '@milo/contract-signing';  // types only

// What v0.2.0 needs:
import { createSigningClient } from '@milo/contract-signing/client';
const signing = createSigningClient({ supabase, tenantId: 'tlp' });
const doc = await signing.createDocument({ ... });
const url = await signing.getSigningUrl(doc.id);
```

**Coder-1 action item:** Add `/client` entry point to each primitive package with tenant-scoped CRUD operations.

---

## 6. Vertical Configuration

### (a) vertical.config.ts

```typescript
export const VERTICAL_CONFIG = {
  // Identity
  productName: "Milo for PPC",
  domain: "TBD",                          // platform subdomain pattern: tlp.<platform-domain>
  tagline: "AI-Powered PPC Operations",
  tenant: "tlp",

  // Branding
  companyName: "TLP Compliance",
  companyLegalName: "The Lead Penguin LLC",
  logoPath: "/tlp-logo.svg",
  primaryColor: "#1a1a2e",
  accentColor: "#4f46e5",

  // Email (per-tenant branding per Q4 answer)
  fromAddress: "Milo <milo@leadpenguin.com>",
  replyTo: "support@leadpenguin.com",

  // Auth (shared Supabase auth, RLS + tenant_id gating)
  cookiePrefix: "mfp",
  hmacSalt: "milo-for-ppc-admin-cookie-v1",

  // Infrastructure
  supabaseUrl: "https://tappyckcteqgryjniwjg.supabase.co",
  supabaseProjectRef: "tappyckcteqgryjniwjg",

  // Contract Analysis Display (Directive #35: no aggregate scores)
  analysis: {
    displayAggregateScore: false,
    displayLetterGrade: false,
    displayRiskBucket: false,
    issueOrdering: 'severity-desc',
    perIssueFields: ['severity', 'plainEnglish', 'recommendation', 'riderText'],
  },
} as const;
```

**No-Score UI Directive (locked, Directive #35):**

The contract analysis UI in Milo-for-PPC renders a severity-ordered issue list ONLY. No numeric scores, letter grades, or aggregate risk bucket labels. Operators care about "which 12 problems to fix" not "is this contract a B+".

Per-issue severity (CRITICAL/HIGH/MEDIUM/LOW) IS displayed — tells the operator how urgently to escalate each specific problem. The engine still computes weighted scores internally for sorting and for other verticals that want aggregate grades.

### (b) PPC Rules (~80 consolidated)

Per the three-way matrix (40e2811) and consolidation plan (71a29b4):

| Source | Rules | After dedup |
|--------|-------|-------------|
| MOP structured registry | 73 (43 PUB_*, 26 BUY_*, 2 legacy, 6 phantom) | 65 (drop 6 phantom + 2 legacy) |
| milo-ops inline prompt | 60+ | ~15 unique additions (rest overlap with MOP IDs) |
| contract-checker PlaybookRule | ~25 | ~3 unique (rest map to PUB_*/BUY_*) |
| **Total consolidated** | | **~80-83** |

Lives in `src/lib/ppc/rules.ts` as `RuleDefinition[]` — the PPC vertical plugin for @milo/contract-analysis.

### (c) Brand/Email Templates

| Template | Source | Purpose |
|----------|--------|---------|
| Signing invitation | MOP `lib/email/signing-invite.ts` | "You have a document to sign" |
| Onboarding welcome | PPCRM `onboardbot/welcome.ts` | "Welcome to TLP" |
| Negotiation review | MOP `lib/email/negotiation-review.ts` | "New contract version for review" |
| Analysis complete | milo-ops (inline) | "Your analysis is ready" |
| Stall reminder | PPCRM `onboardbot/stall.ts` | "Your onboarding is waiting on X" |

All templates reference `VERTICAL_CONFIG` for branding — no hardcoded strings.

### (d) DID/Webhook Config

| Integration | Current home | Milo-for-PPC location |
|-------------|-------------|----------------------|
| TrackDrive webhooks | milo-ops `src/app/api/td/` | Keep as-is (already in milo-ops) |
| Teams channels | PPCRM relay (`send-message` edge fn) | Direct webhook: `src/lib/teams-client.ts` with MS Teams Incoming Webhook URLs in env vars |
| Slack (future) | Not implemented | `src/lib/slack-client.ts` (Phase 2) |

### (e) Teams Integration — Direct (No PPCRM Relay)

Current: `milo-ops → PPCRM edge function → Teams webhook`
Target: `Milo-for-PPC → Teams webhook directly`

```typescript
// src/lib/teams-client.ts (rewritten, no PPCRM dependency)
const CHANNELS: Record<TeamsChannel, string> = {
  operations: process.env.TEAMS_WEBHOOK_OPERATIONS!,
  alerts: process.env.TEAMS_WEBHOOK_ALERTS!,
  billing: process.env.TEAMS_WEBHOOK_BILLING!,
  sales: process.env.TEAMS_WEBHOOK_SALES!,
  huddle: process.env.TEAMS_WEBHOOK_HUDDLE!,
  onboarding: process.env.TEAMS_WEBHOOK_ONBOARDING!,
};
```

---

## 7. Data Ownership

### (a) Shared Supabase Tables (tenant='tlp', read/write via primitives)

All @milo/* primitive tables on shared Supabase (tappyckcteqgryjniwjg):

| Table prefix | Primitive | Rows (TLP) |
|--------------|-----------|-----------|
| `crm_counterparties`, `crm_leads`, `crm_contacts` | @milo/crm | 601 + 9 + 18 |
| `blacklist_entries` | @milo/blacklist | 343 |
| `analysis_records`, `analysis_jobs` | @milo/contract-analysis | 33 + 27 |
| `signing_documents`, `signing_library` | @milo/contract-signing | 109 + TBD |
| `negotiation_records`, `negotiation_rounds`, `negotiation_links` | @milo/contract-negotiation | ~50 |
| `onboarding_records`, `onboarding_steps` | @milo/onboarding | 8 + 129 (post-migration) |

### (b) PPC Vertical-Only Tables (stay in Milo-for-PPC layer)

These tables serve PPC operations specifically and do NOT generalize to other verticals:

| Table | Purpose | Current home | Rows (est.) |
|-------|---------|-------------|------------|
| `prospect_pipeline` | Publisher/buyer pipeline stages | milo-ops Supabase | ~200 |
| `call_logs` | TrackDrive call records | MOP Supabase | ~50K+ |
| `ping_post_logs` | Ping/post records | MOP Supabase | ~10K+ |
| `campaign_assignments` | Publisher↔campaign mapping | milo-ops Supabase | ~100 |
| `operator_assignments` | Who manages which publisher | milo-ops Supabase | ~30 |
| `eod_reports` | End-of-day summaries | milo-ops Supabase | ~200 |
| `disputes` | Billing disputes | milo-ops Supabase | ~50 |
| `invoices` | Generated invoices | milo-ops Supabase | ~100 |
| `milo_conversations` | AI assistant chat history | milo-ops Supabase | ~500 |
| `daily_validations` | Daily data health checks | milo-ops Supabase | ~300 |

### (c) Tables Requiring Decision

| Table | Question | Options |
|-------|----------|---------|
| `rider_templates` | PPC-specific contract riders | Keep in vertical layer OR add to @milo/contract-signing as `signing_library` entries |
| `company_settings` | Per-tenant config (threshold, profiles) | Keep in vertical layer (tenant-specific settings don't generalize) |
| `partner_channels` | Which Teams channel per partner | Keep in vertical layer |

### (d) MOP Tables That Stay in MOP Forever

| Table | Reason |
|-------|--------|
| `call_logs` | TrackDrive data sink — massive volume, PPC-specific |
| `ping_post_logs` | Same — TrackDrive integration |
| `trackdrive_*` | TD config, mappings, costs |
| `financial_*` | Revenue reconciliation (CPA/RTB/CPL/CPQL-specific) |

These stay on MOP's Supabase (wjxtfjaixkoifdqtfmqd). Milo-for-PPC reads them via the existing TrackDrive integration code (already in milo-ops as `trackdrive.ts` + `td-costs.ts`).

---

## 8. Migration from Current State to Milo-for-PPC Live

### Phase 1: Scaffold + Wire (Week 1)

**Goal:** Repo exists, builds clean, runs locally with auth.

| Step | Action | Effort |
|------|--------|--------|
| 1.1 | Fork milo-ops into moscaleflow/milo-for-ppc | 10 min |
| 1.2 | Remove `mop-client.ts`, `ppcrm-client.ts` | Delete + fix imports |
| 1.3 | Create `src/lib/primitives/` SDK wrappers | ~300 LOC |
| 1.4 | Add @milo/* packages to package.json (file: refs → npm publish or file: same pattern) | Config only |
| 1.5 | Create `src/lib/ppc/rules.ts` (consolidated 80 rules) | ~500 LOC (from consolidation plan) |
| 1.6 | Rewire `publisher-onboarding.ts`: PPCRM proxy → @milo/onboarding direct | ~100 LOC delta |
| 1.7 | Rewire `teams-client.ts`: PPCRM relay → direct webhook | ~50 LOC delta |
| 1.8 | Update env vars: remove MOP_*, PPCRM_* → add TEAMS_WEBHOOK_*, confirm shared Supabase | Config |
| 1.9 | `npm run build` passes clean | Verify |
| 1.10 | Deploy to Vercel (new project, staging URL) | 15 min |

**Blocker:** @milo/* packages need client SDK layer (Step 5d above). If not ready, Phase 1 stubs the primitives/ wrappers with direct Supabase queries (same result, less abstraction).

**Fallback if SDK not ready:**
```typescript
// src/lib/primitives/onboarding.ts (direct query, no SDK)
import { supabaseAdmin } from '../supabase-admin';

export async function getOnboardingProgress(counterpartyId: string) {
  const { data } = await supabaseAdmin()
    .from('onboarding_records')
    .select('*, onboarding_steps(*)')
    .eq('counterparty_id', counterpartyId)
    .eq('tenant_id', 'tlp')
    .single();
  return data;
}
```

### Phase 2: Port MOP UIs (Week 2-3)

**Goal:** Signing, negotiation, and analysis UIs live in Milo-for-PPC, consuming primitives.

| Step | Action | Source | Effort |
|------|--------|--------|--------|
| 2.1 | Copy MOP signing UI to `_reference/signing-ui/` | MOP `app/sign/` | Copy only |
| 2.2 | Build `app/contract-review/[token]/page.tsx` | Reference + @milo/contract-signing | ~400 LOC |
| 2.3 | Copy MOP negotiation UI to `_reference/negotiation-ui/` | MOP `app/review/`, `app/negotiations/` | Copy only |
| 2.4 | Build `app/negotiation/page.tsx` + `app/negotiation/review/[token]/page.tsx` | Reference + @milo/contract-negotiation | ~400 LOC |
| 2.5 | Build `app/contract-analysis/page.tsx` (upload + results) | contract-checker UI + @milo/contract-analysis | ~600 LOC |
| 2.6 | Build `app/settings/page.tsx` (playbook config) | contract-checker settings | ~400 LOC |
| 2.7 | DOCX export integration | MOP `lib/contract-checker/docx-*.ts` | ~200 LOC |
| 2.8 | History page with search | contract-checker history | ~500 LOC |

### Phase 3: Operator Switch (Week 3-4)

**Goal:** TLP operators (Fab, Jackie, VA team) use Milo-for-PPC as their primary tool.

| Step | Action | Risk |
|------|--------|------|
| 3.1 | Domain setup: milo.leadpenguin.com (or similar) | LOW |
| 3.2 | Migrate operator accounts from milo-ops auth to Milo-for-PPC auth | LOW (magic-link, same emails) |
| 3.3 | Parallel run: both milo-ops and Milo-for-PPC available (1 week) | LOW |
| 3.4 | Verify: all operator workflows functional in Milo-for-PPC | MEDIUM |
| 3.5 | Cut over: operators switch bookmarks/links | LOW |
| 3.6 | milo-ops continues running (read-only mode) for fallback (2 weeks) | LOW |
| 3.7 | milo-ops archived | LOW |

### Phase 4: MOP Retirement (Week 4+)

**Goal:** MOP becomes TrackDrive microservice only — no contract/signing/negotiation features.

| Step | Action | Dependency |
|------|--------|-----------|
| 4.1 | Remove contract-related routes from MOP (signing, negotiation, analysis) | After Milo-for-PPC stable |
| 4.2 | Remove all @milo/* consumer code from MOP | After Phase 3 complete |
| 4.3 | MOP retains: `call_logs` ingest, `ping_post` management, TrackDrive config, financial reconciliation | Permanent |
| 4.4 | MOP Netlify deploy continues (TrackDrive webhooks depend on it) | Permanent until TD migration |
| 4.5 | MOP freeze guard stays active (prevent accidental contract writes) | Permanent |

---

## 9. Environment Variables

### Remove (MOP/PPCRM proxy vars no longer needed)

```
MOP_API_URL
MOP_BOT_API_KEY
PPCRM_BASE_URL
PPCRM_SUPABASE_ANON_KEY
PPCRM_MESSAGING_URL
PPCRM_MESSAGING_KEY
```

### Keep (shared infrastructure)

```
NEXT_PUBLIC_SUPABASE_URL=https://tappyckcteqgryjniwjg.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=...
SUPABASE_SERVICE_ROLE_KEY=...
ANTHROPIC_API_KEY=...
TRACKDRIVE_API_KEY=...
TRACKDRIVE_WEBHOOK_SECRET=...
```

### Add (direct integrations)

```
TEAMS_WEBHOOK_OPERATIONS=https://...
TEAMS_WEBHOOK_ALERTS=https://...
TEAMS_WEBHOOK_BILLING=https://...
TEAMS_WEBHOOK_SALES=https://...
TEAMS_WEBHOOK_HUDDLE=https://...
TEAMS_WEBHOOK_ONBOARDING=https://...
TENANT_ID=tlp
```

---

## 10. Decisions — Locked (2026-04-20, Directive #28)

| # | Question | Answer |
|---|----------|--------|
| Q1 | Repo name | `moscaleflow/milo-for-ppc` |
| Q2 | Domain | **DEFERRED** — platform pattern confirmed: subdomain-per-tenant (e.g., `tlp.<platform-domain>`). Actual domain TBD pending availability check. Design assumes a platform domain will be wired in later. |
| Q3 | TrackDrive data | Read-only cross-connection to MOP Supabase (temporary). When MOP retires to TD microservice, it writes to shared Supabase instead. No migration script — throwaway. |
| Q4 | Auth | Shared Supabase auth. Same operator accounts. RLS + tenant_id for role gating. Per-tenant email branding (TLP branding for tlp, mysecretary branding for mysecretary). One login across products. |
| Q5 | milo-ops fate | milo-ops BECOMES Milo-for-PPC via fork + rename. No parallel maintenance. Old branches archived after Phase 3. |
| Q6 | SDK timing | Phase 1: direct Supabase queries (unblocks immediately). Phase 2: proper @milo/* client SDKs when Coder-1 has bandwidth. |
| Q7 | Rules authoring | Coder-3 produces consolidated dedup mapping (~77 rules). Mark reviews. Coder-1 implements as @milo/contract-analysis rule plugins. TLP-specific overrides in vertical config. |
| Q8 | Tagalog | Phase 2. Not launch-blocking. Filipino VAs work in English today. |

### Implications for Scaffold

- **No parallel milo-ops:** Phase 3 "operator switch" becomes a rename + rebrand, not a migration between two live systems.
- **Auth simplification:** No account migration needed — same Supabase auth, same session cookies, same operators.
- **Domain placeholder:** Use `NEXT_PUBLIC_APP_URL` env var everywhere. Set to Vercel preview URL in Phase 1, final platform subdomain in Phase 3.
- **TrackDrive env vars stay:** Keep `MOP_SUPABASE_URL` + `MOP_SUPABASE_ANON_KEY` (read-only) for call_logs/ping_post access. Remove `MOP_BOT_API_KEY` (no more bot API proxy).

---

## 11. Dependency Graph

```
                    ┌─────────────────────────────┐
                    │  Milo-for-PPC Scaffolding    │
                    │  (This plan → Coder-1)       │
                    └──────────────┬──────────────┘
                                   │
        ┌──────────────────────────┼──────────────────────────┐
        │                          │                          │
        ▼                          ▼                          ▼
┌───────────────┐     ┌────────────────────┐     ┌──────────────────┐
│ @milo/* data  │     │ Fork milo-ops      │     │ Copy MOP ref     │
│ migrations    │     │ + prune + rewire   │     │ files to _ref/   │
│ (Coder-1 now) │     │ (Phase 1)          │     │ (Phase 1)        │
└───────┬───────┘     └────────┬───────────┘     └────────┬─────────┘
        │                      │                           │
        │                      ▼                           │
        │            ┌────────────────────┐                │
        │            │ PPC rules plugin   │                │
        │            │ (80 RuleDefinition)│                │
        │            └────────┬───────────┘                │
        │                     │                            │
        ▼                     ▼                            ▼
┌───────────────────────────────────────────────────────────────────┐
│ Phase 2: Port signing/negotiation/analysis UIs                     │
│ Requires: data migrated + primitives accessible + ref UI available │
└───────────────────────────────────────────────────────────────────┘
                                   │
                                   ▼
                    ┌─────────────────────────────┐
                    │ Phase 3: Operator switch     │
                    │ (requires Phase 2 stable)    │
                    └──────────────┬──────────────┘
                                   │
                                   ▼
                    ┌─────────────────────────────┐
                    │ Phase 4: MOP retirement      │
                    │ (requires Phase 3 complete)  │
                    └─────────────────────────────┘
```

---

## 12. Risk Matrix

| Risk | Probability | Impact | Mitigation |
|------|-------------|--------|------------|
| @milo/* SDK not ready for Phase 1 | HIGH | LOW | Use direct Supabase queries as fallback (Step 1 fallback) |
| milo-ops fork diverges during build | MEDIUM | LOW | Freeze milo-ops feature work during fork (operator changes go to Milo-for-PPC) |
| TrackDrive webhook interruption during switch | LOW | HIGH | Phase 4 only after parallel run proves stability |
| Operator confusion during parallel run | MEDIUM | MEDIUM | Clear communication + different domains |
| Rule consolidation produces regressions | MEDIUM | MEDIUM | Test consolidated rules against 33 existing analyses |
| MOP signing tokens break (public URLs change) | LOW | HIGH | Keep MOP signing routes alive until token expiry (30 days) |

---

## 13. Success Criteria

| Milestone | Verification |
|-----------|-------------|
| Phase 1 complete | `npm run build` passes, staging URL accessible, auth works |
| Phase 2 complete | Can analyze a contract, sign a document, review a negotiation — all through Milo-for-PPC |
| Phase 3 complete | Operators report no missing functionality vs. milo-ops + MOP combined |
| Phase 4 complete | MOP serves only TrackDrive webhooks, no contract/signing traffic |

---

## Summary

| Aspect | Decision |
|--------|----------|
| Repo | New: `moscaleflow/milo-for-ppc` (forked from milo-ops) |
| Stack | Next.js 16, React 19, TypeScript strict, Tailwind, Vercel |
| Data | Shared Supabase (tappyckcteqgryjniwjg), tenant='tlp' |
| Primitives | All 7 @milo/* packages, direct consumption |
| MOP role | Reference only (UI patterns), then TrackDrive microservice |
| PPCRM role | Dead (frozen forever, data already migrated) |
| Timeline | 4 phases, ~4 weeks from scaffold to operator switch |
| Biggest risk | Signing token URLs changing (mitigate: parallel run + 30-day expiry) |
