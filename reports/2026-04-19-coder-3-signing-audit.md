---
directive: "@milo/contract-signing pre-extraction audit"
lane: research
coder: Coder-3
started: 2026-04-19 ~22:00 MDT
completed: 2026-04-19 ~22:30 MDT
---

# @milo/contract-signing Pre-Extraction Audit — 2026-04-19

## 1. Executive Summary

MOP's e-signing infrastructure is ~4,857 LOC of mostly horizontal code with a fully custom signing flow (no DocuSign). The extraction boundary is clean: signing lifecycle, audit trail, HMAC webhooks, and UI shell are generic; templates and CompanySettings defaults are vertical. Two bugs need fixing before extraction: the HMAC webhook inconsistency in the decline route (a one-liner fix) and missing w8ben/w8bene branches in `generateSignedDocument.ts`. Template dispatch logic is copy-pasted across 4 locations with drift (437 lines of duplication). Multi-tenant migration exists but is entirely unapplied. Estimated `@milo/contract-signing` v0.1.0: ~2,800 LOC.

## 2. HMAC Webhook Inconsistency — Deep Analysis

**Confirmed bug with three distinct discrepancies between submit and decline routes.**

### submit/route.ts (correct implementation, lines 415-419)

```typescript
const { headers: webhookHeaders } = signBody(payloadJson);
```

Uses shared `signBody()` from `@/lib/webhooks`. Computes HMAC over `${timestamp}.${body}` format. Timestamp is Unix epoch seconds. Uses `RECONCILIATION_WEBHOOK_SECRET` with fallback to `WEBHOOK_SIGNING_SECRET`.

### decline/route.ts (broken implementation, lines 104-115)

```typescript
const webhookTimestamp = declinedAt;  // ISO 8601 string!
const webhookHeaders: Record<string, string> = {
  'Content-Type': 'application/json',
  'X-Webhook-Timestamp': webhookTimestamp,
};
const webhookSecret = process.env.WEBHOOK_SIGNING_SECRET;
if (webhookSecret) {
  const hmac = createHmac('sha256', webhookSecret);
  hmac.update(payloadJson);   // body-only, no timestamp prefix
  webhookHeaders['X-Webhook-Signature'] = `sha256=${hmac.digest('hex')}`;
}
```

Imports `createHmac` directly (line 3) instead of using shared helper.

### Three discrepancies

| Aspect | submit (correct) | decline (broken) |
|--------|------------------|------------------|
| HMAC input | `${timestamp}.${body}` | `body` only |
| Timestamp format | Unix epoch seconds | ISO 8601 string |
| Secret env var | `RECONCILIATION_WEBHOOK_SECRET` -> `WEBHOOK_SIGNING_SECRET` | `WEBHOOK_SIGNING_SECRET` only |

### Fix assessment: One-liner

Replace lines 103-115 of `decline/route.ts` with:

```typescript
const payloadJson = JSON.stringify(webhookPayload);
const { headers: webhookHeaders } = signBody(payloadJson);
```

Also: add `import { signBody } from '@/lib/webhooks';`, remove `import { createHmac } from 'crypto';`.

**Severity:** Any PPCRM consumer verifying HMAC via `verifyHmac()` will reject decline webhooks. If PPCRM does not verify, the bug is latent. If it does, decline events silently fail signature validation.

## 3. Template Dispatch Duplication — Full Analysis

### Four locations, 437 lines of duplicated dispatch logic

| Location | File | Lines | Branches | Missing Types |
|----------|------|-------|----------|---------------|
| 1. Generate | `app/api/bot/documents/generate/route.ts` | ~172 | 8 (all types) | none |
| 2. Submit | `app/api/sign/[token]/submit/route.ts` | ~118 | 7 | rider |
| 3. Signed doc | `app/sign/[token]/utils/generateSignedDocument.ts` | ~50 | 5 | w8ben, w8bene, rider |
| 4. Live preview | `app/sign/[token]/hooks/useLivePreview.ts` | ~97 | 7 | rider |

### Drift matrix

| Document Type | generate | submit | generateSigned | livePreview |
|---------------|----------|--------|----------------|-------------|
| buyer+msa | Yes | Yes | Yes | Yes |
| buyer+io | Yes | Yes | Yes | Yes |
| publisher+msa | Yes | Yes | Yes | Yes |
| publisher+io | Yes | Yes | Yes | Yes |
| w9 | Yes | Yes | Yes (partial) | Yes |
| w8ben | Yes | Yes | **NO** | Yes |
| w8bene | Yes | Yes | **NO** | Yes |
| rider | Yes | **NO** | **NO** | **NO** |

### Bugs caused by drift

1. **`generateSignedDocument.ts` missing w8ben/w8bene** — Certificate of Execution page renders empty contract body for these tax document types. A signer who tries to print/download after signing a W-8BEN gets a broken document.

2. **Rider type not in submit/preview/signed-doc** — A signed rider produces empty/broken output because only the generate endpoint knows how to dispatch it.

### Extraction opportunity

A single `dispatchTemplate(entityType, docType, templateData, signerData?, companySettings?)` function in a template registry module would eliminate all four copies and prevent future drift. ~437 lines of duplication -> ~60 lines of registry.

## 4. CompanySettings Coupling

### Defaults file (`lib/companySettingsDefaults.ts`, 52 LOC)

100% TLP-hardcoded:

| Field | Value |
|-------|-------|
| company_name | "The Lead Penguin" |
| legal_name | "Lead Penguin LLC" |
| owner_name | "Tiffani Kinkennon" |
| owner_title | "Managing Member" |
| address | "1500 N Grant St, STE R, Denver, CO 80203" |
| branding_color | `#dd72a6` (TLP pink) |
| bank_name | "US Bank" |
| bank_account_number | "103700980446" |
| bank_routing_number | "102000021" |

Note: `bank_account_holder` has a typo — "Tiffany" vs "Tiffani" (inconsistent with `owner_name`).

### Fetch chain (`lib/companySettings.ts`, 48 LOC)

1. Query `company_settings` table (`.select('*').limit(1).single()`) — **single-tenant assumption**
2. Per-field fallback: each field falls back to `DEFAULT_COMPANY_SETTINGS` if DB value is falsy (using `||`)
3. If DB query errors or returns null, returns full `DEFAULT_COMPANY_SETTINGS` copy
4. Exception: `outreach_auto_send` uses `??` (correctly handles `false`)

### Where consumed

| Location | How |
|----------|-----|
| `submit/route.ts` line 147 | `await getCompanySettings()` (server) |
| `bot/documents/generate/route.ts` line 138 | `await getCompanySettings()` (server) |
| `generateSignedDocument.ts` line 58 | `{ ...DEFAULT_COMPANY_SETTINGS, ...companySettings }` (client merge) |
| `useLivePreview.ts` | Does NOT use company settings (renders without branding) |

### Extraction approach

For `@milo/contract-signing`, defaults should come from `vertical.config.ts` (aligned with Thesis 2.5). The `getCompanySettings()` function needs a tenant_id parameter. The fallback chain stays the same — DB overrides config defaults.

## 5. Multi-Tenant Readiness

### Migration `20300101000000_multi_tenant_rls.sql` (exists, unapplied)

Comprehensive 8-step migration:

1. Creates `tenants` table (id, name, slug)
2. Adds `tenant_id UUID REFERENCES tenants(id)` to 30 tables including `signing_documents` (line 74) and `partner_documents` (line 75)
3. Backfill script (commented out)
4. Enables RLS on 15 tables including `signing_documents` (line 151)
5. Drops duplicate policies
6. Drops open `USING(true)` policies
7. Creates tenant-scoped RLS: `auth.jwt()->>'tenant_id'`
8. NOT NULL constraints (commented out)

### Design decisions documented in migration

- `service_role` key bypasses RLS — bot endpoints switch from anon to service_role
- Public signing page stays server-side token-validated (no JWT for counterparty)
- Reference tables (rider_templates, etc.) shared across tenants
- `company_settings` noted as "needing tenant_id if needed"

### Current state: zero tenant awareness

All Supabase queries use admin client with no tenant filter. `partner_documents` CRUD (`lib/documents/supabase.ts`) still uses the anon key client — will break when RLS is enabled.

## 6. Signing Components Classification

| Component | File | LOC | Props | Classification | Notes |
|-----------|------|-----|-------|----------------|-------|
| PersonalInfoFields | `SigningForm.tsx` | 46 | 13 | **Horizontal** | Generic person-info fields |
| CompanyDetailsFields | `SigningForm.tsx` | 352 | 40+ | Mixed | Address horizontal; W-8BEN/W-8BEN-E tax fields vertical |
| SignatureFields | `SigningForm.tsx` | 110 | 15 | **Horizontal** | Typed/drawn signature, E-SIGN consent |
| SigningSidebar | `SigningSidebar.tsx` | 286 | 113 | **Horizontal** | Orchestrator, 3-step accordion |
| SigningPreview | `SigningPreview.tsx` | 56 | 5 | **Horizontal** | Generic iframe preview |
| SigningOverlay | `SigningOverlay.tsx` | 61 | 5 | **Horizontal** | "Start Signing" CTA |
| SigningConfirmation | `SigningConfirmation.tsx` | 227 | 10 | **Mostly horizontal** | Audit trail display, print/download |

**Component total: 1,138 LOC, ~90% horizontal.**

The only vertical coupling in components is:
- `CompanyDetailsFields`: W-8BEN/W-8BEN-E tax field groups (IRS-specific)
- `SigningConfirmation`: calls `generateSignedDocument()` which has the dispatch duplication
- `SigningPreview`: document type label mapping (msa/io/w9/rider, 6 lines)

## 7. Documents Library Analysis

### `lib/documents/supabase.ts` (305 LOC) — **Horizontal**

CRUD for `partner_documents` table:
- `createDocument()` — insert + auto-update compliance flags
- `getDocument()`, `getPublisherDocuments()`, `getBuyerDocuments()`
- `archiveDocument()` — soft delete + recalculate flags
- `deleteDocumentRecord()` — hard delete + recalculate flags
- `getDocumentCountsByType()` — returns `{MSA, IO, Amendment, NDA, Other}`
- `linkAnalysisToDocument()` — links contract analysis

**Warning:** Uses `supabaseUntyped` (anon key client), not `supabaseAdmin`. Will break when RLS is enabled.

**Case mismatch:** `partner_documents.document_type` uses uppercase (`MSA`, `IO`) while `signing_documents.document_type` uses lowercase (`msa`, `io`). Normalization needed before extraction.

### `lib/documents/compliance.ts` (224 LOC) — Mixed

- `getBlacklistedCompanies()` — fetches from `master_blacklist_publishers` + `master_blacklist_buyers`
- `getAllPartnersCompliance(filters?)` — publishers + buyers with compliance status
- Compliance model: `has_msa === true && has_io === true` = compliant

The compliance model (MSA+IO=compliant) is vertical-specific to performance marketing. Infrastructure (blacklist, flags, filtering) is horizontal.

### `lib/documents/storage.ts` (114 LOC) — **Horizontal**

Generic Supabase Storage helpers for `partner-documents` bucket:
- `generateStoragePath()`, `uploadDocument()`, `getDocumentUrl()`, `deleteDocument()`, `listEntityDocuments()`

### `lib/taxDocRouting.ts` (43 LOC) — **Horizontal**

Routes `tax` document type to `w9`/`w8ben`/`w8bene` based on country + entity status. Standard IRS form routing.

### `app/api/s/[code]/route.ts` (54 LOC) — **Horizontal**

Short URL resolver. Checks `signing_documents` then `invoices` table for `short_code`. Returns JSON redirect path.

## 8. Additional Vertical Coupling Points

Hardcoded TLP strings found in signing routes:

| File | Line(s) | Hardcoded Value |
|------|---------|-----------------|
| `submit/route.ts` | 393, 495 | `tlpmop.netlify.app` URLs |
| `submit/route.ts` | email strings | "The Lead Penguin Contracts", "support@theleadpenguin.com" |
| `decline/route.ts` | 93 | `tlpmop.netlify.app` URL |
| `companySettingsDefaults.ts` | throughout | All TLP branding, address, bank details |

These are parameterizable via CompanySettings — the infrastructure already supports it, just not consistently applied.

## 9. Extraction Recommendations

### Bugs to fix before extraction

1. **HMAC in decline route** — one-liner: replace hand-rolled HMAC with `signBody()` import
2. **Missing w8ben/w8bene in generateSignedDocument.ts** — add 2 branches (~20 lines)
3. **Case mismatch** — normalize document_type to lowercase across both tables

### Architecture changes for extraction

1. **Template registry** — single `dispatchTemplate()` function replaces 4 copy-paste locations (saves 437 lines, prevents drift)
2. **CompanySettings** — move defaults to `vertical.config.ts`, add tenant_id to `getCompanySettings()`
3. **Supabase client consistency** — `lib/documents/supabase.ts` must switch from anon to admin client

### What ships in @milo/contract-signing v0.1.0

| Component | LOC | Notes |
|-----------|-----|-------|
| Signing lifecycle (generate, view, sign, decline, void) | 900 | Extract from submit/decline/capture-view routes |
| Template registry (abstract dispatch) | 60 | Replaces 4 copy-paste locations |
| Audit trail (JSONB event log) | 150 | Already in signing lifecycle |
| HMAC webhook signing | 83 | `lib/webhooks.ts` as-is |
| SHA-256 document/signature hashing | 100 | Extract from submit route |
| Short URL resolution | 54 | `app/api/s/[code]/route.ts` as-is |
| Certificate of Execution generation | 200 | From generateSignedDocument.ts |
| Signing UI shell (5 components) | 780 | ~90% horizontal, strip tax field groups |
| Signing hooks (3 files) | 500 | useDocumentLoader, useSigningForm (parameterized), useLivePreview |
| Document management CRUD | 305 | `lib/documents/supabase.ts` |
| Storage helpers | 114 | `lib/documents/storage.ts` |
| Tax document routing | 43 | `lib/taxDocRouting.ts` |
| CompanySettings fetch | 48 | Parameterized with tenant_id |
| Types | 130 | SigningDocument, PartnerDocument interfaces |
| DB migrations (signing_documents, partner_documents) | 150 | Horizontal columns only |
| **Total** | **~2,817** | |

### What stays in PPC vertical

- All 8 HTML templates (buyer/publisher MSA/IO, W-9, W-8BEN, W-8BEN-E, rider) — 4,088 LOC
- CompanySettings defaults (TLP-specific data) — 52 LOC
- Compliance tracking model (MSA+IO=compliant) — 224 LOC
- Entity compliance flag updates on publishers/buyers — embedded in routes
- Hardcoded TLP email strings and URLs — embedded in routes

## 10. Estimated @milo/contract-signing v0.1.0: ~2,800 LOC

## 11. Open Questions

1. **Template ownership.** Do templates ship with `@milo/contract-signing` as a plugin interface (vertical provides render functions), or do they stay entirely in the PPC vertical and the signing package just receives pre-rendered HTML?

2. **Compliance model.** Is "MSA+IO=compliant" universal enough to extract, or is it PPC-specific? Other verticals might define compliance differently (e.g., NDA+contract=compliant).

3. **Tax forms.** W-9/W-8BEN/W-8BEN-E templates are IRS forms, not PPC-specific. Should they ship with the horizontal package or stay vertical?

4. **Short URL scope.** The resolver currently checks both `signing_documents` and `invoices`. Should `@milo/contract-signing` own the short URL system, or should it be a separate shared utility?
