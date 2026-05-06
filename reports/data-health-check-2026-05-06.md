# Production Data Health Check -- 2026-05-06

**Project:** tappyckcteqgryjniwjg (shared Supabase)
**Scope:** Read-only queries against live production data
**Run by:** Coder-4 (Ops)

---

## 1. Record Counts

| Table | Count | Filter |
|---|---|---|
| crm_counterparties | 664 | tenant_id = 'tlp' |
| crm_contacts | 78 | tenant_id = 'tlp' |
| prospect_pipeline | 1,078 | no tenant_id column |
| vet_results | 224 | tenant_id = 'tlp' |
| chat_logs | 410 | no tenant_id column |
| call_notes | 2 | no tenant_id column |
| send_log | 5 | tenant_id = 'tlp' |
| blacklist_entries | 346 | -- |

**Schema gap noted:** `prospect_pipeline`, `chat_logs`, and `call_notes` have no `tenant_id` column. If this database ever serves multiple tenants, these tables will be unpartitioned.

---

## 2. CRM Counterparty Duplicates

### Exact duplicates (4 pairs, 8 rows)

All are buyers, all status "paused", all created 2026-02-11 (PPCRM migration batch).

| Company | Count | IDs |
|---|---|---|
| Equoto Auto Inbound RTB | 2 | 4ca1ff16..., d3c3403d... |
| Equoto FE IB RTB | 2 | 4cc938f0..., d4d1e66c... |
| Motiv8 ACA Blind TF RTB PH ONLY | 2 | a13e55e1..., 3a2f74d8... |
| Turtle Leads ACA IB RTB | 2 | 42b52805..., dc57cbc8... |

**Root cause:** PPCRM migration on 2026-02-11 inserted these twice. No unique constraint on (tenant_id, company) prevented it.

### Near-duplicates (3 groups)

| Normalized | Variants | Notes |
|---|---|---|
| leadfox | "LeadFox", "Leadfox " (trailing space) | Separate entries -- one has trailing whitespace |
| test publisher | "Test Publisher Co" (lead), "Test Publisher Corp" (lead), "Test Publisher US -- DELETE ME" (onboarding) | Test data that should be cleaned |
| leadfox aca... | "Leadfox ACA Inbound Spanish 30/125" | This is a separate buyer deal line, likely intentional |

**Total unique companies:** 660 out of 664 rows (4 exact dupes).

### CRM Counterparty Distribution

**By status:** paused (361), active (233), lead (57), onboarding (10), contacted (2), qualified (1)
**By type:** buyer (431), publisher (227), prospect (6)

---

## 3. Prospect Pipeline Duplicates

### Multi-stage duplicates: 27 entities appearing in 2-3 pipeline stages

Every pipeline "duplicate" has the same entity in DIFFERENT stages. Zero true duplicates (same entity + same stage). This means the pipeline allows an entity to exist in multiple stages simultaneously rather than moving through a single-row lifecycle.

**Top offenders (3 copies each):**
- BindRight (publisher): qualifying, dormant, dormant
- Call Center Power (publisher): dormant, qualifying, dormant
- Channel Automation (publisher): qualifying, dormant, dormant
- HS Solutions (publisher): dormant, dormant, onboarding

**2-copy entities (23 total):** Ace Solutions, AdBeacon, Archenia Inc, Coversion Finder, Digital Hub LLC, Digital Remedy, Elevance Health, Ideal Concepts Inc, John Leisinger (buyer), Lab6 Media, Lead Generation World, MediaRite LLC, Monarch Media Digital, Peachy Insurance, Providence, Pulaski Law Firm, ResultFirst, Revenue Click Media LLC, Seafront Marketing, Sender Science, Single Source International, Sled Dog Consulting, Tort Expert

**Root cause:** No unique constraint on (entity_name, entity_type). The upsert logic either failed or was bypassed, creating new rows instead of updating existing ones.

### Pipeline Stage Distribution

| Stage | Count |
|---|---|
| qualifying | 820 |
| dormant | 67 |
| outreach | 35 |
| onboarding | 30 |
| activation | 24 |
| active | 23 |
| blacklisted | 1 |

---

## 4. Vet Results Duplicates

**Severe duplication.** 224 total rows, only 127 unique entity_name_normalized values. 97 rows are duplicates.

### Worst offenders

| Entity (normalized) | Copies | Likely Cause |
|---|---|---|
| ameriquote incorporated | 16 | Re-vetted 16 times |
| national media solutions | 16 | Re-vetted 16 times |
| quickconnect digital | 12 | Re-vetted 12 times |
| reachmax networks | 12 | Re-vetted 12 times |
| firmfuel | 11 | Re-vetted 11 times |
| granite media | 10 | Re-vetted 10 times |
| kevin butlers | 10 | Re-vetted 10 times |
| brightpath digital | 7 | Re-vetted 7 times |
| matthew barner / topclass leads | 3 | -- |
| topclass leads | 3 | -- |
| unknown -- no entity provided | 3 | Garbage entries |
| adgenix media | 2 | -- |
| glassman disability | 2 | -- |
| n/a | 2 | Garbage entries |
| testco | 2 | Test data |
| unknown -- no entity submitted | 2 | Garbage entries |

**Root cause:** No dedup on insert. Every chat-based vet request creates a new vet_results row even if the entity was already vetted. The top 8 entities account for 94 of 97 duplicate rows.

**Garbage rows:** 7 rows with entity names like "unknown", "n/a", "testco" that have no real entity behind them.

---

## 5. Orphaned Records

### Pipeline entries with no CRM counterparty match

**911 out of 1,078 pipeline entries (84.5%) have no matching company in crm_counterparties.**

This is the most significant data integrity finding. The pipeline and CRM are almost entirely disconnected.

**Orphans by stage:**
| Stage | Orphan Count |
|---|---|
| qualifying | 784 |
| dormant | 58 |
| onboarding | 29 |
| activation | 18 |
| active | 18 |
| vetted | 2 |
| outreach | 1 |
| blacklisted | 1 |

**Reverse direction:** 575 out of 664 CRM counterparties (86.6%) have no matching pipeline entry.

**Interpretation:** The CRM counterparties are primarily PPCRM-migrated buyer/publisher records (created 2026-02-11). The pipeline is a newer system (created 2026-05-05) populated from a different source. They were never linked. The pipeline has no foreign key to crm_counterparties -- matching is purely by name, and the naming conventions differ (CRM uses deal-line names like "(EC) Phonovo LLC" while pipeline uses clean entity names like "Phonovo LLC").

---

## 6. CRM Contacts Without Email

**64 out of 78 contacts (82.1%) have no email address.**

All 64 null-email contacts are `source: manual`. Only 14 contacts have email addresses. This severely limits outreach capability.

**Schema note:** The contacts table uses a single `name` column -- no `first_name`/`last_name` split. This matches the current code but limits structured name handling.

---

## 7. Schema Gap Analysis (call_notes)

**All expected columns are present:**
- flag_status: present (sample value: "flagged")
- resolution_note: present (sample value: null)
- updated_at: present (sample value: null)

Additional columns: id, call_id, did, operator_id, operator_name, note_text, created_at, note_type

**No schema gaps in call_notes.** The code-audit finding about missing columns does not apply to the current production schema.

---

## 8. Multi-Tenant Schema Gaps

Three tables lack `tenant_id`:

| Table | Row Count | Risk |
|---|---|---|
| prospect_pipeline | 1,078 | HIGH -- core business data, no tenant isolation |
| chat_logs | 410 | MEDIUM -- usage logs, no tenant filtering |
| call_notes | 2 | LOW -- tiny table, but no isolation |

Currently TLP is the only tenant using these tables, so this is not causing data leakage today. But it blocks multi-tenant deployment.

---

## 9. Severity Summary

| Finding | Severity | Rows Affected | Recommended Action |
|---|---|---|---|
| Pipeline-CRM disconnect (84.5% orphaned) | HIGH | 911 pipeline + 575 CRM | Build a linking migration or accept as separate systems |
| Vet results duplication (43% are dupes) | HIGH | 97 duplicate rows | Add upsert dedup; archive stale vet results |
| Contacts missing email (82.1%) | HIGH | 64 contacts | Enrich or accept as known gap |
| Pipeline multi-stage duplicates | MEDIUM | 27 entities, 57 extra rows | Add unique constraint, merge duplicates |
| CRM exact duplicates (PPCRM migration) | MEDIUM | 4 pairs, 8 rows | Merge and add unique constraint |
| CRM near-duplicates (whitespace/suffix) | LOW | 3 groups | Normalize on insert, clean existing |
| Test/garbage data in vet_results | LOW | 7 rows | Delete "unknown", "n/a", "testco" rows |
| Test data in CRM | LOW | 3 rows | Delete "Test Publisher" rows |
| Missing tenant_id on 3 tables | LOW (now) | 1,490 rows | Add column before 2nd tenant |

---

## 10. Recommended Cleanup Actions (Priority Order)

### Immediate (safe, no code changes needed)

1. **Delete CRM test data:** Remove "Test Publisher Co", "Test Publisher Corp", "Test Publisher US -- DELETE ME"
2. **Merge CRM exact duplicates:** For each of the 4 duplicate pairs, keep the older ID, delete the newer, update any FK references
3. **Trim CRM whitespace:** Fix "Leadfox " -> "Leadfox" (trailing space)
4. **Delete vet_results garbage:** Remove rows where entity_name_normalized is "unknown", "n/a", "testco"

### Short-term (requires code changes)

5. **Add unique constraint on crm_counterparties:** `UNIQUE(tenant_id, LOWER(TRIM(company)))` -- prevents future duplicates
6. **Add upsert-or-skip logic for vet_results:** Check if entity was vetted within last N days before creating new row
7. **Add unique constraint on prospect_pipeline:** `UNIQUE(entity_name, entity_type)` -- requires merging existing duplicates first

### Medium-term (architecture decision needed)

8. **Pipeline-CRM linking strategy:** Either add `crm_counterparty_id` FK to pipeline with a migration script, or accept them as independent systems with name-based cross-reference
9. **Contact email enrichment:** Decide whether to backfill emails or accept contacts as metadata-only
10. **Multi-tenant columns:** Add `tenant_id` to prospect_pipeline, chat_logs, call_notes before launching a second vertical

---

Report at: https://raw.githubusercontent.com/moscaleflow/conductor-mark-public/main/reports/data-health-check-2026-05-06.md
