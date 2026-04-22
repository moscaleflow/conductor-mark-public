# D137 Report: Wordmark + SMTP + MOP Audit

**Coder-4 Ops | 2026-04-22**

---

## Item 1: D127 — MILO Wordmark Href Fix

### Status: COMPLETE

### Audit results

| Location | Element | Href before | Action |
|---|---|---|---|
| `src/components/Topbar.tsx:1160` | `<Link>` wrapping MILO wordmark | `/dashboard-v2` (dead route) | Fixed → `/operator` |
| `src/app/page.tsx:307` | `<div>` plain text | N/A (not a link) | No fix needed |
| `src/app/onboard/page.tsx:514` | `<h1>` plain text | N/A (not a link) | No fix needed |

Only one location had a clickable MILO wordmark pointing at a dead route. Fixed to `/operator`.

### Commit

```
53193dd fix(topbar): MILO wordmark href /dashboard-v2 → /operator (D127)
```

### Pattern 16 verification

Before/after Playwright screenshots (desktop 1280x800 + mobile 375x812):
- Stitched output: `/Users/markymark/Desktop/d137-topbar-before-after.png`
- Raw captures: `/tmp/d137-topbar-{before,after}-{desktop,mobile}.png`

Before screenshot captured with wordmark → `/dashboard-v2`. After screenshot captured post-deploy with wordmark → `/operator`. Both show MILO wordmark rendered correctly at both breakpoints.

Console output from after-screenshot:
```
MILO wordmark href: /operator
```

---

## Item 2: D129 — Resend SMTP Swap

### Status: BLOCKED — domain not verified in Resend

### What was done

1. **Before snapshot** (SMTP config via Management API before changes):
   ```
   smtp_host: None
   smtp_port: None
   smtp_user: None
   smtp_pass: None
   smtp_admin_email: None
   smtp_sender_name: None
   ```
   Supabase was using its built-in mailer (no custom SMTP).

2. **Applied Resend SMTP config** via Management API PATCH:
   ```
   smtp_host: smtp.resend.com
   smtp_port: 587
   smtp_user: resend
   smtp_pass: [RESEND_API_KEY from Vercel env, re_Xd9...]
   smtp_admin_email: milo@justmilo.app
   smtp_sender_name: Milo
   ```
   API returned HTTP 200, config applied.

3. **Magic link test failed** — HTTP 500:
   ```json
   {"code":500,"error_code":"unexpected_failure","msg":"Error sending magic link email"}
   ```

4. **Root cause identified** — tested Resend REST API directly:
   ```json
   {"statusCode":403,"message":"The justmilo.app domain is not verified. Please, add and verify your domain on https://resend.com/domains","name":"validation_error"}
   ```
   The `justmilo.app` domain is **not verified** in Resend. SMTP and API both require domain verification.

5. **API key is restricted** — send-only, cannot list or manage domains:
   ```json
   {"statusCode":401,"message":"This API key is restricted to only send emails","name":"restricted_api_key"}
   ```

6. **SMTP reverted** to Supabase defaults (all null) to restore production auth emails. Verified built-in mailer works (HTTP 200 on magic link test).

### After snapshot (reverted):
```
smtp_host: None
smtp_port: None
smtp_user: None
smtp_pass: None
smtp_admin_email: None
smtp_sender_name: None
```

### Connectivity confirmed

SMTP ports are reachable — not a network issue:
```
smtp.resend.com:587 — reachable (STARTTLS)
smtp.resend.com:465 — reachable (TLS)
smtp.resend.com:25  — not reachable
```

### Blocker

To complete D129, the following must happen first:
1. Log into Resend dashboard (need a full-access API key or dashboard login)
2. Add `justmilo.app` as a verified sending domain
3. Add DNS records Resend provides (SPF, DKIM, DMARC) to justmilo.app's DNS (Cloudflare)
4. Wait for verification to complete
5. Re-apply SMTP config via Management API
6. Test magic link

Note: the existing `RESEND_API_KEY` is send-only restricted. Domain management requires either dashboard access or a full-access key.

Also note: `src/app/api/demo/capture/route.ts` already uses Resend SDK with `from: 'Milo <milo@justmilo.app>'`. If that route works in production, the domain may be verified under a different Resend account/API key than the one in `.env.production.local`. Worth checking.

---

## Item 3: MOP TrackDrive Audit

### Status: COMPLETE (partial — webhook timestamps unverifiable)

### Environment variables (via netlify-cli)

| Variable | Value | Status |
|---|---|---|
| `MOP_WRITES_FROZEN` | `false` | OK — writes enabled |
| `CONTRACT_WRITES_FROZEN` | `false` | OK — contract writes enabled |
| `TRACKDRIVE_API_KEY` | Set (present) | OK |
| `TRACKDRIVE_*` env vars | All present | OK |

### Deployment state

- **Site**: tlpmop.netlify.app
- **Published deploy**: `69e7f6fea008` from 2026-04-21
- **Deploy source**: Git (not ad-hoc `netlify deploy --prod`)

### Webhook timestamps

**NOT RETRIEVED** — Netlify CLI v24.11.3 does not support `functions:log` or `functions:invoke --log`. The Netlify dashboard's function logs or Netlify API `/sites/{id}/functions/{name}/log` would be needed. Neither is accessible from CLI alone.

Alternative: check Supabase for recent TrackDrive webhook data (timestamps on trackdrive-related table rows). This was not in the directive scope.

### Self-serve assessment

Netlify-cli successfully linked and read env vars without escalation to Mark. Token reuse from existing `~/.netlify` config worked. Guard 13 (escalation punt) was not triggered.

---

## Summary

| Item | Status | Commit |
|---|---|---|
| D127 wordmark fix | DONE | `53193dd` |
| D129 Resend SMTP | BLOCKED — justmilo.app not verified in Resend | N/A (reverted to safe state) |
| MOP TrackDrive audit | DONE (webhook timestamps unverifiable via CLI) | N/A (read-only audit) |

### Action items for Mark

1. **D129**: Verify `justmilo.app` domain in Resend dashboard (or confirm which Resend account the demo/capture route uses). Once verified, re-apply SMTP config — the Management API PATCH is proven to work.
2. **MOP webhooks**: If webhook recency matters, check Netlify dashboard function logs or query Supabase for recent TrackDrive-originated rows.
