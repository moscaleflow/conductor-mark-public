# ANTHROPIC_API_KEY Drift Fix — Systematic Resolution

**Coder:** Coder-4 (Ops)
**Date:** 2026-04-24
**Decision:** 105
**Status:** COMPLETE

## Problem

ANTHROPIC_API_KEY has gone missing or stale at least 4 times:
- D68: key stale in Vercel production
- D69: masked behind 500 errors during Opus 4.7 deprecation
- D102: missing from .env.local, key had trailing `\n` in .env.production.local
- D102-CRM: Coder-1 blocked on CRM context integration

Root cause: no canonical source documented, no verification tooling, key copied with embedded garbage characters.

## Audit findings

| Location | Before | After |
|---|---|---|
| milo-for-ppc .env.local | 110 chars (2 extra: embedded `\n`) | 108 chars (clean) |
| milo-for-ppc .env.production.local | 110 chars (same issue) | 108 chars (clean) |
| milo-engine .env.local | MISSING | 108 chars (added) |
| milo-outreach .env.local | MISSING | 108 chars (added) |
| Vercel Production | Valid (canonical, 2d old) | Unchanged |
| Vercel Development | MISSING | Added |
| Vercel Preview | MISSING | N/A (no feature branches) |
| mysecretary | No Anthropic usage | OUT OF SCOPE |
| Netlify MOP | No Anthropic usage | OUT OF SCOPE |

## Fixes shipped

### A. Key restored everywhere
- Pulled canonical key from Vercel production via `vercel env pull`
- Verified valid via Anthropic API ping (HTTP 200, model claude-haiku-4-5)
- Replaced corrupted local copies (removed embedded `\n`)
- Added to milo-engine and milo-outreach .env.local
- Added to Vercel Development environment

### B. verify-env.sh
- Location: `milo-for-ppc/scripts/verify-env.sh`
- Checks: ANTHROPIC_API_KEY, SUPABASE keys (presence, length, no trailing whitespace)
- Live pings: Anthropic API + Supabase REST API
- Exit code: 0 = all clear, 1 = fix required
- Usage: `cd ~/Documents/GitHub/milo-for-ppc && ./scripts/verify-env.sh`

### C. CREDENTIALS-MAP.md updated
- Anthropic section rewritten with canonical source, downstream copies, key format
- mysecretary + MOP explicitly marked OUT OF SCOPE
- Known failure mode documented (trailing `\n`, embedded quotes)
- Rotation cadence added

### D. Drift Guard Pattern 22
- "Env var drift across .env.local / Vercel / Netlify"
- Rule: run verify-env.sh before declaring build issues from env vars
- Rule: canonical source is Vercel production, all others sync from there

### E. Gitignore verified
All 4 repos (milo-for-ppc, milo-engine, milo-outreach, mysecretary) have .env.local and .env*.local in .gitignore. No risk of accidental commit.

## verify-env.sh clean output

```
ANTHROPIC_API_KEY present in .env.local (108 chars)
NEXT_PUBLIC_SUPABASE_URL present (40 chars)
NEXT_PUBLIC_SUPABASE_ANON_KEY present (208 chars)
SUPABASE_SERVICE_ROLE_KEY present (219 chars)
Anthropic API key is valid (HTTP 200)
Supabase service role key is valid (HTTP 200)
All checks passed.
```

## Coder-2 unblocked

Coder-1's CRM context integration (D102) should now work — ANTHROPIC_API_KEY is present and valid in milo-for-ppc .env.local (108 chars, HTTP 200 verified).
