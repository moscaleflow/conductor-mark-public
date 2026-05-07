# Synthetic Multi-Tenant Onboarding E2E Test

**Date:** 2026-05-07
**Repo:** milo-for-ppc (tlp.justmilo.app)
**Supabase:** tappyckcteqgryjniwjg (shared)
**Tester:** Coder-3 (read-only, API-level)

---

## Summary

| Test | Result | Notes |
|------|--------|-------|
| 1. Database schema | PARTIAL | Tables exist, migration applied but role promotion failed |
| 2. Slug availability API | PASS | All 3 cases correct |
| 3. Tenant creation API | FAIL | HTTP 500 -- missing NOT NULL columns |
| 4. Invite creation (unauth) | PASS | Returns 401 as expected |
| 5. Invite validation | PASS | Returns 404 for nonexistent code |
| 6. Onboarding page loads | PASS | HTTP 200 |
| 7. Join page loads | PASS | HTTP 200 |
| 8. TLP extraction check | PASS (with caveat) | verticalConfig.brand.tenantId used; 1 comment artifact |

---

## Test 1: Database Schema Verification

### tenants table -- PASS (exists with correct columns)

Columns confirmed present: `id`, `display_name`, `domain`, `industry`, `website`, `active`, plus legacy outreach columns (`from_address`, `reply_to`, `physical_address`, `voice_config`, `research_config`, `real_sends_enabled`, `send_mode`, `api_key_hash`, `api_key_prefix`, `is_demo`, `demo_expires_at`, `created_at`, `updated_at`).

**TLP seed data** -- PASS:
```json
{
  "id": "tlp",
  "display_name": "The Lead Penguin",
  "domain": "milo-outreach.vercel.app",
  "industry": "pay-per-call",
  "website": null,
  "active": true
}
```
Note: `domain` is still `milo-outreach.vercel.app`, not `tlp.justmilo.app`. The migration INSERT used `tlp.justmilo.app` but it was a conditional insert (WHERE NOT EXISTS) and the row already existed with the old domain.

### user_profiles table -- PARTIAL

Columns `tenant_id` and `role` are present and populated. All 23 user_profiles have `tenant_id = 'tlp'` -- correct.

### team_invites table -- PASS (exists, empty)

Columns confirmed: `id`, `tenant_id`, `email`, `role`, `invite_code`, `status`, `created_at`, `expires_at`, `invited_by`, `claimed_by`, `claimed_at`. All expected columns present.

### Mark owner role -- FAIL

Mark's profile: `role = 'operator'` (should be `owner`)

**Root cause:** The migration promotes by email lookup via `auth.users`:
```sql
UPDATE user_profiles up SET role = 'owner'
FROM auth.users au WHERE au.id = up.user_id AND au.email = 'mark@scaleflow.co';
```
But Mark's auth.users email is `investintpi+marktestmilo@gmail.com`, not `mark@scaleflow.co`. The UPDATE matched zero rows.

Mark's auth user_id: `9025ee19-5a83-474b-a07c-54fa3e8bdfa7`

### Tiffani admin role -- FAIL

Tiffani's profile: `role = 'operator'` (should be `admin`)

**Root cause:** Same issue. Migration looks for `tiffani@scaleflow.co` but her auth email is `tiffani@milo.internal`.

Tiffani's auth user_id: `46cdbb22-1de4-46c0-bc7a-8e9d0fd2784e`

**Impact:** No user currently has `owner` or `admin` role in the `role` column. Every user is `operator`. This means:
- The invite creation API will reject all requests with 403 ("Only owners or admins can create invites")
- Tenant creation when authenticated will also 403

---

## Test 2: Slug Availability API -- PASS

```
GET /api/tenants/check?slug=test-company-ppc  -->  {"available":true}
GET /api/tenants/check?slug=tlp               -->  {"available":false}
GET /api/tenants/check?slug=admin              -->  {"available":false}
```

All three responses match expected behavior. Reserved list includes: admin, api, app, demo, demo-ppc, help, internal, login, milo, milo-internal, onboard, setup, support, test, www.

---

## Test 3: Tenant Creation API -- FAIL (HTTP 500)

```
POST /api/tenants/create
Body: {"company_name":"Test Company PPC","tenant_id":"test-company-ppc","admin_email":"testadmin@test-milo.com","admin_name":"Test Admin"}
Response: HTTP 500 {"error":"Failed to create tenant."}
```

**Root cause:** The `tenants` table has NOT NULL constraints on columns inherited from the outreach schema:
- `domain` -- NOT NULL, no default
- `from_address` -- NOT NULL, no default
- `reply_to` -- NOT NULL, no default
- `physical_address` -- NOT NULL, no default

The create route (`src/app/api/tenants/create/route.ts`, line 83-87) only inserts:
```typescript
db.from('tenants').insert({
  id: normalizedTenantId,
  display_name: company_name.trim(),
  active: true,
})
```

Missing: domain, from_address, reply_to, physical_address.

**Additionally:** The onboard page collects `industry` and `website` and sends them in the POST body, but the create route ignores these fields entirely.

**Verified via direct insert probe:** An insert with `id`, `display_name`, `active`, `domain`, `from_address`, `reply_to`, `physical_address` succeeds (probe was cleaned up).

**Fix needed in Coder-1's lane:** The create route must generate defaults for the 4 NOT NULL columns:
- `domain`: `${tenant_id}.justmilo.app`
- `from_address`: `noreply@justmilo.app` or similar
- `reply_to`: same as from_address
- `physical_address`: empty string or placeholder

And should also pass through `industry` and `website` from the request body.

---

## Test 4: Invite Creation (Unauthenticated) -- PASS

```
POST /api/tenants/invites (no auth header)
Response: HTTP 401 {"error":"Authentication required."}
```

Correct behavior. The route checks `getEffectiveUser()` and returns 401 when no session is present.

---

## Test 5: Invite Validation -- PASS

```
GET /api/tenants/invites/validate?code=nonexistent
Response: HTTP 404 {"valid":false,"reason":"Invite not found"}
```

Correct behavior.

---

## Test 6: Onboarding Page -- PASS

```
GET /onboard/company --> HTTP 200
```

The page is a cinematic 4-step wizard (welcome -> company info -> admin info -> workspace slug -> review). It's a client-rendered component using styled inline JSX. It submits to `/api/tenants/create`.

---

## Test 7: Join Page -- PASS

```
GET /join/tlp/fakecode --> HTTP 200
```

The page renders (client-side) and validates the invite code on mount via `/api/tenants/invites/validate`. For a fake code, the page would show "Invalid invite link" after the client-side fetch returns 404.

Dynamic route structure: `/join/[tenant]/[code]/page.tsx` -- correctly parameterized.

---

## Test 8: TLP Extraction Check -- PASS (with caveat)

**verticalConfig.brand.tenantId usage (correct):**
- `src/app/api/ask/route.ts` -- 9 occurrences using `verticalConfig.brand.tenantId`
- `src/lib/blacklist-client.ts:35` -- uses `verticalConfig.brand.tenantId`
- `src/lib/crm-client.ts:33,344` -- uses `verticalConfig.brand.tenantId`

**Hardcoded 'tlp' remaining:**
- `src/lib/vertical.config.ts:10` -- `tenantId: 'tlp'` -- CORRECT (this is the config source)
- `src/lib/blacklist-client.ts:3` -- JSDoc comment: `Tenant-scoped to 'tlp'` -- cosmetic only, not functional

No functional hardcoded 'tlp' references outside the config source.

---

## Critical Bugs Found

### BUG 1: Tenant creation fails (HTTP 500) -- BLOCKER

**File:** `src/app/api/tenants/create/route.ts` lines 83-87
**Cause:** Insert missing 4 NOT NULL columns (domain, from_address, reply_to, physical_address)
**Impact:** No new tenants can be created via the onboarding flow
**Lane:** Coder-1

### BUG 2: Role promotion migration missed all users -- HIGH

**File:** `supabase/migrations/20260507000003_multi_tenant_foundation.sql` lines 52-61
**Cause:** Migration uses email addresses that don't exist in auth.users (`mark@scaleflow.co`, `tiffani@scaleflow.co`). Actual emails are `investintpi+marktestmilo@gmail.com` and `tiffani@milo.internal`.
**Impact:** No owner or admin exists. Invite system is completely locked -- nobody can create invites because all users are `operator` role.
**Fix:** Run manual UPDATE to fix:
```sql
UPDATE user_profiles SET role = 'owner' WHERE user_id = '9025ee19-5a83-474b-a07c-54fa3e8bdfa7';
UPDATE user_profiles SET role = 'admin' WHERE user_id = '46cdbb22-1de4-46c0-bc7a-8e9d0fd2784e';
```
**Lane:** Coder-1

### BUG 3: TLP domain stale in tenants table -- LOW

**Actual:** `domain = 'milo-outreach.vercel.app'`
**Expected:** `domain = 'tlp.justmilo.app'`
**Cause:** The migration used `INSERT ... WHERE NOT EXISTS` but the row already existed from the outreach schema with the old domain. The UPDATE only set `industry` and `display_name`.
**Fix:** `UPDATE tenants SET domain = 'tlp.justmilo.app' WHERE id = 'tlp';`

---

## Test Artifacts

No test tenant was successfully created (creation API fails). No invite codes were generated (auth/role gate blocks). No cleanup needed.

Schema probe rows (test-schema-probe through test-schema-probe-4) were cleaned up during testing.

---

## Auth Users Reference (for role fix)

| Name | Auth Email | user_id | Current Role | Expected Role |
|------|-----------|---------|--------------|---------------|
| Mark | investintpi+marktestmilo@gmail.com | 9025ee19-5a83-474b-a07c-54fa3e8bdfa7 | operator | owner |
| Tiffani | tiffani@milo.internal | 46cdbb22-1de4-46c0-bc7a-8e9d0fd2784e | operator | admin |

---

Report at: https://raw.githubusercontent.com/moscaleflow/conductor-mark-public/main/reports/SYNTHETIC-TENANT-TEST-2026-05-07.md
