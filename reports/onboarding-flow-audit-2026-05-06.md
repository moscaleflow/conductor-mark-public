# Onboarding Flow Audit -- Milo for PPC

**Date:** 2026-05-06
**Auditor:** Coder-3 (Research)
**Repo:** `/Users/markymark/Documents/GitHub/milo-for-ppc/`
**Scope:** Full read-only audit of all onboarding surfaces, API routes, auth integration, role system, and @milo/onboarding package

---

## Executive Summary

There are **four distinct onboarding paths** in the current system. They were built at different times for different audiences and share almost no code. The internal operator onboarding (Path 1 + Path 3) is complete and functional. The external publisher onboarding (Path 2) is polished UI but creates no auth account. The @milo/onboarding state machine (Path 4 via `onboarding-client.ts`) is fully built and wired but only tracks the 12-step publisher checklist -- it does not drive the UI flows. There is **zero multi-tenant infrastructure** -- no organizations table, no org_id, no tenant scoping on user_profiles, no team-invite mechanism.

---

## 1. Internal Operator Onboarding (Milo Chat Flow)

### Files
- `src/app/onboard/page.tsx` (~1,077 lines)
- `src/app/api/onboard/signup/route.ts` (~183 lines)
- `src/lib/focus-area-mapping.ts` (~53 lines)

### What Exists

A cinematic chat-style onboarding at `/onboard`. Steps:

1. **Intro** -- Milo introduces itself ("Hey. I'm Milo. I run operations here at The Lead Penguin.")
2. **Name** -- free text input
3. **Email** -- free text input
4. **Role** -- pill selection from 6 roles: Billing, Publisher QA, Outreach, Prospecting, DID Ops, Admin (Admin only shown to `mark@scaleflow.co` and `tiffani@scaleflow.co`)
5. **Focus** -- pill selection from role-specific options, plus custom "Something else" entry
6. **Setup** -- animated progress screen with role-specific tips while calling `/api/onboard/signup`
7. **Done** -- auto-redirect to `/` (which middleware routes to `/operator`)

### What Works (Happy Path)

1. User provides name, email, role, and focus areas
2. POST to `/api/onboard/signup` creates:
   - `auth.users` row via `supabaseAdmin.auth.admin.createUser()` with auto-generated temp password
   - `user_profiles` row via upsert with: `name`, `roles: [role]`, `is_admin: false`, `team: 'operations'`, `setup_complete: true`, `focus_areas`, `has_completed_onboarding: true`, `has_completed_walkthrough: false`
   - `team_members` row with: `display_name`, `team: 'operations'`, `roles: [role]`, `is_active: true`
   - `milo_conversations` row with the onboarding chat log
3. Client auto-signs in using `signInWithPassword` with the temp password
4. Redirect to `/` which middleware routes to `/operator`

### Re-onboarding Support

The signup route handles duplicate emails gracefully:
- If auth user exists but profile has `has_completed_onboarding === false` (or no profile exists), it re-onboards: resets password, upserts profile, returns new temp password
- If profile exists with `has_completed_onboarding === true`, returns 409

### What's Broken or Incomplete

- **Hardcoded to TLP:** Intro message says "I run operations here at The Lead Penguin" -- not configurable
- **ADMIN_EMAILS hardcoded:** `['mark@scaleflow.co', 'tiffani@scaleflow.co']` -- no dynamic admin detection
- **is_admin always false:** Signup route hardcodes `is_admin: false` for security -- admin must be granted via direct DB update. This is intentional but undocumented as a multi-tenant concern.
- **team always 'operations':** No team selection or organization association
- **No org_id anywhere:** user_profiles has no org_id field
- **Password-based auto-sign-in:** Creates users with temp passwords then immediately signs in. Works but conflicts with "magic-link auth only" policy (Decision 4). The temp password is never shown to the user -- future logins use magic links. This is a pragmatic workaround.
- **team_members table disconnect:** Creates a `team_members` row but it has no `user_id` FK. It's a display-only table, not linked to auth. `team_members.display_name` is the only connection to `user_profiles.name`.

### What's Missing for Multi-Tenant

- Organization creation step (which company are you joining?)
- Tenant-specific branding in the chat flow
- Invite code / organization join mechanism
- Organization-scoped role assignment (admin of Org A != admin of Org B)
- Configurable role list per tenant
- Configurable focus areas per tenant

---

## 2. External Publisher Onboarding (Token-Based)

### Files
- `src/app/onboard/publisher/[token]/page.tsx` (~595 lines)
- `src/app/api/onboard/publisher/route.ts` (~201 lines)
- `src/lib/publisher-onboarding.ts` (~396 lines)

### What Exists

A cinematic external-facing page at `/onboard/publisher/[token]`. The token is a base64-encoded JSON with optional `lead_id`, `company`, `contact`, `rep`, and `verticals`. Phases:

1. **Intro** -- TLP logo, personalized greeting ("Hey, [contact]")
2. **Confirm** -- shows known info from token (company, contact, verticals)
3. **Verticals** -- pill selection from 14 defaults + custom entry
4. **Call Types** -- True Inbound, Warm Transfer, Blind Transfer, Live Transfer. "True Inbound" triggers a confirmation dialog ("Hold on. True inbound?")
5. **Creatives** -- Yes/No for whether they have creatives ready (file upload optional)
6. **Contact** -- Name, Company, Email, Phone + optional creative upload
7. **Done** -- shows 5-step "what happens next" timeline
8. **Form** -- alternate "Just give me the form" path that skips cinematic

### What Works (Happy Path)

1. POST to `/api/onboard/publisher` calls `handleInboundPublisherSubmission()`:
   - Runs blacklist screening via `screenEntity()`
   - Creates/finds contact in `contacts` table
   - Creates/finds `prospect_pipeline` record at stage `onboarding` via `findOrCreatePipelineEntry()` dedup gateway
   - Creates `action_items` row ("New publisher onboarding: [name] -- review submission")
2. PUT to `/api/onboard/publisher` handles creative file upload to `onboarding-creatives` storage bucket
3. Sends confirmation email via Resend with TLP-branded HTML

### What's Broken or Incomplete

- **No auth account created:** This flow creates a pipeline record but NO user account. The publisher has no way to log into Milo. The "done" screen says "Our ops team is on it" -- fully manual handoff.
- **No connection to @milo/onboarding state machine:** The 12-step checklist is not started by this flow. That only happens when an operator triggers via `/api/publisher-onboarding` or the Milo chat tool.
- **Token has no expiration or HMAC:** Anyone who can guess/intercept the base64 token can submit. The token is just `btoa(JSON.stringify({...}))` -- no signature.
- **Hardcoded TLP branding:** Logo, colors (#e87fbf), "The Lead Penguin" text, domain
- **Email from address hardcoded:** `The Lead Penguin <support@theleadpenguin.com>`
- **Dedup relies on company name:** `findOrCreatePipelineEntry` matches on entity_name (ilike). No reliable unique identifier.

### What's Missing for Multi-Tenant

- Token should include `org_id` or `tenant_id`
- Branding (logo, colors, company name) from org config
- Confirmation email template per tenant
- Storage bucket path should include org scope
- Pipeline creation should scope to org

---

## 3. Welcome Flow (Team Invite for TLP's 11 Operators)

### Files
- `src/app/welcome/[slug]/page.tsx` (~208 lines, RSC)
- `src/app/welcome/[slug]/WelcomeExperience.tsx` (~419 lines, client)
- `src/app/api/welcome/[slug]/claim/route.ts` (~281 lines)
- `src/lib/team-roster.ts` (~80+ lines)

### What Exists

A cinematic one-time-use claim flow at `/welcome/[slug]`. Each of the 11 TLP operators has a unique slug (`/welcome/cam`, `/welcome/fab`, etc.). The flow:

1. RSC wrapper validates slug against `TEAM_ROSTER` whitelist
2. Checks `user_profiles.has_claimed_welcome` -- already claimed shows "This link has been used"
3. Client component types welcome_line char-by-char, then recognition_line
4. Shows email + password form
5. POST to `/api/welcome/[slug]/claim`:
   - Validates slug, email, password (min 8 chars, blocks common passwords)
   - Updates existing auth.users row with new email + password via `supabaseAdmin.auth.admin.updateUserById`
   - Updates user_profiles: `has_claimed_welcome: true`, `has_completed_onboarding: true`, `setup_complete: true`, `focus_areas` from roster, `roles`, `is_admin`
   - Plants Supabase session cookies on the response
6. BootingUpSequence animation, then redirect to `/operator`

### What Works

This is the most polished onboarding path. It works end-to-end for its intended use case: 11 pre-provisioned team members claiming their accounts with personalized cinematic sequences.

### What's Broken or Incomplete

- **Fixed roster of 11 people:** `TEAM_ROSTER` is a hardcoded constant. Adding a 12th person requires a code change.
- **Password-based auth:** Uses email + password, not magic links. Contradicts Decision 4 (magic-link auth only), though this is the initial account claim -- subsequent logins can use magic links.
- **Pre-provisioned auth rows required:** The claim route calls `updateUserById`, not `createUser`. Someone (Mark or an agent) must manually create the auth.users row first.
- **No team-invite mechanism:** This is a one-time setup tool, not a repeatable invite system.
- **Name matching fragile:** Profile lookup uses `ilike(name, roster.display_name)` -- if anyone changes their display name in the DB, the claim breaks.

### What's Missing for Multi-Tenant

- This entire flow is TLP-specific and not reusable. For multi-tenant, you'd need:
  - An invite system that creates the auth row + slug dynamically
  - Org-scoped welcome flows with tenant branding
  - Admin-triggered invites (not hardcoded roster)

---

## 4. @milo/onboarding Package (State Machine)

### Files
- `vendor/onboarding/` (local dist in milo-for-ppc)
- `milo-engine/packages/onboarding/` (canonical source)
- `src/lib/onboarding-client.ts` (~193 lines, consumer wrapper)

### What Exists

A fully-built DAG-based onboarding state machine. Architecture:

**Three Supabase tables:**
- `onboarding_flows` -- flow definitions with JSONB steps array
- `onboarding_runs` -- per-counterparty execution with frozen flow snapshot
- `onboarding_steps` -- per-step state within a run

**State Machine:**
- Run lifecycle: `active` -> `completed` / `cancelled` / `expired`
- Step lifecycle: `blocked` -> `pending` -> `active` -> `completed` / `failed` / `skipped`
- DAG execution: multiple steps can be active simultaneously when dependencies satisfied
- Lazy expiration: checked on read, no cron required

**TLP Publisher Flow (12 steps):**
1. add_to_mop (required, no deps)
2. msa_sent (required, depends: add_to_mop)
3. msa_signed (required, depends: msa_sent)
4. w9_sent (optional, depends: msa_signed)
5. w9_signed (optional, depends: w9_sent)
6. creatives_submitted (required, depends: msa_signed)
7. creatives_approved (required, depends: creatives_submitted)
8. did_provisioned (required, depends: creatives_approved)
9. config_validated (required, depends: did_provisioned)
10. io_sent (required, depends: config_validated)
11. io_signed (required, depends: io_sent)
12. go_live (required, depends: io_signed)

**Multi-tenant built in:** All queries scoped by `tenant_id`. Currently hardcoded to `'tlp'` in `onboarding-client.ts`.

### How It's Wired

`triggerPublisherOnboarding()` in `publisher-onboarding.ts` calls `startPublisherOnboarding()` in `onboarding-client.ts`, which:
1. Ensures the `tlp-publisher-onboarding` flow exists (creates if missing)
2. Calls `client.startRun()` with the flow ID and counterparty ID
3. Returns the run_id

This is called from:
- `/api/publisher-onboarding` route (Path B: trigger_onboarding mode)
- `setup-new-publisher` Milo chat tool

### What's Broken or Incomplete

- **No UI for the checklist:** The state machine tracks steps but there's no operator-facing UI to view/advance steps. Operators can't see which step a publisher is on.
- **Steps are never auto-advanced:** The state machine supports it but no automation calls `advanceStep()`. The 12-step flow is write-only -- runs are created but never progress without manual API calls.
- **No webhook handler:** The `webhook_url` field exists on runs but nothing receives the events.
- **No connection to the publisher onboarding UI:** The external publisher form (`/onboard/publisher/[token]`) does NOT start an onboarding run. The run is only started when an operator explicitly triggers it.
- **Stall detection unused:** `findStalledSteps()` exists but nothing calls it on a schedule.
- **Buyer onboarding flow undefined:** Only publisher flow exists. The `setup-new-buyer` tool does NOT create an onboarding run -- it has a TODO stub for Milo Engine sync.

### What's Missing for Multi-Tenant

The package itself is multi-tenant-ready (`tenant_id` on all tables). What's missing:
- Flow definitions per tenant (currently only `tlp-publisher-onboarding`)
- Buyer flow definition
- Operator UI to view/manage runs
- Webhook integration for auto-advancing steps

---

## 5. Auth Integration

### Files
- `src/app/login/page.tsx` (~407 lines)
- `src/app/auth/callback/page.tsx` (~110 lines)
- `src/app/api/auth/set-session/route.ts` (~56 lines)
- `src/middleware.ts` (~165 lines)
- `src/app/setup/page.tsx` (~244 lines)

### Login Page

Three modes:
- **Magic link** (default): `signInWithOtp` -> email with link -> `/auth/callback`
- **Password**: `signInWithPassword` (for users who set passwords via /welcome)
- **Forgot password**: `resetPasswordForEmail`

Link at bottom: "New here? Start onboarding" -> `/onboard`

### Auth Callback

Handles both implicit flow (hash fragment with `access_token`) and PKCE flow (query param `code`). Posts to `/api/auth/set-session` which exchanges tokens/code for a session and sets cookies. Hard 5-second timeout before force-redirecting to `/`.

### Middleware Auth Logic

Public routes (no auth required):
- `/api/*`, `/contract-review/*`, `/c/*`, `/auth/callback`, `/welcome/*`, `/ask`, `/pipeline-board`, `/contract-queue`, `/proof`, `/marketing/*`, `/samples/*`, `/test-fixtures/*`, `/login`, `/onboard`, `/onboard/publisher/*`, `/verify/*`

For authenticated users:
1. Checks `user_profiles.setup_complete` -- if false, redirects to `/setup`
2. Checks `user_profiles.has_completed_onboarding` -- if false, redirects to `/onboard`
3. Root `/` redirects to `/operator` (all users, admin or not)
4. Supports admin impersonation via `milo_impersonating` cookie

For unauthenticated users:
- Root `/` redirects to `/onboard` (new users get cinematic flow)
- All other paths redirect to `/login` (returning users)

### Setup Page

Fallback for users who have an auth account but `setup_complete: false`. Collects name and roles, updates user_profiles directly via Supabase client (not admin). Role options: Operations, Sales, Finance, Management, Admin, Publisher Relations, Buyer Relations.

### What's Broken or Incomplete

- **No org/tenant awareness in auth:** The middleware has no concept of which organization a user belongs to. There's no `org_id` in user_profiles, no organization lookup during auth.
- **Multiple auth mechanisms coexist:** Magic link (login), password (welcome/claim), temp password (onboard/signup). All work but the coexistence is messy for Decision 4 compliance.
- **pipeline-board and contract-queue are public:** These routes bypass auth in middleware. Intentional but worth noting for multi-tenant security.
- **Impersonation is admin-only but not org-scoped:** Any admin can impersonate any user across all data (since there's only one org today).

### What's Missing for Multi-Tenant

- Organization lookup during auth (which org does this email belong to?)
- Org-scoped session (current session has no org context)
- Org-scoped middleware routing (different orgs might have different public routes)
- Invite-based account creation (org admin invites new members)
- Org-level admin vs global admin distinction

---

## 6. Role & Permission System

### Files
- `src/app/api/admin/users/route.ts` (~47 lines)
- `src/app/api/team-members/route.ts` (~26 lines)
- `src/lib/team-roster.ts` (source of truth for TLP's 11 operators)

### user_profiles Table Fields (Observed)

| Field | Type | Notes |
|-------|------|-------|
| user_id | UUID | FK to auth.users |
| name | TEXT | display name |
| roles | TEXT[] | array of role strings |
| is_admin | BOOLEAN | admin flag |
| is_test_user | BOOLEAN | filters test accounts |
| team | TEXT | always 'operations' |
| setup_complete | BOOLEAN | gates /setup redirect |
| has_completed_onboarding | BOOLEAN | gates /onboard redirect |
| has_completed_walkthrough | BOOLEAN | gates guided tour |
| has_claimed_welcome | BOOLEAN | one-time welcome claim |
| has_completed_operator_tour | BOOLEAN | operator tour flag |
| focus_areas | TEXT[] | dashboard block preferences |

### team_members Table

Used by `getCanonicalTeamRoster()` as the canonical identity registry. No `user_id` FK -- connected to user_profiles only by display_name matching. Fields: `display_name`, `team`, `roles`, `is_active`.

### Admin Users Route

`GET /api/admin/users` returns all non-test user_profiles for the impersonation dropdown. Filters: `is_test_user = false`, `name IS NOT NULL`. Admin-only via `getEffectiveUser()`.

### Team Members Route

`GET /api/team-members` returns the canonical team roster from `getCanonicalTeamRoster()` (reads user_profiles, falls back to hardcoded TEAM_ROSTER).

### What's Broken or Incomplete

- **No organization model:** Roles are global, not org-scoped
- **team_members disconnected from auth:** No user_id on team_members
- **Role values inconsistent:** Onboarding collects `billing`, `publisher_qa`, `outreach`, `prospecting`, `did_ops`, `admin`. Setup page collects `Operations`, `Sales`, `Finance`, `Management`, `Admin`, `Publisher Relations`, `Buyer Relations`. Welcome flow uses `outreach`, `publisher_qa`, `qc`, `billing`, `billing_support`, `ping_validation`, `call_monitoring`, `campaign_ops`, `admin`. Three different role vocabularies.
- **No permission enforcement:** Roles stored but no middleware or API checks beyond `is_admin`. There's no "billing users can only see billing data" enforcement.

### What's Missing for Multi-Tenant

- Organizations table
- user_profiles.org_id
- Org-scoped role assignment
- Permission enforcement layer
- Org admin vs global admin
- Team-invite mechanism (admin of Org A invites user to Org A)

---

## 7. Setup Tools (Milo Chat)

### Files
- `src/lib/tools/setup-new-publisher.ts` (~373 lines)
- `src/lib/tools/setup-new-buyer.ts` (~327 lines)

### setup_new_publisher

End-to-end publisher setup via Milo chat:
1. Confirms signed IO/MSA in contract_documents
2. Calls `triggerPublisherOnboarding()` (creates contact, pipeline record, runs blacklist, starts @milo/onboarding run)
3. Resolves campaigns from campaigns table
4. Creates traffic source in TrackDrive
5. Inserts campaign_routes rows
6. Runs /api/campaigns/auto-detect
7. Syncs to CRM (crm_counterparties)

### setup_new_buyer

Similar but for buyers:
1. Runs blacklist screening
2. Finds/creates signed IO
3. Creates contact + pipeline entry
4. Creates buyer in TrackDrive
5. Resolves campaigns
6. Runs auto-detect
7. CRM sync is **STUBBED** ("Mark to enter buyer in Milo manually")

### Relationship to Onboarding UI

These tools are **completely independent** from the onboarding UI flows. A publisher can be set up entirely through Milo chat without ever visiting `/onboard` or `/onboard/publisher/[token]`. The chat tools are the more complete path because they:
- Create TrackDrive entities
- Start the @milo/onboarding run
- Create campaign routes
- Sync to CRM

The UI flows do none of that.

---

## 8. Document Generation (Supporting)

### Files
- `src/app/api/onboard/generate-docs/route.ts` (~295 lines)
- `src/app/api/onboard/doc-status/route.ts` (~58 lines)

### generate-docs

Proxies document generation to Milo Engine's `/api/bot/documents/generate`. Supports MSA, W9, W8BEN, W8BENE, and IO. For IO documents, pulls commercial terms from `agreed_terms` table and builds merge-field payload. Guards: IO requires confirmed agreed_terms with non-empty qualifiers.

### doc-status

Checks document signing status. Tries Milo Engine first for live status, falls back to local `contract_documents` table.

These are **supporting infrastructure** for the onboarding checklist steps but are not directly wired into any onboarding UI flow.

---

## 9. Gap Analysis

### Flows That Exist But Aren't Wired

| Gap | Description |
|-----|-------------|
| Publisher form -> onboarding run | `/onboard/publisher/[token]` creates a pipeline record but never starts an @milo/onboarding run. The checklist state machine only activates via operator trigger or Milo chat. |
| Onboarding run -> step advancement | Runs are created but never progress. No automation calls `advanceStep()` when MSA is signed, W9 is returned, etc. |
| Stall detection -> alerts | `findStalledSteps()` exists but nothing calls it on a schedule. Stalled onboardings go unnoticed. |
| Buyer onboarding flow | No flow definition exists for buyers. `setup-new-buyer` skips the @milo/onboarding package entirely. |
| Operator onboarding -> team invite | No mechanism for an operator to invite another team member. New users must either know the `/onboard` URL or be given a `/welcome/[slug]` link. |

### Security Concerns

| Issue | Severity | Notes |
|-------|----------|-------|
| Publisher token has no HMAC | Medium | Token is plain base64 JSON. Guessable, interceptable, no expiry. |
| pipeline-board is public | Low | Middleware bypasses auth for this route. May expose pipeline data. |
| contract-queue is public | Low | Same concern. |
| Temp passwords in transit | Low | `/api/onboard/signup` returns temp password in JSON response body. Used only for immediate auto-sign-in, but logged in server if any logging middleware exists. |

---

## 10. Prioritized Build/Fix List

### Pre-Multi-Tenant (Ship Now, Improves TLP's Experience)

1. **Wire publisher onboarding form to @milo/onboarding run** -- When a publisher submits via `/onboard/publisher/[token]`, automatically start an onboarding run. Currently this is a dead end that creates a pipeline record and an action item but no tracked checklist.

2. **Build operator-facing onboarding checklist UI** -- There's a full state machine tracking 12 steps but operators have no way to see it. Add a panel in the EntityDetailPanel or a dedicated route showing run state, step status, and allowing manual advancement.

3. **Wire document signing webhooks to advanceStep()** -- When an MSA or IO is signed (webhook from Milo Engine), automatically advance the corresponding onboarding step. The state machine supports this but the webhook handler doesn't exist.

4. **Add stall detection cron** -- Call `findStalledSteps()` on a schedule (daily at 6 AM Mountain) and create action_items for stalled steps. Infrastructure exists but nobody calls it.

5. **Define and wire buyer onboarding flow** -- Create a `tlp-buyer-onboarding` flow definition in onboarding-client.ts (parallel to publisher flow). Wire `setup-new-buyer` to start an onboarding run.

6. **Unify role vocabulary** -- Three different role string sets across onboard/setup/welcome. Consolidate to one canonical set.

7. **Sign publisher onboarding tokens** -- Add HMAC signature and expiration to the base64 tokens used for `/onboard/publisher/[token]`. Prevent token guessing/replay.

8. **Connect team_members to user_profiles** -- Add `user_id` FK to team_members or deprecate team_members entirely in favor of user_profiles as the sole identity table.

### Requires Multi-Tenant (Blocked Until Organizations Table Lands)

9. **Create organizations table** -- `organizations(id, name, slug, branding, config)`. Every tenant gets a row.

10. **Add org_id to user_profiles** -- FK from user_profiles to organizations. All queries must scope by org.

11. **Org-scoped onboarding flow** -- `/onboard` reads org branding from config. Chat messages, role options, and focus areas come from org config, not hardcoded constants.

12. **Team-invite system** -- Org admins generate invite links (or enter emails) to onboard new team members into their org. Replaces the hardcoded welcome/slug flow.

13. **Org-scoped middleware** -- Auth middleware resolves org from user session. Routes and data queries scope by org.

14. **Org-scoped publisher onboarding** -- Token includes org_id. Pipeline records, action items, confirmation emails all tenant-scoped.

15. **Org-level admin vs global admin** -- `is_admin` becomes org-scoped. Global admin is a separate concept for ScaleFlow operations.

16. **Org-scoped @milo/onboarding flows** -- Each org defines its own flow steps. The package already supports this via `tenant_id` -- just needs org-aware flow management UI.

---

Report at: https://raw.githubusercontent.com/moscaleflow/conductor-mark-public/main/reports/onboarding-flow-audit-2026-05-06.md
