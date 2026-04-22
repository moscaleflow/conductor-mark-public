# Coder-2 Report: milo-outreach @milo/blacklist Gate Integration

**Date**: 2026-04-20
**Lane**: Coder-2 (Migration)
**Task**: Wire milo-outreach to check @milo/blacklist before any outreach send
**Status**: Complete

---

## Integration commit

**SHA**: `0433957` — feat: add @milo/blacklist gate to email send path
- Added `@milo/blacklist` as workspace dep (`file:../milo-engine/packages/blacklist`)
- Created `lib/blacklist-gate.ts` — singleton client + `checkBlacklist()` wrapper
- Extended `GateState` in `lib/types.ts` with `blacklist_blocked` and `blacklist_reason`
- Gate 1 (blacklist) fires before env/tenant/manual gates in `lib/email-sender.ts`
- Added `vitest` as devDependency for testing

**SHA**: `da29226` — fix: move blacklist gate to position 1 + fix env var fallback
- Moved blacklist check before opt-out, draft, and env gates (compliance first)
- Fixed env var fallback chain: `BLACKLIST_SUPABASE_URL` → `MILO_ENGINE_SUPABASE_URL` → `NEXT_PUBLIC_SUPABASE_URL`
- Updated ARCHITECTURE.md Section 14 with new 4-gate ordering

## Tests commit

**SHA**: `f59a324` — test: blacklist gate screening — 9 cases
- Clean recipient passes through (2 tests)
- Exact name match blocks with high confidence
- Normalized name match blocks (strips LLC suffix)
- Alias match blocks with high confidence
- Email domain match returns medium (not blocked alone)
- Email domain + exact name match blocks
- Tenant isolation: entries from different tenant not matched
- Disabled gate bypass via empty entries

All 9 tests pass against `@milo/blacklist`'s pure `screenAgainst` function.

## Curl verification

### Blocked send

```
POST /api/admin/leads/send
Lead: Anne Calarco at Level Community Management (help@levelprop.com)
Blacklist entry: company_name="Level Community Management", email_domains=["levelprop.com"]

Response:
{
  "ok": true,
  "status": "blocked_blacklist",
  "reason": "Exact match on company name \"Level Community Management\"",
  "gate_state": {
    "env_allows": false,
    "tenant_allows": false,
    "send_mode": "manual",
    "confirmed": false,
    "blacklist_blocked": true,
    "blacklist_reason": "Exact match on company name \"Level Community Management\""
  }
}
```

Resend was NOT called. send_log entry created with status=blocked_blacklist.

### Passed send

```
POST /api/admin/leads/send
Lead: Terra West Management Services (customerservice@terrawest.com)
No blacklist entry for this company.

Response:
{
  "ok": false,
  "status": "failed",
  "error": "No email draft for opener."
}
```

Blacklist gate passed (no blocked_blacklist). Proceeded to draft check (no draft exists, so failed there). Resend NOT called (expected — no draft to send).

## Response-shape shims

None. `SendResult` and `GateState` interfaces extended in-place with optional fields.

## Feature gaps in @milo/blacklist

None encountered. The `check()` function and `screenAgainst()` pure function work exactly as documented. The `ScreenInput` interface accepts `company_name` and optional `email` — matches our needs.

## Gate ordering (final)

1. **Blacklist** — compliance first, runs even in dry-run mode
2. **Environment** — global dry-run kill switch
3. **Tenant flag** — per-tenant enable/disable
4. **Manual mode** — requires per-lead confirmation

## Files changed (milo-outreach)

| File | Change |
|------|--------|
| `package.json` | Added `@milo/blacklist`, `vitest` |
| `lib/blacklist-gate.ts` | New — singleton client + check wrapper |
| `lib/types.ts` | Extended `GateState` with blacklist fields |
| `lib/email-sender.ts` | Gate 1 (blacklist) before gates 2-4 |
| `lib/__tests__/blacklist-gate.test.ts` | New — 9 unit tests |
| `ARCHITECTURE.md` | Section 14 rewritten with 4-gate ordering |

---

*Coder-2 — Migration lane*
