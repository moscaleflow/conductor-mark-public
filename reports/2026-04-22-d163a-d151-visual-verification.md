# D163a Report: D151 Visual Verification — E-sign, Drafts, Publishers, Buyers

**Agent:** Coder-2 | **Date:** 2026-04-22 | **Status:** Complete

---

## Seed Data Extended

Extended `tests/seed-v4-pills.ts` (Coder-1's D146 seed) with 4 new source tables:

| Table | Rows | Notes |
|-------|------|-------|
| signing_documents | 3 | Expired (-2d), 48h expiry, 7d expiry. Required `signing_token` NOT NULL fix. |
| leads | 2 | FK requirement for send_log. Tenant: tlp. |
| send_log | 2 | blocked_manual + dry_run. Required `from_address` NOT NULL fix. |
| crm_counterparties (publisher) | 3 | lead (30d stall), onboarding (18d stall), contacted (12h recent). |
| crm_counterparties (buyer) | 3 | qualified (45d stall), contacted (20d stall), lead (18h recent). |

Schema constraint fixes during seed (Pattern 17 applied):
- `signing_documents.signing_token` — NOT NULL, added seed tokens
- `send_log.from_address` — NOT NULL, added `milo@scaleflow.co`
- `send_log.lead_id` — FK to `leads(id)`, inserted seed leads first
- `crm_counterparties.source` — CHECK constraint: `'outbound'` invalid, changed to `'manual'`

---

## API Verification

All 4 pills return correct data via `/api/operator/pill`:

| Pill | Count | Seed Rows | Production Rows | Severity Mapping |
|------|-------|-----------|-----------------|-----------------|
| esign | 84 | 3 (expired=critical, 48h=critical, 7d=warning) | 81 (viewed=warning) | Correct |
| drafts | 2 | 2 (blocked_manual=warning, dry_run=info) | 0 | Correct |
| publishers | 8 | 3 (30d lead=critical, 18d onboarding=warning, 12h recent=info) | 5 | Correct |
| buyers | 4 | 3 (45d qualified=critical, 20d contacted=warning, 18h recent=info) | 1 | Correct |

---

## Visual Verification (Playwright)

Screenshots captured at desktop (1440x900) + mobile (375x812) for all 4 pills.

**Files on Desktop:**
- `d151-esign-desktop.png` / `d151-esign-mobile.png`
- `d151-drafts-desktop.png` / `d151-drafts-mobile.png`
- `d151-publishers-desktop.png` / `d151-publishers-mobile.png`
- `d151-buyers-desktop.png` / `d151-buyers-mobile.png`

### Visual Audit Results

| Pill | Drawer Header | Badge Count | Severity Colors | Action Button | Snooze Hidden | Mobile Wrap |
|------|--------------|-------------|-----------------|---------------|---------------|-------------|
| E-sign | "QUEUE E-sign 84" | 84 | red/orange correct | "Open signing" | Yes | Clean |
| Drafts | "QUEUE Drafts 2" | 2 | orange/blue correct | "Review draft" | Yes | Clean |
| Publishers | "QUEUE Publishers 8" | 8 | red/orange correct | "Open publisher" | Yes | Clean |
| Buyers | "QUEUE Buyers 4" | 4 | red/orange/blue correct | "Open buyer" | Yes | Clean |

### Defects Found

**None.** All 4 pills render correctly in both viewport sizes.

### Observations (non-blocking)

1. **E-sign production data**: 81 production rows from signing_documents with `counterparty_name = NULL` render as "Unknown" — expected behavior per the `fetchEsignPill` fallback chain (`counterparty_name ?? counterparty_email ?? 'Unknown'`). Several "Flex Marketing" entries appear as duplicates — likely multiple documents per counterparty.

2. **Pill bar badges**: All 19 drawer pills show orange badge counts. The pill bar wraps correctly across 4 rows at desktop width.

3. **SyncFooter visible**: "Rendering..." + "Offline - MCP API down" footer appears at bottom — expected for localhost without MCP connection.

---

## Test Artifacts

- Seed script: `tests/seed-v4-pills.ts` (extended, not committed)
- Playwright spec: `tests/d163a-d151-visual.spec.ts` (new, not committed)
- Screenshots: `tests/screenshots/d151-*.png` + `~/Desktop/d151-*.png`

---

Report at: https://raw.githubusercontent.com/moscaleflow/conductor-mark-public/main/reports/2026-04-22-d163a-d151-visual-verification.md
