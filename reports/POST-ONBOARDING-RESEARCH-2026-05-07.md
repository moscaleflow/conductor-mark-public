# Post-Company-Onboarding Research Spec

**Author:** Coder-3 (Research lane)
**Date:** 2026-05-07
**Scope:** What happens after a new company admin signs up via /onboard/company and logs in for the first time

---

## Executive Summary

The company admin onboarding flow (`/onboard/company`) is fully built and functional: 4-step cinematic wizard collects company name, industry, admin info, and workspace slug, then calls `POST /api/tenants/create` to provision a tenant + owner profile. The admin gets told "check your email for a magic link."

**What exists after that point is incomplete.** The post-login journey currently lands the new admin on the operator page (`/operator`) which expects TrackDrive data, call history, and buyer/publisher relationships -- none of which exist for a brand-new company. There is no "connect your platforms" wizard, no TrackDrive credential collection UI, no document upload for MSA/IO, and no company logo upload mechanism.

---

## A. TrackDrive Integration -- What Do We Need From New Companies?

### Current State

**Single-tenant TrackDrive client.** The TrackDrive client (`src/lib/trackdrive.ts`, 848 lines) uses two global env vars:
- `TD_API_BASE_URL` -- the TrackDrive API endpoint
- `TD_API_AUTH` -- the authorization header value (Bearer token)

There is zero tenant awareness. Every API call uses the same credentials. The client supports reads (calls, campaigns, traffic sources, buyers, offers, leads, change logs, DIDs) and writes (create/update/pause publishers, create/update buyers, create routes, DID management).

**Key file paths:**
- `src/lib/trackdrive.ts` (lines 34-35) -- env var usage
- `src/lib/td-costs.ts` -- platform cost constants (ping costs, DNC check costs, per-second billing rates)
- `src/app/api/cron/sync/route.ts` -- 15-minute sync cron (uses single TD credential set)
- `src/app/api/cron/td-change-detection/route.ts` -- change detection cron
- `src/app/api/td-changes/review/route.ts` -- TD mutation review (mutations disabled per Decision 175.1)

### What TrackDrive Credentials/Config Milo Needs Per Tenant

1. **TD API Auth Token** -- each company has their own TrackDrive account with its own API key. This is the `Authorization` header value (format: `Token token=<key>`).
2. **TD API Base URL** -- typically `https://api.trackdrive.net/api/v1` but could differ per account.
3. **Offer IDs** -- which TrackDrive offers (campaigns) belong to this company. Currently the sync pulls ALL offers.
4. **Timezone** -- currently hardcoded to Denver (`src/lib/denver.ts`). New companies may be in different timezones.

### Minimum TrackDrive Setup for Data Flow

1. Company must have an active TrackDrive account with API access enabled
2. Company provides their API token (read-only is sufficient for data sync; write access needed only if Milo manages DID provisioning)
3. Milo stores the token encrypted per tenant
4. The sync cron (`/api/cron/sync`) must become tenant-aware -- iterate over all active tenants, use each tenant's credentials
5. The existing TD client needs a factory pattern: `createTrackDriveClient(tenantId)` instead of global singleton

### Can We Automate TrackDrive Account Creation?

**No.** TrackDrive does not expose an account provisioning API. The `/me` endpoint confirms an existing account's identity but cannot create new ones. TrackDrive account setup is a manual process done through their admin dashboard. Companies must already have a TrackDrive account before connecting to Milo.

### Questions to Ask New Companies About Their TrackDrive Setup

1. "Do you have a TrackDrive account?" (required)
2. "What is your TrackDrive API key?" (required -- they get this from TrackDrive settings)
3. "Which offers/campaigns should Milo track?" (optional -- could auto-discover via `GET /offers`)
4. "What timezone does your team operate in?" (for accurate daily reporting)
5. "Do you want Milo to have write access (DID provisioning, buyer settings), or read-only monitoring?" (determines which API key to request)

### What Needs to Be Built

1. **Tenant-scoped credential storage** -- `tenants` table needs columns: `td_api_auth` (encrypted), `td_api_base_url`, `td_timezone`, `td_write_enabled`
2. **TrackDrive connection wizard** -- a post-login setup step that collects the API key, validates it via `GET /me`, and stores it
3. **Tenant-aware TrackDrive client factory** -- replace global `TD_API_AUTH`/`TD_API_BASE_URL` with per-tenant lookup
4. **Tenant-aware sync cron** -- iterate over tenants with valid TD credentials
5. **Connection status indicator** -- show green/red status on the operator page for TrackDrive connectivity

---

## B. Document Management -- MSA, IO, Tax Docs

### Current State

The contract/document infrastructure is mature but oriented toward operator-initiated workflows, not self-service upload by new companies.

**Contract Documents Table:** `contract_documents` in Supabase stores:
- `document_url` -- URL to the file (currently expects external hosting; v1.1 TODO for S3/blob fetch)
- `document_type` -- msa, io, w9, w8ben, etc.
- `counterparty_name`, `prospect_id` -- entity linkage
- `status` -- draft, sent, signed, redlined, etc.
- `analyzed_clauses`, `analysis_result`, `risk_level` -- AI analysis output
- `signing_token`, `public_review_url`, `short_slug` -- for buyer review links
- `contract_group_id`, `parent_version_id`, `version` -- version chain

**Key file paths:**
- `src/lib/tools/analyze-contract.ts` -- AI clause analysis (Opus tier)
- `src/lib/tools/publish-contract-for-buyer-review.ts` -- generates review URLs, sends email via Resend
- `src/lib/tools/ingest-returned-contract.ts` -- ingests redlined contracts, diffs against prior version
- `src/lib/contract-analysis-prompt.ts` -- 60+ rule contract analysis prompt (495 lines)
- `src/app/api/contract/documents/route.ts` -- fetch docs by entity, grouped by contract_group_id
- `src/app/api/documents/entity/route.ts` -- unified doc endpoint (partner + local)
- `src/app/api/webhooks/document-signed/route.ts` -- webhook for signing events, triggers onboarding auto-advancement
- `src/lib/onboarding-automation.ts` -- auto-advances onboarding steps when documents are signed

**Contract Signing Flow:** Currently uses a custom token-based system (not DocuSign):
1. Operator creates a contract_documents row
2. `publish_contract_for_buyer_review` generates a public review URL (`/contracts/review/:id` and `/c/:slug`)
3. Email sent to buyer via Resend with the review link
4. Buyer reviews at the public URL
5. When signed, webhook at `/api/webhooks/document-signed` fires
6. `onboarding-automation.ts` auto-advances the corresponding onboarding step

### Where Do MSAs/IOs Currently Get Stored?

Documents are stored as URLs in `contract_documents.document_url`. The actual files are external -- there is no Supabase Storage bucket for contract files yet. The `notes` column is used as a text staging area for v1 analysis (line 293 of `analyze-contract.ts`: "v1.1 will fetch document_url").

### Is There Already an Upload Mechanism?

**Partially.** The buyer onboarding pages (`/onboard/buyer/[token]` and `/onboard/publisher/[token]`) are form-submission pages for buyers/publishers being onboarded BY an operator. They do not include file upload for contracts. The `creatives_submitted` step in the publisher onboarding flow references `integration: { type: 'file_upload', bucket: 'creatives' }` but the actual upload UI for this is not built.

### What's the Current Contract Signing Flow?

Custom built, not DocuSign. The flow is:
1. Token-based review URLs (HMAC-signed tokens via `src/lib/onboarding-token.ts`)
2. Email delivery via Resend (from `support@theleadpenguin.com`)
3. Review pages at `/contract-review/` and `/c/[slug]`
4. Webhook-based status tracking

### Where Would a Legal Disclaimer Checkbox Need to Go?

Two locations:
1. **Company onboarding (`/onboard/company`)** -- before the "Create My Workspace" button in the review phase (line 542 of `src/app/onboard/company/page.tsx`). This would be a TOS/privacy agreement.
2. **Team invite claim (`/join/[tenant]/[code]`)** -- before the "Join" button in the confirm phase. Same TOS checkbox.

### What Needs to Be Built

1. **File upload mechanism** -- Supabase Storage bucket for contract documents, with upload API
2. **MSA/IO template system** -- pre-populated MSA templates per industry that can be customized
3. **Self-service document upload in onboarding** -- step in post-login wizard for company admin to upload existing MSA/IO/W-9
4. **TOS/privacy checkbox** -- on `/onboard/company` review step and `/join/[tenant]/[code]` confirm step
5. **Document gallery** -- visible on operator page showing all uploaded company docs with status

---

## C. Post-Login Empty State

### Current State

**The middleware redirect chain (src/middleware.ts, lines 110-158):**
1. If `user_profiles.setup_complete === false` --> redirect to `/setup`
2. If `user_profiles.has_completed_onboarding === false` --> redirect to `/onboard`
3. If path is `/` --> redirect to `/operator`

**For a brand-new company admin created via `/api/tenants/create`:**
- `setup_complete` is set to `true` (line 140 of tenants/create/route.ts)
- `has_completed_onboarding` is NOT set (column not written)
- Result: admin lands on `/operator` immediately

**For a user created via `/api/onboard/signup` (the /onboard conversational flow):**
- `setup_complete = true` and `has_completed_onboarding = true` (lines 136-140)
- Result: lands on `/operator`

**What does /operator show for a brand-new user?**

The operator page (`src/app/operator/page.tsx`) loads:
1. **Morning Briefing** (`MorningBriefing.tsx`) -- shows greeting + "needs you" cards. For a new company with no data, this will show empty state: "No briefing available / The morning briefing will appear when business data is ready" (from `empty-state-copy.ts`, line 128).
2. **Pill Bar** -- role-based category pills. These will all show empty drawers since there are no calls, invoices, publishers, etc.
3. **Search Bar** -- Milo chat. This works but has no business context to draw from.
4. **OnboardingChecklist** component -- imported on line 28 but unclear if it renders for company-level onboarding (it's tied to publisher/buyer onboarding runs).
5. **OperatorTour** component -- a guided tour (`src/components/OperatorTour.tsx`) that shows for users with `has_completed_walkthrough === false`. This IS the current "getting started" experience, but it's a tooltip-based product tour, not a platform setup wizard.

### What Data Sources Need to Be Connected?

1. **TrackDrive** -- the primary data source. Without it, calls/revenue/publishers/buyers are all empty
2. **Resend** -- for outreach email sending (already configured globally via env var)
3. **Bank transactions** -- optional, for reconciliation (has its own import API at `/api/bank-transactions/import`)

### Is There a Setup Flow That Walks Through Connecting Platforms?

**No.** The `/setup` page (`src/app/setup/page.tsx`) is a minimal 2-field form: name + role selection. It does not include platform connections, TrackDrive setup, or document uploads. It's designed for individual user profile setup, not company/platform configuration.

The OperatorTour is a product walkthrough (UI tour), not a platform setup wizard.

### What Needs to Be Built

1. **Post-company-creation setup wizard** -- a multi-step flow after `/onboard/company` success:
   - Step 1: Connect TrackDrive (enter API key, validate via `/me`)
   - Step 2: Upload company documents (MSA template, W-9, insurance cert)
   - Step 3: Company logo upload
   - Step 4: Invite team members
   - Step 5: Review & launch

2. **Empty state dashboard** -- when a new company has no data, the operator page should show a "Getting Started" card grid instead of empty pills:
   - "Connect TrackDrive" (with status indicator)
   - "Upload your MSA template" (with progress)
   - "Invite your team" (with invite count)
   - "Your first call sync will happen automatically once TrackDrive is connected"

3. **Progress tracking** -- a `tenant_setup_progress` table or JSONB column on `tenants` tracking which setup steps are complete

4. **Conditional redirect** -- middleware should check if tenant setup is complete, not just user profile setup. If tenant has no TD credentials, redirect to `/setup/connect` or similar.

---

## D. Logo Handling

### Current State

**TLP logo is hardcoded everywhere.** All references point to `/tlp-logo.png` (static file in `/public/`):

| File | Usage |
|------|-------|
| `src/app/verify/[token]/page.tsx:401` | Verification page header |
| `src/app/onboard/publisher/[token]/page.tsx:286,297,325,552` | Publisher onboarding (4 refs) |
| `src/app/onboard/buyer/[token]/page.tsx:273,284,312,554` | Buyer onboarding (4 refs) |
| `src/app/api/onboard/publisher/route.ts:183` | Publisher onboarding email (hardcoded full URL: `https://tlp.justmilo.app/tlp-logo.png`) |
| `src/app/api/onboard/buyer/route.ts:203` | Buyer onboarding email (same full URL) |
| `src/lib/tools/publish-contract-for-buyer-review.ts:236` | Contract review email |

**The company onboarding pages (`/onboard/company`, `/onboard`, `/join`) do NOT use the TLP logo.** They use the text wordmark "MILO" rendered as styled text (not an image). This is correct for multi-tenant -- Milo is the platform brand, TLP is a tenant brand.

**Physical file:** `/Users/markymark/Documents/GitHub/milo-for-ppc/public/tlp-logo.png` exists.

### Is There a Tenant-Specific Logo Field in the Database?

**No.** The `tenants` table (based on the `tenants/create` route, line 83) stores: `id`, `display_name`, `active`. No `logo_url` column.

The `vertical.config.ts` file is a static config with no per-tenant logo field. The `getTenantConfig()` function (line 67) ignores the tenantId and always returns the same config.

### What Should Happen if No Logo Is Uploaded?

Recommended fallback chain:
1. If `tenants.logo_url` is set --> use that image
2. If no logo --> show company name as styled text (already done in the company onboarding success screen: the "MILO" text wordmark pattern)
3. For emails (where images are preferred) --> use company name text fallback with the same styling

### What Needs to Be Built

1. **Logo column on tenants table** -- `ALTER TABLE tenants ADD COLUMN logo_url TEXT`
2. **Logo upload endpoint** -- `POST /api/tenants/logo` that uploads to Supabase Storage and sets `tenants.logo_url`
3. **Logo upload UI** -- in the post-company-creation setup wizard
4. **Tenant-aware logo component** -- replace hardcoded `/tlp-logo.png` references with a `<TenantLogo tenantId={...} />` component that falls back to text wordmark
5. **Email logo resolver** -- for Resend emails, resolve logo URL from tenant record instead of hardcoding `https://tlp.justmilo.app/tlp-logo.png`

---

## Recommended Build Order

### Phase 1: Foundation (blocks everything else)

1. **Tenant table expansion** -- add `logo_url`, `td_api_auth` (encrypted), `td_api_base_url`, `td_timezone`, `td_write_enabled`, `setup_progress` columns
2. **Tenant-aware TrackDrive client factory** -- `createTDClient(tenantId)` pattern replacing global env vars
3. **Post-login setup wizard** -- the multi-step "connect your platforms" flow

### Phase 2: Data Flow

4. **Tenant-aware sync cron** -- iterate tenants with valid TD credentials
5. **TrackDrive connection validation** -- API key test on input, periodic health check
6. **File upload infrastructure** -- Supabase Storage bucket + upload API

### Phase 3: Polish

7. **Logo upload** -- UI + storage + tenant-aware component
8. **Document upload in onboarding** -- MSA/IO/W-9 upload step
9. **Empty state dashboard** -- "Getting Started" card grid for new tenants
10. **TOS/privacy checkboxes** -- on company onboarding + team invite claim
11. **Legal disclaimer page** -- linked from checkbox

### Phase 4: Multi-tenant Hardening

12. **Row-level security** -- ensure all Supabase queries filter by tenant_id
13. **Tenant-scoped API routes** -- every API route must resolve the caller's tenant_id
14. **Credential encryption** -- encrypt TD API keys at rest
15. **Tenant admin panel** -- settings page for company admins to manage their setup

---

## Key File Index

| File | Lines | Purpose |
|------|-------|---------|
| `src/middleware.ts` | 168 | Auth + redirect logic (setup_complete, onboarding gates) |
| `src/lib/trackdrive.ts` | 848 | Single-tenant TrackDrive API client |
| `src/lib/td-costs.ts` | 25 | TrackDrive platform cost constants |
| `src/lib/vertical.config.ts` | 70 | Static tenant config (hardcoded TLP) |
| `src/lib/onboarding-client.ts` | 249 | Publisher/buyer onboarding flow definitions |
| `src/lib/onboarding-automation.ts` | 372 | Auto-advance onboarding on document events |
| `src/lib/onboarding-token.ts` | 240 | HMAC-signed onboarding tokens |
| `src/lib/publisher-onboarding.ts` | 396 | Publisher onboarding trigger (3 entry paths) |
| `src/lib/contract-analysis-prompt.ts` | 496 | 60+ rule contract analysis prompt |
| `src/lib/empty-state-copy.ts` | 133 | Per-surface empty state messages |
| `src/lib/tools/analyze-contract.ts` | 365 | AI contract clause analysis |
| `src/lib/tools/publish-contract-for-buyer-review.ts` | 290 | Contract review URL generation + email |
| `src/lib/tools/ingest-returned-contract.ts` | 335 | Redlined contract ingestion + diff |
| `src/app/onboard/company/page.tsx` | 639 | Company onboarding wizard (4 steps) |
| `src/app/onboard/page.tsx` | 1078 | Individual user onboarding (chat-style) |
| `src/app/setup/page.tsx` | 245 | User profile setup (name + role) |
| `src/app/join/[tenant]/[code]/page.tsx` | 478 | Team invite claim page |
| `src/app/operator/page.tsx` | ~200+ | Operator landing page |
| `src/app/api/tenants/create/route.ts` | 166 | Tenant + owner creation API |
| `src/app/api/tenants/check/route.ts` | 49 | Slug availability check |
| `src/app/api/tenants/invites/route.ts` | 211 | Invite CRUD (create/list/revoke) |
| `src/app/api/onboard/signup/route.ts` | 184 | User signup during onboarding |
| `src/app/api/webhooks/document-signed/route.ts` | 170 | Document signing webhook |
| `src/app/api/contract/documents/route.ts` | 110 | Contract documents fetch |
| `src/app/api/documents/entity/route.ts` | 112 | Unified document endpoint |
| `src/app/api/cron/sync/route.ts` | 60+ | 15-minute TrackDrive sync cron |

---

## Critical Gap Summary

| Gap | Severity | Blocking? |
|-----|----------|-----------|
| No TrackDrive credential collection UI | Critical | Yes -- no data flows without it |
| Single-tenant TrackDrive client (global env vars) | Critical | Yes -- multi-tenant impossible |
| No post-login setup wizard | High | Yes -- new admin sees empty operator page |
| No file upload for contracts/docs | High | Partially -- manual URL entry works |
| No tenant logo support | Medium | No -- text wordmark fallback exists |
| No TOS/privacy checkbox | Medium | No -- legal risk but not blocking |
| No tenant-scoped sync cron | Critical | Yes -- data sync only works for TLP |
| Hardcoded TLP logo in 12 locations | Medium | No -- only affects emails/onboarding pages |
| No encrypted credential storage | High | Security requirement before multi-tenant launch |
| vertical.config.ts returns same config for all tenants | High | Multi-tenant brand isolation impossible |

---

Report at: https://raw.githubusercontent.com/moscaleflow/conductor-mark-public/main/reports/POST-ONBOARDING-RESEARCH-2026-05-07.md
