# Handoff Snapshot — 2026-04-25 (v2)

> End-of-day state for next session pickup. v2 captures directive relay system, late-day vet card shipments, research quality improvements, and two-stage extraction.

---

## 1. Production state

**Vercel deploy:** LIVE on main. All commits auto-deploy. No build errors.

**milo-for-ppc (tlp.justmilo.app):**
- /ask: 4-pill chat tool with conversation memory, Surface 18 auto-classify (0.85 threshold), vet non-streaming, long-paste UX, 6-button action tray, animated thinking dots, needs-research badge. 7 catch blocks wired to ops_error_log.
- /ask vet pipeline: Two-stage extraction (Haiku entity list + parallel Sonnet research per entity). Per-entity caching. Research quality fix shipped — associated entity context, aggressive search instructions, maxTokens 2048. EntityVetCard: Edit button, secondary name stubs, assign dropdown with full_name, needs-research badge + pipeline write-path.
- /review: Spend dashboard + synthetic harness + chat_logs growth + OpsHealthWidget (24h error aggregation). Auth bypass via `?dev=true`.
- /operator: V4 shell Phase 1, 15 pills live.
- /proof + justmilo.app hero: Stable, no changes today.

**Shared Supabase (tappyckcteqgryjniwjg):** Healthy. 72 migrations applied, zero drift.

**PAT rotation:** Old leaked `sbp_34a8...` stripped from all code. New `sbp_edfe...` in .env.local. Supabase Dashboard rotation is a Mark manual step (may already be done).

---

## 2. Directive relay system (NEW — most significant coordination change today)

**What:** Filesystem-based message passing replaces human copy-paste relay between Strategy-Claude and Coders. Eliminates ~300 manual copy-paste actions per session.

**How it works:**
- Strategy-Claude writes directives to `directives/inbox/coder-N/`
- Coders poll inboxes (30s recommended), execute, write reports to `directives/outbox/coder-N/`
- Mark/PM archives completed pairs to `directives/archive/YYYY-MM-DD/`
- Approval gates: safe work ships immediately; destructive work (schema amendments, data migrations, credential rotations, DNS changes) requires explicit `-approved.md` file from Mark

**Scripts:**
- `poll-inbox.sh N` — show Coder-N's pending directives
- `poll-outbox.sh` — show all outbox reports
- `dashboard.sh` — 4-lane status overview (IDLE / WORKING / DONE)
- `archive.sh N slug` — move completed pairs to archive

**Status:** Bootstrapped and tested. Round-trip verified: directive written at 19:55:14 → detected → report written → archived → all IDLE by 19:56:30. All scripts handle macOS space-in-path (`mark conduct`). Coder-4 RELAY READY confirmed in outbox.

**Adoption:** Opt-in next session. Agent Teams sessions skip relay (direct spawn). Falls back to copy-paste by ignoring the folders. Fully reversible — delete inbox/outbox/archive/scripts to revert.

**Design doc:** `docs/RELAY-SYSTEM.md`. Decision: D146.

---

## 3. Active in-flight work

None. All 4 Coders at end-of-day idle.

---

## 4. Open Mark action items

1. **10 NEEDS-OWNER-DECISION items from dead code audit (D137):** 6 DELETE, 2 KEEP, 2 REWRITE recommended. Includes security escalation on auth-server.ts query-param fallback (Item J — unauthenticated privilege escalation on 5 API routes). Report: `reports/2026-04-25-coder-3-owner-decision-recommendations.md`.

2. **Supabase PAT rotation on Dashboard:** If not already done, revoke `sbp_34a8...` in Supabase Dashboard > Settings > Access Tokens. New token already in .env.local.

3. **Resend justmilo.app DNS verification:** D162 Item 1 still GATED. Demo emails silently failing until justmilo.app is verified in Resend.

4. **PROPOSED decisions awaiting review:** D98 (22 pill refinements), D100 (10 contract gaps), D102 (Snapshot 2 plan), D106 (12 voice refinements).

### Coder-3 research deliverables ready for directive creation

1. **Entity profile + kanban (D132):** 8-directive sequence mapped. Directives 1-3 minimum for "vet results spawn entity profile > kanban board." Two parallel entity systems (crm_counterparties + prospect_pipeline) need reconciliation.

2. **Secondary names UX (D141):** 6-phase implementation spec. Phases 1-3 core (~7h), phases 4-5 profile building (~4h), phase 6 assign fix (~0.5h).

3. **Workflow audit blockers (D127):** 5 HIGH blockers. #1 conversation memory CLOSED (D131). #2-3 persist state still open. #4 file picker still open. #5 Surface 18 CLOSED (D128).

---

## 5. Decision log delta since v1 snapshot

| # | Title | Status |
|---|---|---|
| D144 | Two-stage vet extraction — Haiku entity list + parallel Sonnet research | SHIPPED |
| D145 | Vet research quality fix — associated entity context + aggressive search | SHIPPED |
| D146 | Shared-file directive relay system | BOOTSTRAPPED |

v1 covered D127-D143. Sequence is now 1-73, 77-146, canonical, zero TBD headers remain.

---

## 6. Known issues

1. **auth-server.ts query-param fallback (D137 Item J):** `?userId=X&isAdmin=true` bypasses auth on 5 routes. URGENT security fix recommended.
2. **chat_logs 90-day archive cron:** Table + edge function deployed but nightly cron registration pending.
3. **Resend SMTP:** justmilo.app not verified — demo emails silently failing (D162).
4. **10 MEDIUM/RISKY dead code items:** Not yet deleted, awaiting Mark review (D137).

---

## 7. 10x roadmap — what ships next

**Polished (production-ready, iterate on feedback):**
- /ask conversation memory, Surface 18 auto-classify, vet non-streaming, long-paste UX, animated thinking dots, action tray, needs-research badge
- Two-stage vet extraction with per-entity caching
- EntityVetCard with Edit button, assign dropdown, secondary name stubs
- Directive relay system (ready for first real multi-session use)

**Prototype (works but needs hardening):**
- /operator V4 shell (15 pills, no deep CRUD)
- OpsHealthWidget (reads ops_error_log but no alerting)
- Vet research quality (good for 2 entities, untested at scale >5)

**Queued next (highest leverage):**
- Entity profile + kanban (D132) — vet results spawn actionable entity cards on a board
- Secondary names UX (D141) — compact stubs + server-side classification for known-entity recognition
- Workflow audit blockers #2-4 (D127) — persist state, file picker
- Dead code owner decisions (D137) — especially Item J auth-server.ts security fix
- Resend DNS verification (D162) — unblocks demo email flow

**Architectural bets in play:**
- @milo/ai-client extraction (D129) — voice, model routing, caching as a package
- Two-stage extraction pattern — generalizable beyond vet (contract analysis, review summarization)
- Relay system — if adopted, reduces PM coordination overhead by ~60%

---

## Coordination docs refreshed this session

| Doc | Change |
|---|---|
| DECISION-LOG.md | 17 TBD entries > D127-D143. Relay system > D146. Zero TBDs remain. |
| CURRENT-STATE.md | v13 > v14. /ask section expanded (10+ new capabilities). Active lanes refreshed. |
| TASKS.md | Current state > 2026-04-25. Lane statuses refreshed. 18 entries added to Recently completed. |
| CLAUDE.md | Directive flow updated to reference relay system + docs/RELAY-SYSTEM.md. |
| docs/RELAY-SYSTEM.md | NEW — full design doc for relay system. |

---

## Late-day commits (milo-for-ppc, chronological)

| Hash | Description |
|---|---|
| 24d31b3 | feat: two-stage vet extraction — Haiku entity list + parallel Sonnet research |
| bd25503 | feat: EntityVetCard action tray — equal-width edge-to-edge buttons |
| 2e73dd1 | feat: EntityVetCard refinements — kill left ribbon, add Edit button, secondary name stubs |
| 18c94ab | fix: vet research quality — associated entity context + aggressive search |
| d1f23ec | fix: separate JSON parse from DB save in per-entity result assembly |
| e106312 | perf: reduce per-entity web_search max_uses from 3 to 2 |
| d9829c3 | fix: per-entity Sonnet calls pass only entity name, not full chat paste |
| 401adde | fix: include team-roster full_name field + assign dropdown improvements |
| a0d4c93 | fix: increase per-entity vet maxTokens from 1024 to 2048 |
| 3764198 | fix: needs-research badge + pipeline write-path fixes for EntityVetCard |

---

## References

- CURRENT-STATE.md — v14 (master state snapshot)
- DECISION-LOG.md — Decisions 1-73, 77-146 (canonical, collision-free)
- TASKS.md — Live lane states
- MILESTONE-LOG.md — Narrative journey log
- docs/RELAY-SYSTEM.md — Directive relay system design doc
- reports/2026-04-25-coder-3-owner-decision-recommendations.md — 10 items for Mark
- reports/2026-04-25-coder-3-entity-profile-kanban-research.md — Entity system map
- reports/2026-04-25-coder-3-secondary-names-spec.md — Secondary names UX spec
- reports/2026-04-25-coder-3-dead-code-audit.md — Full dead code audit
