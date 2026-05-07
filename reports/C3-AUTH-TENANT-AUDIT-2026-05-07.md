# Auth, Tenant, and User Infrastructure Audit

**Coder-3 Research | 2026-05-07**
**Target:** milo-for-ppc, milo-engine, milo-outreach, mysecretary

---

## Executive Summary

Milo-for-ppc has a solid single-tenant auth system built on Supabase Auth. Multi-tenancy exists only in the @milo/* engine packages and milo-outreach, not in the product layer. The infrastructure for company admin onboarding and team invites will need: a `tenant_id` column on `user_profiles`, a proper invite-link flow, and role escalation controls. The good news is the auth patterns (`getEffectiveUser`, impersonation, RLS) are well-built and can be extended rather than replaced.

---

## 1. Auth Infrastructure

### 1.1 Core Auth Files

| File | Purpose | Status |
|------|---------|--------|
| `src/lib/auth-server.ts` | Server-side user resolution with impersonation | Implemented, production |
| `src/lib/auth-context.ts` | Client-side React context for user/profile | Implemented, production |
| `src/components/AuthProvider.tsx` | React provider wrapping Supabase auth listener | Implemented, production |
| `src/lib/supabase-browser.ts` | Browser Supabase client (singleton, cookie-based) | Implemented, production |
| `src/lib/supabase-admin.ts` | Service-role client (server-only, bypasses RLS) | Implemented, production |
| `src/middleware.ts` | Auth gate + routing + impersonation-aware redirects | Implemented, production |

### 1.2 `getEffectiveUser(req)` -- The Central Auth Function

**Location:** `src/lib/auth-server.ts`

This is the core auth function used by all API routes. It:

1. Creates a server-side Supabase client from request cookies
2. Calls `supabase.auth.getUser()` to get the authenticated user
3. Looks up the real user's `user_profiles` row via `supabaseAdmin` (service role, bypasses RLS)
4. Checks for an `is_admin` flag on the profile (never trusted from session)
5. If admin + `milo_impersonating` cookie is set, loads the target profile
6. Returns an `EffectiveUser` object with both real and effective identities

**Return shape:**
```typescript
interface EffectiveUser {
  realUserId: string | null;        // always the human behind the keyboard
  effectiveUserId: string | null;   // target user when impersonating
  effectiveIsAdmin: boolean;        // false when impersonating a non-admin
  realIsAdmin: boolean;             // true admin flag from profile
  isImpersonating: boolean;
  targetName: string | null;
  realName: string | null;
}
```

**`resolveRequestUser(req)`** is a thin wrapper that returns a flattened version. The old query-parameter auth bypass (`?userId=X&isAdmin=true`) was removed per D-2026-04-25-008.

### 1.3 Magic Link Auth

**Status: Fully implemented, with password fallback**

The `/login` page (`src/app/login/page.tsx`) supports three modes:
- **Magic link** (default): `supabase.auth.signInWithOtp()` with email redirect to `/auth/callback`
- **Password**: `supabase.auth.signInWithPassword()`
- **Forgot password**: `supabase.auth.resetPasswordForEmail()`

**Note:** Decision 4 (LOCKED) says "magic-link only, no passwords, no OAuth." However, the production `/login` page supports all three modes. This is a drift from the locked decision -- likely intentional for the TLP operator team who prefer passwords, but not documented.

### 1.4 Auth Callback Flow

**Location:** `src/app/auth/callback/page.tsx`

Handles both PKCE and implicit Supabase auth flows:
1. Captures `#access_token` hash or `?code` query param at module import time (before React renders)
2. POSTs to `/api/auth/set-session` to plant server-side cookies
3. Redirects to `/` on success (5-second hard timeout)

**`/api/auth/set-session`** (`src/app/api/auth/set-session/route.ts`):
- Accepts either `{ access_token, refresh_token }` or `{ code }`
- Creates server Supabase client, calls `setSession()` or `exchangeCodeForSession()`
- Cookies are set on the response

### 1.5 Middleware Auth Gate

**Location:** `src/middleware.ts`

**Multi-host routing:**
- `justmilo.co` -> 301 to `justmilo.app`
- `justmilo.app` (marketing): public pages only, no auth
- `tlp.justmilo.app` (product): full auth gate

**Auth flow:**
1. Skip auth for: `/api/*`, `/login`, `/onboard`, `/auth/callback`, `/ask`, `/welcome/*`, `/verify/*`, `/contract-review/*`, `/marketing/*`, etc.
2. For all other routes: check Supabase session via `getUser()`
3. No session -> redirect to `/onboard` (root) or `/login` (other routes)
4. Has session but `setup_complete=false` -> redirect to `/setup`
5. Has session but `has_completed_onboarding=false` -> redirect to `/onboard`
6. Admin with `milo_impersonating` cookie -> resolve effective admin status for routing
7. Root `/` -> redirect to `/operator`

### 1.6 Impersonation System

**Fully implemented with audit trail:**

| Endpoint | Purpose |
|----------|---------|
| `POST /api/admin/impersonate` | Start impersonation (admin-only, non-admin targets only) |
| `POST /api/admin/stop-impersonate` | End impersonation (safe for anyone to call) |
| `GET /api/admin/impersonation-state` | Check current impersonation status |

**Security controls:**
- Only `is_admin=true` users can impersonate
- Admin-to-admin impersonation blocked
- `milo_impersonating` cookie: httpOnly, secure, sameSite=strict, 8h expiry
- All start/stop events logged to `admin_impersonation_log` table with actor, target, IP, user-agent

---

## 2. User Tables

### 2.1 `user_profiles` Table

**Created:** Migration `20260327100001_user_profiles.sql`

**Current schema (consolidated from all migrations):**

```sql
CREATE TABLE user_profiles (
  id                          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id                     UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE UNIQUE,
  name                        TEXT,
  roles                       TEXT[] DEFAULT '{}',
  team                        TEXT DEFAULT 'operations',
  is_admin                    BOOLEAN DEFAULT false,
  setup_complete              BOOLEAN DEFAULT false,
  custom_pills                JSONB DEFAULT '[]',
  focus_areas                 TEXT[] DEFAULT '{}',
  hidden_pills                JSONB DEFAULT '[]',
  is_test_user                BOOLEAN NOT NULL DEFAULT false,
  has_completed_operator_tour BOOLEAN DEFAULT false,
  has_claimed_welcome         BOOLEAN DEFAULT false,
  has_completed_onboarding    BOOLEAN,
  has_completed_walkthrough   BOOLEAN NOT NULL DEFAULT false,
  pill_config                 JSONB DEFAULT '{}',
  created_at                  TIMESTAMPTZ DEFAULT NOW(),
  updated_at                  TIMESTAMPTZ DEFAULT NOW()
);
```

**Key observations:**
- NO `tenant_id` column. This is single-tenant.
- NO `email` column (email lives in `auth.users` only)
- `is_admin` is boolean, not a role level -- binary admin/non-admin
- `roles` is a text array, not a FK to a roles table
- `team` is always `'operations'` -- hardcoded, not tenant-scoped
- `focus_areas` drives which operator dashboard pills are shown

**RLS policies:**
- Users can read/update/insert their own profile (`auth.uid() = user_id`)
- Admins can read all profiles (via `public.is_admin()` SECURITY DEFINER function)
- Service role bypasses RLS
- NO write policy for admins on other profiles (admin writes go through `supabaseAdmin`)

**Auto-create trigger:**
```sql
CREATE TRIGGER on_auth_user_created
  AFTER INSERT ON auth.users
  FOR EACH ROW EXECUTE FUNCTION handle_new_user();
```
Inserts a bare `user_profiles` row with only `user_id` set (no name, no roles).

### 2.2 `team_members` Table

**Created:** Migration `20260329000001_action_items_task_system.sql`

```sql
CREATE TABLE team_members (
  id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  display_name TEXT NOT NULL UNIQUE,
  team         TEXT DEFAULT 'operations',
  roles        TEXT[] DEFAULT '{}',
  is_active    BOOLEAN DEFAULT true,
  created_at   TIMESTAMPTZ DEFAULT NOW()
);
```

**Key observations:**
- NO `user_id` column -- not linked to `auth.users` or `user_profiles`
- NO `tenant_id` column
- NO `email` column
- Seeded with 9 hardcoded names (Mark, Fab, Malvin, Jen, Vee, Cam, Lee, Jackie, Tiffani)
- Kate added via separate migration `20260402000001_add_kate_team_member.sql`
- Used by `action_items.assigned_to` FK
- RLS: anon read all, service role full access

**This table is a display-name registry for task assignment, NOT a real user management table.**

### 2.3 `admin_impersonation_log` Table

```sql
CREATE TABLE admin_impersonation_log (
  id             UUID PRIMARY KEY,
  actor_user_id  UUID NOT NULL,
  actor_name     TEXT,
  target_user_id UUID NOT NULL,
  target_name    TEXT,
  action         TEXT NOT NULL CHECK (action IN ('start', 'stop')),
  started_at     TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  ip_address     TEXT,
  user_agent     TEXT,
  created_at     TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### 2.4 `auth.users` (Supabase managed)

Standard Supabase Auth table. Users created via:
1. **Welcome flow** (`/api/welcome/[slug]/claim`): admin pre-creates placeholder auth users, operators claim via slug + email + password
2. **Onboard flow** (`/api/onboard/signup`): creates auth user with auto-generated password, then auto-signs in
3. **Direct signup**: not exposed in UI but possible via Supabase Auth APIs

---

## 3. User Onboarding Flows

### 3.1 Team Member Welcome Flow (Primary, existing team)

**Flow:** `/welcome/[slug]` -> claim route -> set email + password -> plant session -> `/operator`

1. Mark pre-creates `auth.users` rows for each team member (test-*@scaleflow.co placeholder emails)
2. The `handle_new_user()` trigger auto-creates a `user_profiles` row
3. Team member visits `/welcome/cam` (or their slug)
4. Submits real email + password
5. `POST /api/welcome/[slug]/claim`:
   - Validates slug against `TEAM_ROSTER` hardcoded list (11 entries)
   - Updates `auth.users` row with real email + password
   - Updates `user_profiles` with name, roles, focus_areas, is_admin, setup_complete=true, has_completed_onboarding=true
   - Plants browser session
6. Redirects to `/operator`

**This is a one-time-use link system.** `has_claimed_welcome` prevents re-use.

### 3.2 New User Onboard Flow (Cinematic)

**Flow:** `/onboard` -> conversational chat -> signup -> auto-sign-in -> `/operator`

1. User arrives at `/onboard` (or gets redirected there by middleware)
2. Cinematic chat experience collects: name, email, role, focus areas
3. `POST /api/onboard/signup`:
   - Creates `auth.users` with auto-generated password (`Milo-<uuid>!`)
   - Upserts `user_profiles` with name, roles, focus_areas, setup_complete=true
   - Inserts `team_members` row (display name only, no user_id link)
   - Returns `tempPassword` for client-side auto-sign-in
4. Client signs in with `signInWithPassword(email, tempPassword)`
5. Future logins use magic link

**Security note:** `is_admin` is hardcoded to `false` in signup. Admin must be granted via direct DB update.

### 3.3 Setup Page (Legacy)

**Flow:** `/setup` -> name + role selection -> update profile -> `/operator`

Used when `setup_complete=false`. Simple form: enter name, pick roles, update `user_profiles`.

---

## 4. Tenant Infrastructure

### 4.1 The `tenants` Table

**Defined in:** `milo-outreach/supabase/migrations/00001_initial_schema.sql`
**Database:** Shared Supabase (`tappyckcteqgryjniwjg`)

```sql
CREATE TABLE tenants (
  id                  TEXT PRIMARY KEY,          -- e.g. 'mysecretary', 'tlp'
  display_name        TEXT NOT NULL,
  domain              TEXT NOT NULL,
  from_address        TEXT NOT NULL,
  reply_to            TEXT NOT NULL,
  physical_address    TEXT NOT NULL,
  voice_config        JSONB DEFAULT '{}',
  research_config     JSONB DEFAULT '{}',
  real_sends_enabled  BOOLEAN DEFAULT false,
  send_mode           TEXT DEFAULT 'manual',
  active              BOOLEAN DEFAULT true,
  is_demo             BOOLEAN DEFAULT false,     -- added 20260420
  demo_expires_at     TIMESTAMPTZ,               -- added 20260420
  api_key             TEXT UNIQUE,                -- added 00002
  api_key_hash        TEXT,                       -- added 00004
  api_key_prefix      TEXT,                       -- added 00004
  created_at          TIMESTAMPTZ DEFAULT NOW(),
  updated_at          TIMESTAMPTZ DEFAULT NOW()
);
```

**Key observations:**
- `id` is TEXT slug, not UUID (Decision 24, LOCKED)
- Table lives in the shared Supabase, not in milo-for-ppc's migrations
- No FK from `user_profiles` to `tenants`
- Seeded tenants: `mysecretary` (seed data in migration), `tlp` (created at runtime)
- Demo tenants supported: `demo-ppc`, `milo-internal` via `src/lib/demo/ensure-tenant.ts`

### 4.2 Where `tenant_id` Is Used

**In @milo/* engine packages (every table):**
- `crm_counterparties.tenant_id TEXT NOT NULL REFERENCES tenants(id)`
- `crm_leads.tenant_id TEXT NOT NULL REFERENCES tenants(id)`
- `crm_contacts.tenant_id TEXT NOT NULL REFERENCES tenants(id)`
- `crm_activities.tenant_id TEXT NOT NULL REFERENCES tenants(id)`
- `onboarding_flows.tenant_id TEXT NOT NULL`
- `onboarding_runs.tenant_id TEXT NOT NULL`
- `onboarding_steps.tenant_id TEXT NOT NULL`
- All `@milo/contract-*` tables have `tenant_id NOT NULL`
- `blacklist_entries.tenant_id TEXT NOT NULL`

**In milo-outreach (every table):**
- `leads.tenant_id TEXT NOT NULL REFERENCES tenants(id)`
- `outreach_templates.tenant_id`
- `send_log.tenant_id`
- `research_runs.tenant_id`
- `opt_outs.tenant_id`

**In milo-for-ppc operational tables (recent additions, DEFAULT 'tlp'):**
- `entity_aliases.tenant_id TEXT NOT NULL DEFAULT 'tlp'`
- `outreach_log.tenant_id TEXT DEFAULT 'tlp'`
- `verification_tokens.tenant_id TEXT DEFAULT 'tlp'`
- `vet_results.tenant_id TEXT NOT NULL`

**NOT tenant-scoped (milo-for-ppc foundation tables):**
- `user_profiles` -- NO tenant_id
- `team_members` -- NO tenant_id
- `contacts` -- NO tenant_id
- `prospect_pipeline` -- NO tenant_id (but uses tenant_id in some queries)
- `call_records` -- NO tenant_id
- `invoices` -- NO tenant_id
- `campaigns` -- NO tenant_id
- `campaign_routes` -- NO tenant_id
- `alerts` -- NO tenant_id
- `contract_documents` -- NO tenant_id
- All other foundation tables from `20260311000001_foundation_schema.sql`

### 4.3 Tenant Context Resolution

**In milo-for-ppc:** Hardcoded as `'tlp'` everywhere. No middleware resolution, no subdomain parsing, no runtime tenant detection.

Examples from source:
```typescript
// src/lib/crm-client.ts
cached = createCrmClient({ supabaseUrl: url, supabaseServiceKey: key, tenantId: 'tlp' });

// src/lib/onboarding-client.ts
cached = createOnboardingClient({ supabase, tenantId: 'tlp', ... });

// src/app/api/ask/route.ts
sb.from('crm_counterparties').select('company').eq('tenant_id', 'tlp')
```

**In milo-engine packages:** Tenant is a required config parameter passed at client creation time:
```typescript
// @milo/crm
export function createCrmClient(opts: { supabaseUrl, supabaseServiceKey, tenantId }): CrmClient
// All queries scope by client.tenantId automatically
```

**In milo-outreach:** Per-tenant API key auth (Decision 20). Tenant resolved from Bearer token against `tenants.api_key`.

**Onboarding RLS:** Uses `current_setting('app.tenant_id', true)` for RLS policies -- proper PostgreSQL session variable approach, but this is only in the engine, not in milo-for-ppc.

### 4.4 `vertical.config.ts`

**Location:** `src/lib/vertical.config.ts`

Centralized brand config (Decision from 2026-04-18 locked decisions):
```typescript
export const verticalConfig = {
  brand: {
    name: 'Milo for PPC',
    domain: 'justmilo.app',
    productHost: 'tlp.justmilo.app',
    tenantId: 'tlp',  // <-- hardcoded tenant slug
    colors: { primary: '#7c3aed', ... },
  },
  company: {
    name: 'TLP Compliance',
    legalName: 'The Lead Penguin LLC',
    ownerName: 'Tiffani Kinkennon',
    ...
  },
  ...
};
```

---

## 5. RLS Policy Summary

### milo-for-ppc foundation tables (single-tenant, wide-open)
```sql
-- 15 foundation tables: anon_read_all policy allowing SELECT for any user
CREATE POLICY "anon_read_all" ON contacts FOR SELECT USING (true);
-- ... same for all foundation tables
```
**No write policies for anon.** All writes go through `supabaseAdmin` (service role, bypasses RLS).

### user_profiles (role-based)
- Users: read/update/insert own profile
- Admins: read all profiles (via `is_admin()` SECURITY DEFINER function)
- No RLS-based tenant scoping

### @milo/* engine tables (tenant-scoped)
```sql
-- Onboarding tables use session variable
CREATE POLICY onboarding_flows_tenant ON onboarding_flows
  USING (tenant_id = current_setting('app.tenant_id', true));
```
But: milo-for-ppc uses `supabaseAdmin` (service role) which bypasses these policies entirely. RLS tenant scoping in the engine is a defense-in-depth layer, not the primary isolation mechanism.

---

## 6. What Exists vs. What's Decided

### Implemented and Production

| Feature | Status |
|---------|--------|
| Supabase Auth (magic link + password) | Production |
| `user_profiles` table with roles, focus_areas | Production |
| `team_members` display-name registry | Production (limited utility) |
| `getEffectiveUser()` server auth resolution | Production |
| Admin impersonation with audit log | Production |
| Middleware auth gate with setup/onboard redirects | Production |
| Welcome slug claim flow (one-time team onboarding) | Production |
| Cinematic `/onboard` new user flow | Production |
| `AuthProvider` client-side context | Production |
| `tenants` table in shared Supabase | Production (outreach/engine only) |
| Per-tenant API keys for milo-outreach | Production |
| `tenant_id` on all @milo/* engine tables | Production |

### Decided but NOT Implemented

| Decision | What's Missing |
|----------|----------------|
| D4: Magic-link only | Password auth exists on `/login` -- drift from decision |
| D24: tenant_id is TEXT slug | Correct in engine/outreach, but milo-for-ppc's own tables have NO tenant_id |
| All multi-tenant @milo/* tables | milo-for-ppc hardcodes `'tlp'` everywhere -- no dynamic tenant resolution |

### NOT Decided / NOT Implemented

| Feature | Status |
|---------|--------|
| `tenant_id` on `user_profiles` | Does not exist |
| `tenant_id` on `team_members` | Does not exist |
| `tenant_id` on foundation tables (call_records, invoices, etc.) | Does not exist |
| Role levels beyond admin/non-admin | Not implemented (is_admin is binary) |
| Team invite link infrastructure | Does not exist |
| Company admin panel | Does not exist |
| Tenant creation in milo-for-ppc | Only exists in `src/lib/demo/ensure-tenant.ts` for demo tenants |
| Subdomain-based tenant resolution | Middleware has multi-host routing but only for marketing vs. product |
| RLS policies scoped by tenant | Only in engine tables, bypassed by service role anyway |

---

## 7. Cross-Repo Patterns

### mysecretary

mysecretary has NO auth infrastructure of its own. It's a vanilla React app (htm + esm.sh, no build step) that doesn't use Supabase Auth. Its outreach is handled via milo-outreach as `tenant_id='mysecretary'`. No patterns to borrow for auth/team management.

### milo-engine

The engine packages have a clean tenant pattern:
- Every `create*Client()` takes `tenantId: string` as required config
- Every query scopes by `tenant_id`
- RLS policies on onboarding tables use `current_setting('app.tenant_id', true)`
- This is the pattern to follow for tenant scoping

### milo-outreach

Has the most complete tenant system:
- `tenants` table with config (domain, from_address, send_mode, etc.)
- Per-tenant API key auth (hashed with bcrypt)
- All data tables scoped by `tenant_id REFERENCES tenants(id)`
- This is the authoritative tenant registry

---

## 8. Foundation for Company Admin Onboarding + Team Invites

### What You Can Build On

1. **Auth is solid.** `getEffectiveUser()`, the impersonation system, and the middleware auth gate are production-ready and extensible. No need to replace any of this.

2. **`user_profiles` is the right user table.** It already has: `is_admin`, `roles[]`, `focus_areas[]`, `team`, `setup_complete`, `has_completed_onboarding`. Just needs `tenant_id`.

3. **`tenants` table exists in shared Supabase.** The `tlp` tenant likely already exists (created at runtime by `ensure-tenant.ts` or engine client initialization).

4. **@milo/* client pattern is the right approach.** Pass `tenantId` at client creation, scope all queries automatically.

5. **Onboard flow is extensible.** The cinematic `/onboard` page collects name, email, role, focus areas. Adding a "company" step or "invite token" verification is straightforward.

### What Needs to Be Built

1. **Add `tenant_id TEXT REFERENCES tenants(id)` to `user_profiles`:**
   - Default `'tlp'` for existing rows
   - All 11 current users get `tenant_id='tlp'`
   - New signups must specify tenant (from invite link or company selection)

2. **Link `team_members` to `user_profiles`:**
   - Add `user_id UUID REFERENCES auth.users(id)` to `team_members`
   - Add `tenant_id` to `team_members`
   - Or: deprecate `team_members` entirely and use `user_profiles` as the single source of truth (recommended -- `team_members` is barely used, only for `action_items.assigned_to` FK)

3. **Invite link system:**
   - New table: `team_invites` (id, tenant_id, email, role, invited_by, token, expires_at, claimed_at)
   - New route: `POST /api/admin/invite` (admin creates invite)
   - New route: `GET /api/invite/[token]` (validates invite, shows onboard page)
   - Modify `/api/onboard/signup` to accept invite token and auto-assign tenant_id + role

4. **Role escalation:**
   - Current: binary `is_admin` flag
   - Needed: at minimum, three levels: `owner`, `admin`, `operator`
   - `owner` = company admin who can invite/remove team members
   - `admin` = current admin behavior (impersonate, see all profiles)
   - `operator` = current non-admin behavior
   - Could use `roles[]` array or a separate `role_level` column

5. **Tenant creation for new companies:**
   - `POST /api/admin/create-tenant` or a self-serve company signup flow
   - Insert into `tenants` table with company branding
   - Auto-assign the creating user as `owner`

6. **Tenant-aware middleware:**
   - Resolve tenant from subdomain (`acme.justmilo.app`) or from `user_profiles.tenant_id` after auth
   - Set `app.tenant_id` PostgreSQL session variable for RLS

---

## Self-Attestation Checklist

- [x] Auth infrastructure: `auth-server.ts`, `getEffectiveUser()`, `resolveRequestUser()` fully characterized
- [x] Magic-link auth: implemented, plus password fallback (drift from Decision 4 noted)
- [x] `/login` page: three modes (magic link, password, forgot), dark theme
- [x] Auth middleware: full characterization of multi-host routing + auth gate + onboard redirects
- [x] Impersonation: fully implemented with audit trail, admin-only, non-admin targets
- [x] User tables: `user_profiles`, `team_members`, `auth.users` schemas documented
- [x] `tenants` table: schema documented, lives in milo-outreach schema on shared Supabase
- [x] `tenant_id` usage: comprehensive sweep across all repos
- [x] RLS: all policies documented, wide-open foundation tables noted
- [x] Tenant creation: only `ensure-tenant.ts` for demo tenants
- [x] `vertical.config.ts`: documented with hardcoded `tenantId: 'tlp'`
- [x] Team member management: `team_members` has no `user_id` link, no invite system
- [x] Role system: binary `is_admin` only, no graduated levels
- [x] Cross-repo patterns: milo-engine tenant client pattern, milo-outreach tenant table, mysecretary has no auth
- [x] DECISION-LOG: 10+ tenant/auth decisions cross-referenced (D4, D15, D16, D20, D21, D22, D24, D25, D34, D47, D50, D51)

## Known Gaps / No-Change Items

- No UI surfaces touched -- screenshot N/A (research-only directive)
- No production code modified (read-only audit)
- Decision 4 drift (password auth exists despite "magic-link only" lock) flagged for Mark's review
- `team_members` table appears vestigial -- recommend deprecation in favor of `user_profiles`
- Foundation tables (15 tables from `20260311000001`) have `anon_read_all` policies -- security risk for multi-tenant, will need tenant-scoped RLS when multi-tenancy lands
- No known gaps in the audit itself -- all 6 checklist areas covered

---

Report at: https://raw.githubusercontent.com/moscaleflow/conductor-mark-public/main/reports/C3-AUTH-TENANT-AUDIT-2026-05-07.md
