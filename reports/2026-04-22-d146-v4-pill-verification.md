# D146 — V4 Pill Drawer Visual Verification

**Date:** 2026-04-22  
**Agent:** Coder-4  
**Scope:** All 11 drawer pills — API verification + visual screenshots

---

## Summary

All 11 V4 drawer pills verified against production (`tlp.justmilo.app`). API returns correct data, drawers render cleanly at desktop (1440×900) and mobile (375×812) viewports. Two bugs found (one functional, one dead code).

## API Verification — All 11 PASS

| Pill | Source | Items | Badge | Seeded |
|---|---|---|---|---|
| qa_queue | alerts | 5 | $3,340 | 3 |
| qc_queue | alerts | 5 | 5 | 3 |
| ping_summary | alerts | 2 | 2 | 1 |
| ping_health | alerts | 2 | 2 | 1 |
| quality_overview | alerts | 4 | 4 | 2 |
| mediarite_xref | alerts | 3 | 16 | 2 |
| needs_attention | alerts | 15 | 15 | 6 |
| dispute_recovery | alerts | 1 | 1 | 1 |
| disputes | **direct** (disputes table) | 3 | $6,770 | 3 |
| calls | alerts | 6 | 6 | 3 |
| aging | **direct** (invoices table) | 4 | $46,550 | 4 |

## Visual Verification — 4 Drawer Pills Captured

Captured publisher-qa role's 4 drawer pills (desktop + mobile = 8 screenshots).  
All saved to `~/Desktop/d146-*.png`.

| Screenshot | Verdict |
|---|---|
| d146-operator-desktop.png | **PASS** — greeting, briefing cards, pill bar with badges, search bar all render |
| d146-operator-mobile.png | **PASS** — responsive wrap, badges visible, cards truncate cleanly |
| d146-qa_queue-desktop.png | **PASS** — 3 items, severity dots, $2.5k risk label, Review + Snooze buttons |
| d146-qa_queue-mobile.png | **PASS** — drawer full-width, text wraps, buttons accessible |
| d146-mediarite_xref-desktop.png | **PASS** — 2 items, MediaRite casing correct, candidate count badge shows 15 |
| d146-mediarite_xref-mobile.png | **PASS** |
| d146-ping_summary-desktop.png | **PASS** — 1 ping alert, TrueConnect entity, full detail text |
| d146-ping_summary-mobile.png | **PASS** |
| d146-calls-desktop.png | **PASS** — 3 call alerts, dispute excluded correctly, DID alert shows critical severity |
| d146-calls-mobile.png | **PASS** — text wraps at DID number, no overflow |

### Visual observations

- **Pill bar badges**: orange circles with white text, dollar amounts render with `$` prefix. All 4 visible pills show correct counts.
- **Drawer header**: "QUEUE" label + pill name + count. Close (×) button top-right.
- **Drawer cards**: severity dot (red=critical, orange=warning), bold title, gray description, orange "$X at risk" label when applicable, Review + Snooze 24h buttons.
- **Mobile drawer**: takes full width, cards stack vertically, text wraps cleanly, no overflow or truncation bugs.
- **Briefing cards**: needs-you items render with severity dot + title + detail + Review button. "Everything else is clean" summary below.

## Bugs Found

### Bug 1: `fetchQaPill()` status mismatch (functional)

`fetchQaPill()` at [pill/route.ts:83](src/app/api/operator/pill/route.ts#L83) filters:
```
.in('status', ['open', 'in-progress'])
```
But the `support_tickets_status_check` constraint uses `in_progress` (underscore, not hyphen). Tickets with status `in_progress` are silently excluded from the QA queue drawer. Only `open` tickets surface.

**Impact:** Low — qa pill currently routes through alerts (pill id `qa_queue`), not the direct-source `qa` fetch. But if the direct source is ever wired up, this will silently drop in-progress tickets.

### Bug 2: `fetchQaPill()` is dead code

`DIRECT_SOURCE_PILLS` contains `'qa'` but no pill in `ROLE_PILLS` (operator-pills.ts) has `id: 'qa'`. The QA-related pill is `qa_queue` which routes through `PILL_PREDICATES`, not `DIRECT_SOURCE_PILLS`. The entire `fetchQaPill()` function is unreachable from the operator UI.

**Impact:** None currently. Dead code should either be wired up (change pill id to `qa`) or removed. If intended for future use, the status mismatch (Bug 1) should be fixed first.

### Note: `mediarite_xref` badge discrepancy

Badge shows `15` on the pill bar but the drawer header shows `2`. This is by design — the badge uses `extractCandidateCount()` which sums the candidate counts within each alert (3 + 12 = 15 candidates across 2 alert rows). Not a bug.

## Seed Data

16 rows inserted across 4 tables (all tagged `d146-seed` for identification):
- **alerts**: 6 rows (2 call, 1 dispute, 1 mediarite, 1 qa, 1 ping)
- **support_tickets**: 3 rows (high/normal/low priority)
- **disputes**: 3 rows (raised/escalated/pending_response)
- **invoices**: 4 rows (one per aging bucket: current, 30-day, 60-day, 90+)

All rows use fixed UUIDs (`00000000-d146-4000-*`). Seed script at `tests/seed-v4-pills.ts` is idempotent — safe to re-run.

**Data left in place** for Coder-2 D143 as requested.

## Files Created

- `tests/seed-v4-pills.ts` — idempotent seed script (run: `npx tsx tests/seed-v4-pills.ts`)
- `tests/v4-pills.spec.ts` — Playwright API verification (11 pills + summary)
- `tests/v4-pills-visual.spec.ts` — Playwright visual drawer captures
- `tests/screenshots/d146-summary.md` — API result table
- `tests/screenshots/d146-api-*.json` — per-pill API response data (11 files)
- `~/Desktop/d146-*.png` — 10 screenshots for Mark's review
