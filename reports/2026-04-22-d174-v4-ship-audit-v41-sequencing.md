# D174 — V4.0 Ship Audit + V4.1 Implementation Sequencing

**Date:** 2026-04-22  
**Agent:** Coder-3  
**Scope:** (1) Audit V4.0 as COMPLETE after D173a/b/c ship. (2) Sequence V4.1 implementation from D164b stubs.

---

## Item 1: V4.0 Ship Audit

### Gate status: D173a/b/c NOT YET SHIPPED

D173a (mic), D173b (jiggle), D173c (picker) have **zero commits** in milo-for-ppc, **zero code** in the codebase (no SpeechRecognition, no jiggle/longPress handlers, no PillPicker/CuratedPicker), **zero reports** in conductor-mark, and **no ledger entries**. The directive numbers have not been issued yet.

**V4.0 cannot be closed until D173a/b/c ship.** The audit below is pre-filled — once those 3 land, update the SHAs and this document becomes the V4.0 closure record.

### Complete V4.0 scope — 19 items

#### A. Pills (Decision 86) — 8/8 SHIPPED

| # | Pill | Directive | SHA (ppc) |
|---|---|---|---|
| A1 | Alerts | Pre-existing | (9 alert predicates, pre-V4) |
| A2 | Contracts | D139 | `32355b4` |
| A3 | Aging | D134 | `c89ed50` |
| A4 | Sales | D143 | `ec51453` |
| A5 | Accounting | D143 | `ec51453` |
| A6 | Q&A | D145 | `7cdf684` |
| A7 | Calls | D145 | `7cdf684` |
| A8 | Disputes | D134 | `c89ed50` |

Visual verification: D146 (`b9da9e6`) + D146b (`a0bcfe4`) + D163a (`a220911`).

#### B. Pill infrastructure — 3/3 SHIPPED

| # | Item | Directive | SHA (ppc) |
|---|---|---|---|
| B1 | /api/operator/pill multi-source | D134+D145 | `c89ed50`, `7cdf684` |
| B2 | DIRECT_SOURCE_PILLS branching | D134 | `c89ed50` |
| B3 | PillDrawer Option B (snoozable + tabs) | D143 | `ec51453` |

#### C. Shell refinements (D85) — 3/6 SHIPPED, 3 PENDING D173

| # | Refinement | Status | Directive | SHA (ppc) |
|---|---|---|---|---|
| C1 | Mic right of SearchBar | **PENDING D173a** | — | — |
| C2 | Footer sync status | SHIPPED | D155 | `5aea221` |
| C3 | Long-press jiggle | **PENDING D173b** | — | — |
| C4 | Curated + picker | **PENDING D173c** | — | — |
| C5 | File-drop auto-routing | ALREADY SHIPPED | D161 (UniversalDropZone) | Pre-existing (985 LOC) |
| C6 | Decommission /dashboard-v2 | SHIPPED | D159 | `63c0762` |

#### D. Other — 1/2 SHIPPED, 1 NON-BLOCKING

| # | Item | Status | Directive | SHA |
|---|---|---|---|---|
| D1 | "Full dashboard →" link removal | SHIPPED | D159 | `63c0762` (ppc) |
| D2 | D144 HTML design reference audit | IN FLIGHT | D144 | — (non-blocking for ship) |

### Summary

| Category | Shipped | Pending |
|---|---|---|
| Pills | 8 | 0 |
| Infrastructure | 3 | 0 |
| Shell refinements | 3 | **3 (D173a/b/c)** |
| Other | 1 | 1 (D144, non-blocking) |
| **Total** | **15** | **4** |

**Action required:** Issue D173a (mic ~60 LOC), D173b (jiggle ~210 LOC), D173c (picker ~230 LOC) to Coder-1. Specs ready at `v4-shell-refinements/01`, `03`, `04`. After those 3 ship, update this doc with SHAs → V4.0 CLOSED.

---

## Item 2: V4.1 Implementation Sequencing

### Already shipped (remove from queue)

| Stub | Item | Shipped by | SHA (ppc) | Notes |
|---|---|---|---|---|
| 01 | Dashboard-v2 component relocation | D167 (Coder-1) | `36d10f8` | 3 components → shared/, 29 dead files remain |
| 02 | API rename /dashboard-v2 → /operator/executive | D167 (Coder-1) | `971566c` | 301 redirect added |
| 04 | Snooze for non-alert DrawerItems | D171 (Coder-2) | `fd1c499` | 109 LOC, snoozed_items table + API + PillDrawer branch |

**3 of 10 V4.1 items already shipped.** 7 remain.

### Remaining 7 items — dependency graph

```
D173b (jiggle) ──→ Item 03 (drag-reorder)
                        │
                        ↓
                   Item 08 (surface types) ← needs PillDrawer stable + Mark input
                        │
Item 05 (contract clauses) ← independent (needs Contracts drawer stable)
Item 06 (stall thresholds) ← independent (needs Sales drawer stable)
Item 07 (cross-pill linking) ← independent (needs all drawers stable)
Item 09 (health cache) ← independent (needs SyncFooter stable = D155 ✓)
Item 10 (file-drop enhancements) ← independent (needs UniversalDropZone stable)
```

### Ordered implementation plan

**Wave 1 — Independent items, no blockers (parallel across lanes)**

| Order | Stub | Item | ~LOC | Lane | Pre-reserved directive | Dependency |
|---|---|---|---|---|---|---|
| 1a | 09 | Health check cache endpoint | ~31 | Coder-1 or 4 | **D175** | D155 SyncFooter ✓ shipped |
| 1b | 05 | Contract inline clause expansion | ~110 | Coder-1 | **D176** | Contracts drawer ✓ shipped (D139) |
| 1c | 06 | Configurable stall thresholds | ~50 | Coder-2 | **D177** | Sales drawer ✓ shipped (D143) |
| 1d | 07 | Cross-pill linking | ~25 | Coder-1 or 2 | **D178** | All V4.0 drawers ✓ shipped |

All 4 can run in parallel immediately. Zero dependencies on unshipped work.

**Wave 2 — Blocked on D173b (jiggle mode)**

| Order | Stub | Item | ~LOC | Lane | Pre-reserved directive | Dependency |
|---|---|---|---|---|---|---|
| 2a | 03 | Pill drag reorder | ~80 | Coder-1 | **D179** | D173b jiggle must ship first |

Cannot start until jiggle mode (D173b) deploys. PillBar needs the jiggle-mode state hook to attach drag handlers.

**Wave 3 — Needs Mark design input**

| Order | Stub | Item | ~LOC | Lane | Pre-reserved directive | Dependency |
|---|---|---|---|---|---|---|
| 3a | 08 | Surface types (left drawer, bottom sheet, full-screen) | ~305 | Coder-1 | **D180** | Mark must decide: which pills use which surface, per-pill vs per-device |
| 3b | 10 | File-drop enhancements (Vision, multi-file, dispute evidence, timesheet) | ~155 | Coder-1 or 2 | **D181** | Mark must decide: parallel vs sequential multi-file, Vision API cost budget |

Both require design decisions before implementation. Can be issued once Mark answers the open questions in the stubs.

### Directive reservation summary

| Directive | Item | Wave | Lane | ~LOC | Blocked on |
|---|---|---|---|---|---|
| D175 | Health check cache endpoint | 1 | Coder-1/4 | 31 | Nothing |
| D176 | Contract inline clauses | 1 | Coder-1 | 110 | Nothing |
| D177 | Configurable stall thresholds | 1 | Coder-2 | 50 | Nothing |
| D178 | Cross-pill linking | 1 | Coder-1/2 | 25 | Nothing |
| D179 | Pill drag reorder | 2 | Coder-1 | 80 | D173b jiggle |
| D180 | Surface types | 3 | Coder-1 | 305 | Mark design input |
| D181 | File-drop enhancements | 3 | Coder-1/2 | 155 | Mark design input |

**Total remaining V4.1 LOC: ~756** (across 7 items, down from ~956 after 3 early ships).

### Recommended firing order

1. **Now:** Fire D175–D178 (Wave 1) across available lanes. All 4 are independent and unblocked.
2. **After D173b ships:** Fire D179 (drag-reorder).
3. **After Mark answers open questions:** Fire D180 (surface types) and D181 (file-drop enhancements).

### Dead code cleanup note

D167 relocated 3 shared components but noted **29 dead component files remain** in `src/components/dashboard-v2/`. These are only referenced by each other (no live page imports them). A bulk-delete directive (~-10,000 LOC) can run any time — it has zero dependencies and zero risk. Not numbered above because it's cleanup, not feature work. Recommend bundling with the first Wave 1 directive going to Coder-1.

---

Report at: https://raw.githubusercontent.com/moscaleflow/conductor-mark-public/main/reports/2026-04-22-d174-v4-ship-audit-v41-sequencing.md
