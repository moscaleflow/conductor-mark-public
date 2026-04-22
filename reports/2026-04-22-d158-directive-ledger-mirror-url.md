# D158 — Directive Ledger + Mirror URL Convention Enforcement

**Coder:** 4 (Ops)  
**Directive:** D158  
**Date:** 2026-04-22  
**Status:** COMPLETE

---

## Item 1: Directive Ledger

### Created

`DIRECTIVE-LEDGER.md` — canonical record of all directives from D40 onward (plus D23/D28/D35/D38 recovered from early commits).

### Scope

- **95 rows** covering D23 through D158
- Backfilled from: conductor-mark git log, milo-for-ppc git log, report filenames, DECISION-LOG.md, TASKS.md recently-completed section
- Statuses: COMPLETE, DEFERRED, CANCELLED, SUPERSEDED, IN FLIGHT

### Verification

- **Commit SHAs:** 14 sampled across both repos, all verified present in git log
- **Mirror URLs:** 5 sampled, all return HTTP 200. 1 excluded file (d147) confirmed 404.
- **Excluded reports:** d94-live (PII) and d147 (env vars) marked with EXCLUDED note instead of mirror URL

### Gap documentation

Directive numbers with no commit or report evidence are documented in the "Gap notes" section. Known cancelled/unverified directives (D75-D76, D91, D93, D98, D104, D108, D124) are explicitly marked CANCELLED with notes.

Numbers in the D112-D113, D116, D119-D120, D123, D130-D133, D135 range have no evidence — likely verbal/chat-only directives or numbers consumed by multi-part batches.

---

## Item 2: Mirror URL Convention Enforcement

### Audit results (last 5 report commits)

| Commit | Report | Coder | Had URL? | Fixed? |
|---|---|---|---|---|
| c3ca75d (D157) | d157-resend-smtp-lane-check.md | **Coder-4** | YES | — |
| 8bc9dd3 (D156) | v4-shell-refinements/04, 06 | **Coder-3** | N/A (spec files) | Exempt |
| af97218 (D153) | d153-dashboard-v2-reference-audit.md | **Coder-1** | NO | YES |
| a0bcfe4 (D146b) | d146b-v4-pill-visual-verification.md | **Coder-1** | NO | YES |
| b503ab2 (D152) | v4-shell-refinements/04, 05, 06 | **Coder-3** | N/A (spec files) | Exempt |

Additional reports fixed:
- D146 (Coder-1): d146-v4-pill-verification.md — missing, now added
- D151 (Coder-1): d151-v41-pills.md — missing, now added

### Coders not following convention

**Coder-1** — 4 reports missing the mirror URL line (D146, D146b, D151, D153). All fixed in this commit.

**Coder-3** — Spec files in `v4-shell-refinements/` subdirectory don't have the line, but these are exempt (referenced from parent reports, not fetched individually). Standard Coder-3 reports (D149, D140, D136) were committed before the convention existed (D150). No violation.

**Coder-4** — Compliant since D150 when the convention was established.

### Rule update

AGENT-SCOPES/README.md updated: "Coder reporting convention" section now reads **"NON-NEGOTIABLE"** and states "A report without this line is incomplete." Spec file subdirectories explicitly exempted.

---

## Commits

| SHA | Description |
|---|---|
| (this commit) | D158 ledger + mirror URL fixes + AGENT-SCOPES update |

---

Report at: https://raw.githubusercontent.com/moscaleflow/conductor-mark-public/main/reports/2026-04-22-d158-directive-ledger-mirror-url.md
