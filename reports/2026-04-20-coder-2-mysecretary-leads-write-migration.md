# mysecretary Leads Write-Path Migration to @milo/crm

**Date**: 2026-04-20
**Lane**: Coder-2 (Migration)
**Directive**: #33 (E1)
**Status**: COMPLETE

---

## Summary

All mysecretary lead write operations now go through `@milo/crm` SDK with `tenant_id='mysecretary'`. Zero writes to the local `leads` table remain.

---

## Commits (4 total, per-file discipline)

| SHA | File | What changed |
|-----|------|--------------|
| `0081a58` | `lib/crm-lead-writer.ts` | NEW — reverse mapper (mysecretary → CRM columns + metadata) |
| `061da6f` | `lib/outreach-client/internal.ts` | 5 write functions migrated (createLead, updateLead, deleteLead, generateDraft save, classifyReply save) |
| `1936783` | `lib/research-agent.ts` | Bulk lead insert from directory research → sequential `crmCreateLead` |
| `562e4e2` | `app/api/admin/leads/bulk/route.ts` | Bulk CSV import → sequential `crmCreateLead` |

---

## Write sites migrated (7 total)

| # | File | Operation | Before | After |
|---|------|-----------|--------|-------|
| 1 | `internal.ts:createLead` | INSERT | `sb.from("leads").insert(...)` | `crmCreateLead(crm, {...insert, tenant_id})` |
| 2 | `internal.ts:updateLead` | UPDATE | `sb.from("leads").update(...)` | `crmUpdateLead(crm, id, update)` with metadata merge |
| 3 | `internal.ts:deleteLead` | DELETE | `sb.from("leads").delete(...)` | `crm.supabase.from("crm_leads").delete()` with tenant isolation |
| 4 | `internal.ts:generateDraft` | UPDATE (draft save) | `sb.from("leads").update({[draftColumn]: next, template_used})` | `crmUpdateLead(crm, id, {metadata: {...existing, draft, template}})` |
| 5 | `internal.ts:classifyReply` | UPDATE (reply save) | `sb.from("leads").update({reply_classification, reply_draft_response, reply_classified_at})` | `crmUpdateLead(crm, id, {metadata: {...existing, reply fields}})` |
| 6 | `research-agent.ts` | bulk INSERT | `sb.from("leads").insert(rows)` | Loop `crmCreateLead` per row |
| 7 | `admin/leads/bulk/route.ts` | bulk INSERT | `sb.from("leads").insert(rowsToInsert)` | Loop `crmCreateLead` per row |

---

## Verification

```
grep '\.from\(["'"'"']leads["'"'"']\)\s*\.(insert|update|delete|upsert)' — 0 matches
npm run build — clean (all 4 commits)
```

---

## Column mapping (crm-lead-writer.ts)

| mysecretary field | CRM column | Notes |
|-------------------|-----------|-------|
| `first_name` | `metadata.first_name` | Vertical-specific |
| `last_name` | `metadata.last_name` | Vertical-specific |
| `organization` | `company` | Direct |
| `email` | `contact_email` | Rename |
| `title` | `role` | Rename |
| `linkedin_url` | `linkedin_url` | Direct |
| `website` | `website` | Direct |
| `status` | `status` (mapped values) | new→new, sent→contacted, demo→qualified, etc. |
| `tier` | `metadata.tier` | Vertical-specific |
| `hook` | `metadata.hook` | Vertical-specific |
| `source_url` | `metadata.source_url` | Vertical-specific |
| `credentials` | `metadata.credentials` | Vertical-specific |
| `channel_preference` | `metadata.channel_preference` | Vertical-specific |
| `message_draft_*` | `metadata.message_draft_*` | Draft JSONB in metadata |
| `template_used` | `metadata.template_used` | Vertical-specific |
| `reply_classification` | `metadata.reply_classification` | Vertical-specific |
| `reply_draft_response` | `metadata.reply_draft_response` | Vertical-specific |
| `reply_classified_at` | `metadata.reply_classified_at` | Vertical-specific |

---

## Notes

- **No `deleteLead` in @milo/crm**: Used `crm.supabase.from("crm_leads").delete()` directly with tenant_id scoping. Could be added to SDK later.
- **Bulk inserts are sequential**: @milo/crm doesn't have a bulk create API. Each lead is created individually. For typical batch sizes (10-50 leads), this is fine. Research agent runs are weekly and produce ~15-30 leads.
- **Read-path already done**: Decision 36 migrated reads. The round-trip (write metadata → read metadata) was verified via the existing `toMySecretaryLead` mapper in `crm-lead-mapper.ts`.

---

*Coder-2 — Migration lane*
