# D169 — V4.0 Completion Audit

**Date:** 2026-04-22  
**Agent:** Coder-3  
**Scope:** Cross-reference D85/D86 V4.0 scope against DIRECTIVE-LEDGER + git log to produce single source of truth on V4.0 completion.

---

## Item 1: Mirror URL Backfill

6 Coder-3 reports were missing the `Report at:` footer line per D158 convention. Fixed:

| Report | Directive |
|---|---|
| 2026-04-21-d106-milo-outreach-crm-scoping.md | D106 |
| 2026-04-21-d114-v4-design-surface-audit.md | D114 |
| 2026-04-21-d118-v4-design-surface-audit.md | D118 |
| 2026-04-21-d136-deferred-predicate-specs.md | D136 |
| 2026-04-22-d140-sales-accounting-impl-specs.md | D140 |
| 2026-04-22-d149-git-bus-feasibility.md | D149 |

Files in `v4-shell-refinements/` and `v4-1-scope/` subdirectories remain exempt per D158 ruling.

---

## Item 2: V4.0 Completion Status

### Source of truth

- **Decision 85** (DECISION-LOG.md:1162) — V4 shape definition + amendment listing remaining work
- **Decision 86** (DECISION-LOG.md:1194) — V4.0 pill scope: 8 pills, Billing→Accounting merge

### V4.0 Scope Items

#### A. Pill infrastructure (Decision 86: 8 pills)

| Pill | Status | Evidence | Notes |
|---|---|---|---|
| Alerts | SHIPPED | Pre-existing (alert-based) | 9 alert predicates already wired before V4.0 |
| Contracts | SHIPPED | D139 `32355b4` (ppc) | Direct-source pill, Coder-1 |
| Aging | SHIPPED | D134 `c89ed50` (ppc) | Direct-source pill, Half 1 |
| Sales | SHIPPED | D143 `ec51453` (ppc) | Snoozable + tabs, Coder-1 |
| Accounting | SHIPPED | D143 `ec51453` (ppc) | Snoozable + tabs, Billing merged in |
| Q&A | SHIPPED | D145 `7cdf684` (ppc) | Half 2, Coder-1 |
| Calls | SHIPPED | D145 `7cdf684` (ppc) | Predicate with dispute exclusion |
| Disputes | SHIPPED | D134 `c89ed50` (ppc) | Direct-source pill, Half 1 |

**Pill status: 8/8 SHIPPED.** Verified by D146 + D146b visual verification (`b9da9e6`, `a0bcfe4`). 15 total pills live (9 alert-based + 6 direct-source).

#### B. Pill route infrastructure (D85 amendment)

| Item | Status | Evidence |
|---|---|---|
| /api/operator/pill multi-source expansion | SHIPPED | D134 Half 1 (`c89ed50`), D145 Half 2 (`7cdf684`) |
| DIRECT_SOURCE_PILLS branching | SHIPPED | 19 drawer-capable pills via DRAWER_PILL_IDS |
| PillDrawer Option B (snoozable + tabs) | SHIPPED | D143 (`ec51453`) |

**Infrastructure status: SHIPPED.**

#### C. Shell refinements (D85 design iteration)

| Refinement | Spec | Status | Evidence | Blocker |
|---|---|---|---|---|
| 01 Mic right of SearchBar | `v4-shell-refinements/01-mic-placement.md` | NOT STARTED | Spec written (D152, `784d512`) | No directive issued to Coder-1 |
| 02 Footer sync status | `v4-shell-refinements/02-footer-sync-status.md` | SHIPPED | D155 (`f50e8ae`), SyncFooter.tsx deployed | — |
| 03 Long-press jiggle | `v4-shell-refinements/03-long-press-jiggle.md` | NOT STARTED | Spec written (D152, `784d512`) | No directive issued |
| 04 Curated + picker | `v4-shell-refinements/04-curated-picker.md` | NOT STARTED | Spec written (D152+D156, `b503ab2`+`8bc9dd3`) | No directive issued |
| 05 File-drop auto-routing | `v4-shell-refinements/05-file-drop-auto-routing.md` | ALREADY SHIPPED | D161 discovered UniversalDropZone (985 LOC) already handles all D85 requirements | Removed from build order |
| 06 Decommission /dashboard-v2 | `v4-shell-refinements/06-decommission-dashboard-v2.md` | SHIPPED | D159 (`7d71a09`), -2,214 LOC | — |

**Shell refinements: 3/6 SHIPPED, 3 NOT STARTED** (mic, jiggle, picker).

#### D. Other V4.0 items (D85 amendment)

| Item | Status | Evidence | Notes |
|---|---|---|---|
| "Full dashboard →" link removal | SHIPPED | D159 decommission removed it along with /dashboard-v2 | — |
| D144 HTML design reference audit | IN FLIGHT | Ledger: IN FLIGHT, no commit | Coder-3 — never completed. Non-blocking for V4.0 functionality |

---

### Summary

| Category | Shipped | Remaining | Total |
|---|---|---|---|
| Pills (Decision 86) | 8 | 0 | 8 |
| Pill infrastructure | 3 | 0 | 3 |
| Shell refinements | 3 | 3 | 6 |
| Other | 1 | 1 | 2 |
| **Total** | **15** | **4** | **19** |

### Remaining V4.0 work

| # | Item | ~LOC | Spec ready? | Directive needed |
|---|---|---|---|---|
| 1 | Mic right of SearchBar | ~60 | YES (spec 01) | Coder-1 directive |
| 2 | Long-press jiggle mode | ~210 | YES (spec 03) | Coder-1 directive |
| 3 | Curated + picker | ~230 | YES (spec 04) | Coder-1 directive |
| 4 | D144 HTML design reference | ~0 (research) | N/A | Coder-3 to complete |

**Items 1-3** are spec'd and ready for Coder-1 implementation directives. Total ~500 LOC across ~12 files.

**Item 4** (D144) is a research/documentation task that does not block any implementation. It can be completed in parallel or deferred.

### Verdict

**V4.0 is NOT complete.** 3 shell refinements from D85 (mic, jiggle, picker) have specs written but no implementation directives issued. All pill work and infrastructure is done. The remaining ~500 LOC of implementation is self-contained — no dependencies on each other or on V4.1 work.

V4.1 work (D164b scope stubs, D167 component relocation) should NOT start until these 3 refinements ship, since V4.1 items like drag-reorder depend on jiggle mode (spec 03).

---

Report at: https://raw.githubusercontent.com/moscaleflow/conductor-mark-public/main/reports/2026-04-22-d169-v4-completion-audit.md
