# Legacy Blacklist Table Deprecation Audit

**Coder:** Coder-4 (Ops)
**Date:** 2026-04-24
**Decision:** 114
**Status:** COMPLETE (audit only — no code changes, no table drops)

## Summary

Legacy `blacklist` table has **0 rows**. `blacklist_entries` (@milo/blacklist) has **343 rows**. Data migration already complete. Two consumers of legacy table remain in milo-for-ppc — both effectively dead code. Drop is safe after consumer wireup.

---

## A. Legacy `blacklist` consumers

### Consumer 1: /ask blacklist pre-check (DEAD CODE)
- **File:** `src/app/api/ask/route.ts:136-158`
- **Function:** `blacklistPreCheck(input: string)`
- **Intent:** Check user input against blacklisted company names before sending to Claude. Prepends `[BLACKLIST HIT: ...]` to user message. Sets `blacklistHit = true` for red 2px left-border visual tell.
- **Table:** `from('blacklist')` — reads `company_name, reason, blacklisted_at`
- **Scope:** Only runs when `pill === 'vet'` (route.ts:297)
- **Status:** **NON-FUNCTIONAL.** Queries 0 rows. Always returns null. The red left-border visual tell (Decision 88) never fires from this code path. Every /ask Vet query wastes a Supabase query on an empty table.

### Consumer 2: /api/blacklist CRUD endpoint (DATA LEAKAGE RISK)
- **File:** `src/app/api/blacklist/route.ts` (full file, 86 lines)
- **Intent:** Admin API for managing blacklist entries — GET (list all), POST (add new), DELETE (remove by ID)
- **Table:** `from('blacklist')` — all three methods
- **Status:** **DATA LEAKAGE.** If an admin POSTs a new blacklist entry, it writes to legacy `blacklist` (which nothing reads effectively) instead of `blacklist_entries` (which @milo/blacklist reads). New entries are invisible to `screenEntity()` callers. GET returns 0 entries even though 343 exist in the real table.

### Consumer 3: Migration script (REFERENCE ONLY)
- **File:** `milo-engine/packages/blacklist/scripts/migrate-blacklist-data.ts:336`
- **Intent:** One-time migration from `blacklist` → `blacklist_entries`
- **Status:** Already ran successfully. 0 rows remain in legacy. Script preserved for reference.

### Consumers already on @milo/blacklist (CORRECT — no action needed)
All `screenEntity()` callers use `src/lib/blacklist-client.ts` which reads `blacklist_entries`:
- `src/app/api/capture/route.ts:330`
- `src/app/api/leads/import/route.ts:271`
- `src/app/api/blacklist/screen/route.ts:16`
- `src/app/api/entities/[name]/route.ts:286`
- `src/app/api/publisher-onboarding/route.ts:44`
- `src/lib/publisher-onboarding.ts:93, :276`
- `src/lib/milo-queries.ts:1349, :1965`
- `src/lib/tools/setup-new-buyer.ts:199`

### Repos with zero legacy blacklist references
- **milo-outreach:** 0 hits
- **mysecretary:** 0 hits

---

## B. Data comparison

| Table | Rows | Status |
|---|---|---|
| `blacklist` (legacy) | 0 | Empty — all data migrated |
| `blacklist_entries` (@milo/blacklist) | 343 | Authoritative (tenant='tlp') |

**Rows in `blacklist` NOT in `blacklist_entries`:** 0 (table is empty)
**Rows in `blacklist_entries` NOT in `blacklist`:** 343 (all entries)

**No data migration needed.** Legacy table is completely empty.

---

## C. Migration plan

**No data migration required.** Legacy table has 0 rows.

Schema comparison for reference:

| Legacy `blacklist` column | `blacklist_entries` equivalent |
|---|---|
| company_name | company_name |
| reason | reason |
| severity | severity |
| blacklisted_by | added_by |
| company_aliases | company_aliases |
| contact_names | contact_names |
| email_domains | email_domains |
| ip_addresses | ip_addresses |
| linkedin_urls | linkedin_urls |
| registration_states | (not present — TLP-specific field) |
| evidence | evidence |
| blacklisted_at | created_at |
| — | tenant_id (added in @milo/blacklist, always 'tlp') |

---

## D. Drop sequence

**Prerequisites (Coder-1/2 work, not this audit):**

1. **Replace `blacklistPreCheck()` in ask/route.ts** with `screenEntity()` from `blacklist-client.ts`. This is the D101 wireup (Decision 101, item #2: "/ask → @milo/blacklist screenEntity() — HIGH"). The function already exists — it's used by 10+ other callsites in the same repo.

2. **Replace api/blacklist/route.ts CRUD** to use @milo/blacklist SDK (`add()`, `remove()`, `list()` from vendor/blacklist). Or delete the endpoint entirely if admin blacklist management moves to /operator UI.

**After wireup completes:**

3. **7-day cooldown observation.** Monitor for:
   - Any 500 errors mentioning `blacklist` table
   - Any Supabase logs showing queries to `blacklist` table
   - Confirm `screenEntity()` returns matches on /ask Vet queries

4. **Schema amendment** per Decision 38 — propose `DROP TABLE blacklist` in SCHEMA.md.

5. **Apply drop migration** after Mark approves.

---

## E. Risk assessment

**If legacy `blacklist` is dropped while a consumer still queries it:**
- `blacklistPreCheck()` would throw → caught by try/catch in ask/route.ts:297-302 → returns null → /ask continues without blacklist check. **No user-facing error, just silent bypass** (same behavior as now since table is empty).
- `api/blacklist/route.ts` GET/POST/DELETE would return 500. **Admin-only endpoint, no public user impact.**

**Rollback plan:**
- Recreate empty `blacklist` table via migration (schema preserved in milo-engine migration script).
- No data to restore (table was empty at drop time).

**Risk rating: LOW.** Table is empty. Both consumers are effectively dead code. Drop would change behavior from "query succeeds with 0 results" to "query fails, caught silently." The real fix is the D101 wireup, not the drop.

---

## Cross-references

- **Decision 101:** Plug-in gap audit identified `/ask → @milo/blacklist screenEntity()` as HIGH priority (item #2). Fuzzy match gap: legacy `blacklistPreCheck` does exact substring match only; `screenEntity()` supports fuzzy name, alias, contact, email, IP, LinkedIn matching.
- **Decision 113:** Coder-3 readiness audit flagged "blacklist table mismatch" as hard blocker #7. Mark confirmed `blacklist_entries` is authoritative.
- **Decision 34-35:** @milo/blacklist v0.1.0 shipped + data migrated (343 entries).

---

## Evidence

- Legacy `blacklist` row count: `content-range: */0` (0 rows)
- `blacklist_entries` row count: `content-range: 0-342/343` (343 rows)
- Consumer grep across 4 repos: 2 legacy consumers in milo-for-ppc, 1 reference-only in milo-engine, 0 in milo-outreach, 0 in mysecretary
- `blacklist-client.ts` confirmed using `@milo/blacklist` → `blacklist_entries`
