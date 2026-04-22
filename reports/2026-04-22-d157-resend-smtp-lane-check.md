# D157 — Resend SMTP Status Resolution + Lane-Check Script

**Coder:** 4 (Ops)  
**Directive:** D157  
**Date:** 2026-04-22  
**Status:** COMPLETE

---

## Item 1: D129 Resend SMTP Status Resolution

### Investigation

**Q: Are the Resend keys different across repos?**  
A: No. Same key (`[REDACTED_KEY]_*`) on both milo-for-ppc and milo-outreach Vercel production environments.

**Q: Is justmilo.app verified for this key?**

```
$ curl -s -X POST "https://api.resend.com/emails" \
  -H "Authorization: Bearer [REDACTED_KEY]_*" \
  -d '{"from":"Milo <milo@justmilo.app>","to":"test@example.com","subject":"test","text":"test"}'

{"statusCode":403,"message":"The justmilo.app domain is not verified.","name":"validation_error"}
```

**Q: Can this key manage domains?**

```
$ curl -s -X GET "https://api.resend.com/domains" -H "Authorization: Bearer [REDACTED_KEY]_*"

{"statusCode":401,"message":"This API key is restricted to only send emails","name":"restricted_api_key"}
```

**Q: Is mysecretary.app verified?**

```
$ curl -s -X POST "https://api.resend.com/emails" \
  -H "Authorization: Bearer [REDACTED_KEY]_*" \
  -d '{"from":"test <hello@mysecretary.app>","to":"test@example.com","subject":"test","text":"test"}'

{"id":"ec0b6cb9-274b-4913-b6c6-970ab2955206"}
```

Yes — mysecretary.app sends successfully with the same key.

**Q: Does the demo capture route actually send emails?**  
No. `src/app/api/demo/capture/route.ts` (line 68-101) wraps Resend in a try/catch that swallows errors. The capture endpoint returns `{ success: true }` regardless of whether the email actually sent. Every demo capture email from `milo@justmilo.app` has silently failed since launch.

**Q: What about the milo-outreach demo-ppc tenant?**  
The `tenants` table has a `demo-ppc` tenant with `from_address: "Milo <noreply@justmilo.app>"`. This tenant's outreach emails also silently fail for the same reason.

### Resolution

**D129 status: DEFERRED — blocked on Mark manual action.**

The current Resend API key is send-restricted (cannot manage domains via API). justmilo.app must be verified in the Resend dashboard at https://resend.com/domains by someone with account access.

**Impact assessment:**
- **Auth magic-link emails:** Unaffected. Use Supabase's built-in SMTP (not Resend). Working correctly.
- **Demo capture thank-you emails:** Silently failing since launch. Low impact — the CRM lead capture works, only the follow-up email is lost.
- **milo-outreach demo-ppc tenant:** Would fail if triggered. Currently idle.
- **milo-outreach mysecretary tenant:** Working. mysecretary.app is verified.

**Mark action required:**
1. Log into Resend dashboard (https://resend.com/domains)
2. Add justmilo.app as a sending domain
3. Complete DNS verification (add TXT/CNAME records in Cloudflare for justmilo.app)
4. Once verified, demo capture emails will start working immediately (no code change needed)
5. Supabase Auth SMTP swap becomes possible (D129 original goal)

**Alternative (if verification is not worth the effort):** Close D129 permanently. Auth emails work fine via Supabase default SMTP. Demo capture can stay as lead-only (no email). The thank-you email is nice-to-have, not critical.

---

## Item 2: Lane-Check Script

### What was built

`scripts/lane-check.sh` — single-command lane status overview.

Output sections:
1. **conductor-mark last 5 commits** — what recently landed in coordination
2. **milo-for-ppc last 5 commits** — what recently landed in product
3. **Public mirror last 3 reports** — most recent synced report filenames
4. **Uncommitted work** — both repos, `git status --short`
5. **Lane states** — extracted from TASKS.md Coder-N status lines

### Usage

```bash
./scripts/lane-check.sh
# or from anywhere:
~/mark\ conduct/conductor-mark/scripts/lane-check.sh
```

Single screen output. Color-coded. No arguments needed. Requires `gh` CLI for mirror section (gracefully skips if missing).

### Tested output

Verified all 5 sections render correctly. Shows Coder-1 through Coder-4 status lines with headers. Mirror section shows actual report filenames (e.g., `2026-04-22-d154-backfill-mirror-state-refresh.md`) instead of raw sync commit SHAs.

---

## Commits

| SHA | Description |
|---|---|
| (this commit) | D157 report + lane-check.sh |

---

Report at: https://raw.githubusercontent.com/moscaleflow/conductor-mark-public/main/reports/2026-04-22-d157-resend-smtp-lane-check.md
