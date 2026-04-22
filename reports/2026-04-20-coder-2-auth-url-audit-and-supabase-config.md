# Auth URL Audit + Shared Supabase Config Update

**Date**: 2026-04-20
**Lane**: Coder-2 (Migration)
**Directive**: #44 (Tasks A + B)
**Status**: COMPLETE

---

## Task A: Auth URL Audit

### mysecretary

**Auth model**: Custom magic-link (NOT Supabase Auth). No `signInWithOtp()` or `signInWithOAuth()` calls found.

| File | URL pattern | Type |
|------|-------------|------|
| `api/auth/send-link/route.ts` | `${siteOrigin(req)}/api/auth/confirm?token=${token}` | Magic-link callback (custom) |
| `api/auth/confirm/route.ts` | `${siteOrigin(req)}/generate?welcome=true` | Post-confirm redirect (custom) |
| `lib/email.ts` | `https://mysecretary.app` (hardcoded) | Unsubscribe URL, login link, FROM_ADDRESS |
| `api/auth/unsubscribe/route.ts` | `https://mysecretary.app` (hardcoded) | Unsubscribe link |
| `lib/admin-auth.ts` | Cookie-based, HMAC with `mysecretary-admin-cookie-v1` | Admin auth (custom) |

`siteOrigin()` reads `NEXT_PUBLIC_SITE_ORIGIN` env var, falls back to `https://mysecretary.app`.

**Key finding**: mysecretary does NOT use Supabase Auth on the shared instance (tappyckcteqgryjniwjg). Its Supabase usage is data-only via @milo/crm. The shared Supabase auth config redirect URLs are exclusively for Milo-for-PPC magic-link flows.

### milo-outreach

| File | URL pattern | Type |
|------|-------------|------|
| `lib/email-sender.ts` | Dynamic per tenant domain | Unsubscribe URL |

No Supabase Auth usage. Data-only via @milo/blacklist and direct queries.

### Env files checked

| File | Relevant vars |
|------|--------------|
| `mysecretary/.env.local` | `NEXT_PUBLIC_SUPABASE_URL=nuequnfpnvbryvhuyuvz` (mysecretary-specific, not shared) |
| `mysecretary/.env.local` | `CRM_SUPABASE_URL=tappyckcteqgryjniwjg` (shared, data only) |
| `mysecretary/.env.local` | `MILO_OUTREACH_URL=https://milo-outreach-scale-flow.vercel.app` |

---

## Task B: Shared Supabase Auth Config Update

**Project**: tappyckcteqgryjniwjg (shared — milo-engine + @milo/crm + @milo/blacklist + milo-outreach)

### Pre-update config

- `site_url`: `https://tlp.justmilo.app`
- `uri_allow_list`: `https://tlp.justmilo.app/**,https://mysecretary.app/**,http://localhost:3000/**`

### Changes applied (PATCH via Management API)

| URL | Purpose | Action |
|-----|---------|--------|
| `https://tlp.justmilo.app/**` | Milo-for-PPC production (custom domain) | Kept |
| `https://mysecretary.app/**` | mysecretary (precautionary — doesn't use Supabase Auth here) | Kept |
| `http://localhost:3000/**` | Local dev | Kept |
| `https://milo-for-ppc.vercel.app/**` | Milo-for-PPC Vercel production URL | **Added** |
| `https://*-moscaleflow-milo-for-ppc.vercel.app/**` | Milo-for-PPC preview deploys | **Added** |
| `http://localhost:3001/**` | Secondary local dev port (mysecretary e2e / parallel dev) | **Added** |

### Post-update config

- `site_url`: `https://tlp.justmilo.app` (unchanged)
- `uri_allow_list`: `https://tlp.justmilo.app/**,https://mysecretary.app/**,http://localhost:3000/**,https://milo-for-ppc.vercel.app/**,https://*-moscaleflow-milo-for-ppc.vercel.app/**,http://localhost:3001/**`

### Snapshots

- Pre-update: `backups/supabase-auth-config-pre-update-2026-04-20.json`
- Post-update: `backups/supabase-auth-config-post-update-2026-04-20.json`

### Abort conditions checked

- No unexpected OAuth providers configured (all `external_*_enabled: false` except `external_email_enabled: true`)
- No third-party callback URLs found in audit
- No `signInWithOtp()` calls in mysecretary or milo-outreach — safe to update without breaking existing auth flows
- `site_url` already correct (`https://tlp.justmilo.app`)

---

*Coder-2 — Migration lane*
