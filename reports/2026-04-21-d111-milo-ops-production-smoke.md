# D111 Report: milo-ops Post-Migration Production Smoke Test

**Agent:** Coder-2 | **Date:** 2026-04-21 | **Status:** PASS

---

## Pre-Test State

```
crm_contacts (tenant=tlp): 41
```

---

## Write Path Tested

**Endpoint:** `POST https://tlp.justmilo.app/api/contacts`

This is the production `/api/contacts` route on the milo-for-ppc Vercel deploy. It calls `findOrCreateCrmContact()` from `src/lib/crm-client.ts`, which:
1. Checks for existing contact by email (dedup)
2. Calls `findOrCreateCounterparty()` — ilike match on company name
3. Creates `crm_contacts` row via `@milo/crm` SDK with tenant='tlp'

**Request payload:**

```json
{
  "full_name": "Smoke Test 2026-04-21",
  "email": "smoke-test+20260421@justmilo.app",
  "phone": "+1-555-000-0000",
  "entity_name": "Flex Marketing",
  "entity_type": "publisher"
}
```

**Response:** HTTP 201

```json
{
  "id": "ceb9d729-7a18-48cb-8af6-649d81bb3f73",
  "full_name": "Smoke Test 2026-04-21",
  "email": "smoke-test+20260421@justmilo.app",
  "phone": "+1-555-000-0000"
}
```

---

## crm_contacts Row Verification

```json
{
  "id": "ceb9d729-7a18-48cb-8af6-649d81bb3f73",
  "tenant_id": "tlp",
  "counterparty_id": "a7d47ad2-a66f-4d07-a3d2-80df9c8f417f",
  "lead_id": null,
  "name": "Smoke Test 2026-04-21",
  "email": "smoke-test+20260421@justmilo.app",
  "phone": "+1-555-000-0000",
  "role": null,
  "is_primary": true,
  "linkedin_url": null,
  "source": "manual",
  "tags": [],
  "last_contacted_at": null,
  "notes": null,
  "created_at": "2026-04-21T22:30:41.879953+00:00",
  "updated_at": "2026-04-21T22:30:41.879953+00:00",
  "metadata": {}
}
```

**Checks:**

| Field | Expected | Actual | Status |
|-------|----------|--------|--------|
| tenant_id | tlp | tlp | PASS |
| counterparty_id | a7d47ad2... (Flex Marketing) | a7d47ad2-a66f-4d07-a3d2-80df9c8f417f | PASS |
| name | Smoke Test 2026-04-21 | Smoke Test 2026-04-21 | PASS |
| email | smoke-test+20260421@justmilo.app | smoke-test+20260421@justmilo.app | PASS |
| phone | +1-555-000-0000 | +1-555-000-0000 | PASS |
| source | manual | manual | PASS |
| is_primary | true | true | PASS |
| metadata | {} (no optional fields sent) | {} | PASS |

Counterparty resolution confirmed: `findOrCreateCounterparty("Flex Marketing", "publisher")` performed ilike match and resolved to existing counterparty `a7d47ad2...` (Flex Marketing, publisher). No new counterparty created.

**Note on metadata:** The `/api/contacts` route (route.ts) does not forward timezone, preferred_channel, or other vertical-specific fields from the request body. The `findOrCreateCrmContact` helper supports them (`opts.timezone`, `opts.preferredChannel`, etc.) but the route doesn't pass them. metadata is empty as expected for this code path. Other write paths (publisher-onboarding, leads/import, capture) may populate these fields.

---

## milo-ops Local Contacts Verification

**Query:** `SELECT * FROM contacts WHERE full_name = 'Smoke Test 2026-04-21'` on milo-ops Supabase (tmzsmdkfqvjjjwkukysg)

**Result:** `[]` — empty. No row written.

**Finding: CRM-only write. No dual-write.**

D61 migrated all 10 contact write sites to `@milo/crm`. Grep confirms zero `.from('contacts').insert` or `.from('contacts').upsert` calls remain in the codebase. The local contacts table on milo-ops Supabase is now a read-only legacy table used only by the `getContacts()` fallback path in `milo-queries.ts` (which never fires because the CRM path returns data first).

---

## Post-Test State

```
crm_contacts (tenant=tlp): 42 (+1 from pre-test)
```

---

## Cleanup Verification

**DELETE:** `crm_contacts WHERE id = ceb9d729-... AND tenant_id = 'tlp'`

```
HTTP 200 — 1 row returned in representation
```

**Post-cleanup row check:** `[]` — row gone.

**Post-cleanup count:**

```
crm_contacts (tenant=tlp): 41 (matches pre-test exactly)
```

No local contacts cleanup needed (no dual-write occurred).

---

## Verdict: PASS

| Check | Status |
|-------|--------|
| Production API reachable | PASS (HTTP 201) |
| Contact created in crm_contacts | PASS |
| counterparty_id resolves correctly | PASS (Flex Marketing) |
| tenant_id = tlp | PASS |
| source = manual | PASS |
| Dual-write to local contacts | **No** — CRM-only (D61 migration complete) |
| Post-test count = pre-test + 1 | PASS (41 → 42) |
| Cleanup successful | PASS (42 → 41) |
| Post-cleanup matches pre-test | PASS (41 = 41) |

End-to-end production write path through `findOrCreateCrmContact` is healthy. Counterparty ilike resolution, CRM SDK insert, and tenant scoping all work correctly in production.
