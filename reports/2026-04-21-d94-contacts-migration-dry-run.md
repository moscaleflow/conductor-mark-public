# D94 Report: milo-ops Contacts Data Migration — Dry-Run

**Agent:** Coder-2 | **Date:** 2026-04-21 | **Status:** Dry-run complete, awaiting Mark approval

---

## Pre-Flight Audit

| Metric | Count |
|--------|-------|
| Source contacts (milo-ops `tmzsmdkfqvjjjwkukysg`) | 239 |
| Target crm_contacts (shared, tenant=tlp) | 18 |
| Counterparties (shared, tenant=tlp) | 601 |

### Key Discovery

The source `contacts` table lives on the **milo-ops Supabase** (`tmzsmdkfqvjjjwkukysg`) — a separate project from the shared Supabase (`tappyckcteqgryjniwjg`). The D61 commit notes said "~20 secondary read sites still reference local contacts table" — these reads would currently fail in production (Vercel's `NEXT_PUBLIC_SUPABASE_URL` points to shared Supabase, which has no `contacts` table). The CRM-first + local-fallback pattern from D61 gracefully handles this.

---

## Dry-Run Results

| Action | Count |
|--------|-------|
| **Would create** | 23 |
| Skipped (duplicate — name already in crm_contacts) | 1 |
| Skipped (no counterparty match) | 215 |
| Skipped (no name) | 0 |
| Errors | 0 |
| **Total processed** | 239 |

### 23 Contacts to Create

| Name | Company | Has Email |
|------|---------|-----------|
| (EC) Alpha Solutions | (EC) Alpha Solutions | no |
| (EC) Bluecross Biz Solutions | (EC) Bluecross Biz Solutions | no |
| (EC) Hey Solution | (EC) Hey Solution | no |
| (EC) NET | (EC) NET | no |
| (EC) Perfecient BPO | (EC) Perfecient BPO | no |
| (EC) SN Media | (EC) SN Media | no |
| (EC) Vedanta Infotech | (EC) Vedanta Infotech | no |
| (M8) Call Pixel Media | (M8) Call Pixel Media | no |
| (MM) IClick | (MM) IClick | no |
| 1st Choice Marketing | 1st Choice Marketing | no |
| 1st Choice Marketing Auto IB 27/155 | 1st Choice Marketing Auto IB 27/155 | no |
| 1st Choice Marketing FE IB $30/90s | 1st Choice Marketing FE IB $30/90s | no |
| ActualSales ACA IB Short Duration RTB | ActualSales ACA IB Short Duration RTB | no |
| Alberto Moraes | Interest Media | yes |
| Ariel Tolome | Pigeonfi Inc | yes |
| Aware Ads Auto IB RTB New | Aware Ads Auto IB RTB New | no |
| EJ Sinlao | Upbeat | yes |
| Jess Hans | Aware Ads | yes |
| John | LeadSource Pro | no |
| Mariela | Ray Advertising | yes |
| Nir Algazy | Lead Buyer Hub | yes |
| Target Web Media | Target Web Media | no |
| Turtle Leads | Turtle Leads | no |

6 of 23 have email addresses (real person contacts). 17 of 23 are company-name-as-contact-name entries (entity records masquerading as contacts in the flat milo-ops model).

### 215 Skipped: No Counterparty

These source contacts have `company_name` values that don't match any `crm_counterparties.company` (exact, case-insensitive). Breakdown:

- **~9 numeric-only names** (TrackDrive IDs like `14512875`) — not real contacts
- **~1 coded prefix** — campaign code with no base company match
- **~205 real companies/people** — their companies weren't migrated in D32 (PPCRM→CRM migration) because they weren't in PPCRM's partner/lead tables. These are milo-ops-native contacts created after the PPCRM freeze.

Per directive constraint: "If counterparty doesn't exist, log + skip the contact. Don't auto-create — that's Wave 2 territory."

### 1 Skipped: Duplicate

- **Santos Corp** (email: corp@test.com) — already in crm_contacts from ppcrm-migration

---

## Decision Points for Mark

1. **Proceed with 23-contact live run?** These are the contacts with matching counterparties. Safe, idempotent, no auto-creation of counterparties.

2. **The 215 skipped contacts represent a gap.** Most are milo-ops-native contacts whose companies don't exist in crm_counterparties. Options:
   - **Option A:** Accept the gap. 23 contacts migrate now; the rest migrate when their counterparties are created (Wave 2 or manual).
   - **Option B:** Run a companion counterparty migration from milo-ops' `prospect_pipeline` or `contacts.company_name` → `crm_counterparties` first, then re-run this migration to pick up more matches. Larger scope.
   - **Option C:** Allow the script to auto-create counterparties for the 215. Directive says no, but Mark can override.

3. **17 of 23 "contacts" are actually company/campaign records** (full_name = company_name, no email). The flat milo-ops model used one table for both people and entities. These will look odd in crm_contacts alongside real person contacts. Harmless but noisy.

---

## Script

`scripts/migrate-milo-ops-contacts.ts` — committed, ready for live run.

```
DRY_RUN=false \
  MILO_OPS_SERVICE_ROLE_KEY=<key> \
  SHARED_SERVICE_ROLE_KEY=<key> \
  npx tsx scripts/migrate-milo-ops-contacts.ts
```

---

## Verification Plan (post live-run)

1. crm_contacts count: 18 → 41 (delta = 23)
2. Spot-check 5 random migrated contacts: counterparty_id resolves, metadata populated
3. All 239 source rows accounted for: 23 created + 1 duplicate + 215 no-counterparty + 0 errors = 239
