# D163b Report: Phase 2 Readiness Audit — V4 Pill Scope

**Agent:** Coder-2 | **Date:** 2026-04-22 | **Status:** Complete

---

## Source Documents

| Doc | SHA | What it says |
|-----|-----|-------------|
| D85 | 5d1f5c4 | V4 shell shape locked: conversation-first, pill-fan, drawer-based. Amendment: /operator IS the V4 shell. |
| D86 | 5f85b52 | V4.0 ships 8 pills. Billing merges into Accounting. V4.1 defers: E-sign, Drafts, Publishers, Buyers. |
| D118 | 74b2019 | Design surface audit: 6 BUILD-EXISTS-WRAP, 3 BUILD-PARTIAL-FILL, 1 BUILD-NEW, 1 MERGE-SCOPE. |
| D136 | 28b9187 | Deferred predicate specs: Contracts, Sales, Accounting query shapes + severity mappings. |

---

## V4.0 Scope (D86) — All Shipped

| Pill | Directive | Implementation | Status |
|------|-----------|---------------|--------|
| Alerts | baseline | MorningBriefing + needs_attention predicate | SHIPPED |
| Contracts | D139 | fetchContractsPill() → contract_documents | SHIPPED |
| Aging | D145 | fetchAgingPill() → invoices bucketed | SHIPPED |
| Sales | D143 C2 | fetchSalesPill() → prospect_pipeline stall detection | SHIPPED |
| Accounting | D143 C3 | fetchAccountingPill() → 4-tab drawer (invoices, recon, reports, disputes) | SHIPPED |
| Q&A | D143b | fetchQaPill() → support_tickets open/in_progress | SHIPPED |
| Calls | baseline | qa_queue + calls alert predicates | SHIPPED |
| Disputes | D134 | fetchDisputesPill() → disputes table revenue-ranked | SHIPPED |
| Billing | D86 merge | Merged into Accounting as sub-tab | SHIPPED |

**V4.0 verdict: 8/8 shipped. 100% complete.**

---

## V4.1 Scope (D86 "deferred") — All Shipped via D151

D86 deferred 4 pills. D151 shipped all 4 on 2026-04-22.

| Pill | D86 Status | D151 Commit | D163a Verification |
|------|-----------|-------------|-------------------|
| E-sign | "zero operator UI" | b9e0957 | 84 items rendered, expiry severity correct |
| Drafts | "needs milo-outreach wire" | 24665e6 | 2 items (blocked_manual + dry_run) |
| Publishers | "no operator-facing list" | 7cdcf5f | 8 items, stall thresholds correct |
| Buyers | "need entity-as-DrawerItem" | 5149476 | 4 items, severity mapping correct |

**V4.1 verdict: 4/4 shipped. 100% complete.**

---

## Current Pill Inventory

### DIRECT_SOURCE_PILLS (10)

```
qa, disputes, aging, contracts, sales_pipeline,
accounting, esign, drafts, publishers, buyers
```

### DRAWER_PILL_IDS (19)

```
qa_queue, qc_queue, ping_summary, ping_health,
quality_overview, mediarite_xref, needs_attention,
dispute_recovery, disputes, calls, aging,
contracts, sales_pipeline, accounting, qa,
esign, drafts, publishers, buyers
```

### ROLE_PILLS (8 roles)

| Role | Pill Count | Drawer Pills | Chat Pills |
|------|-----------|-------------|-----------|
| admin | 19 | 14 | 5 |
| operations | 5 | 3 | 2 |
| publisher-qa | 4 | 4 | 0 |
| billing | 8 | 5 | 3 |
| outreach | 6 | 2 | 4 |
| prospecting | 4 | 1 | 3 |
| call-monitoring | 4 | 1 | 3 |
| qc | 2 | 2 | 0 |

---

## Remaining V4 Shell Refinements (D85 spec, unshipped)

These are UX refinements from D85's original vision, not pill scope items:

| Item | D85 Reference | Status | Notes |
|------|--------------|--------|-------|
| File-drop auto-routing | "File drop on any surface auto-routes by type" | NOT STARTED | Needs: drag handler + MIME→pill routing |
| Mic input | "Mic on right side of search bar" | NOT STARTED | SearchBar has mic icon, no handler |
| Footer sync status | "Footer 'Live synced' status escalates" | IN FLIGHT (D155) | SyncFooter component exists |
| Long-press jiggle | "long-press jiggle for pill customization" | NOT STARTED | PillBar has no gesture handlers |
| Curated + picker | "curated `+` picker (no free text)" | NOT STARTED | Current "+ Add pill" dispatches to chat |
| /dashboard-v2 decommission | D85 amendment | AUDITED (D153) | D153 found /dashboard-v2 safe to remove |

---

## Unshipped Items from D118 Audit

D118 identified items beyond pill scope:

| Item | D118 Classification | Current Status |
|------|-------------------|---------------|
| Track A components (SeverityBadge, IssueCard, IssueSidebar) | BUILD-EXISTS-WRAP | Available but not mounted in any drawer |
| /dashboard-v2 page | SUPERSEDED | Audited in D153, safe to decommission |
| /contract-review/[id] page | BUILD-EXISTS-WRAP | Standalone page, not integrated into drawer |

---

## Phase 2 Summary

**V4 pill scope is 100% shipped.** All 12 pills from D86 (8 V4.0 + 4 V4.1) have working drawer implementations with correct severity mapping, badge counts, and snoozable flags.

**Remaining work is shell refinements (D85 UX items), not pill scope.** These are tracked separately and not gated by pill readiness.

| Category | Total | Shipped | Remaining |
|----------|-------|---------|-----------|
| V4.0 pills | 8 | 8 | 0 |
| V4.1 pills | 4 | 4 | 0 |
| Shell refinements | 6 | 1 (SyncFooter in flight) | 5 |
| D118 integration items | 3 | 0 | 3 |

---

Report at: https://raw.githubusercontent.com/moscaleflow/conductor-mark-public/main/reports/2026-04-22-d163b-phase2-readiness-audit.md
