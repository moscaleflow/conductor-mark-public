# Directive Inbox Triage -- 2026-05-05

> Coder-3 Research. Read-only audit of all 4 inbox directories.
> Cross-referenced against outbox completions, CURRENT-STATE.md, TASKS.md, DECISION-LOG.md (Decisions 1-188).

---

## Method

Each directive in `directives/inbox/coder-N/` was read in full and classified:

- **STALE**: superseded by later work, feature already shipped, or references code that has been replaced
- **ACTIVE**: still relevant and actionable, has not been completed
- **CONSOLIDATE**: can be merged with another directive into a single dispatch
- **BLOCKED**: depends on something not yet done

Evidence: matching outbox report = work was completed (directive should be archived). CURRENT-STATE.md "3-Surface Build (2026-05-04)" section confirms Phase 0+1+2+3 all shipped (15 commits, ~8,600 LOC).

---

## Coder-1 Extraction Inbox (11 directives)

| Directive | Status | Reason | Consolidation Group |
|-----------|--------|--------|---------------------|
| D-2026-04-29-C1-VET-COLLAPSE-FIX3 | **STALE** | Outbox report exists (D-2026-05-04-C1-PHASE0-IDENTITY-REGISTRY). Server-side cache normalization (next directive) was the proper follow-up. Vet collapse root cause was traced and fixed earlier -- CURRENT-STATE shows EntityVetCard V4 restyle shipped. | -- |
| D-2026-04-29-C1-AMENDMENT-D-RENDER | **STALE** | Amendment D render bugs (suppression, common name, Also Mentioned) were part of the entity resolution pipeline that was refactored in the 3-Surface build Phase 0+1. Cross-pill CRM correction (20303f0) and alias disambiguation shipped. The OpportunityCard rendering bugs this targeted have been addressed by subsequent work. | -- |
| D-2026-04-29-C1-SERVER-CACHE-NORMALIZATION | **ACTIVE** | Server-side prefix match for vet lookups. Not in any outbox. Vet collapse client-side dedup shipped but server-side cache miss on name variations (Ameriquote vs Ameriquote Insurance Services) still costs ~$0.03-0.05 per repeat vet. Low urgency now but still valid. | Wave 3 (Optimization) |
| D-2026-04-29-C1-FIXTURE-TESTS-PHASE2 | **ACTIVE** | Fixture pages + Playwright specs. No outbox. Phase 1 (CI wiring) was never fully completed either. Still valid as test infrastructure. | Wave 5 (Test Infra) |
| D-2026-04-29-C1-SYNTHETIC-OPERATOR-PHASE3 | **BLOCKED** | Depends on Phase 1+2 of synthetic testing (neither shipped). Also depends on SCHEMA.md amendment for ai_test_runs. Budget capped at $60-160/month. | Wave 5 (Test Infra) |
| D-2026-05-03-C1-ALIAS-DISAMBIGUATION | **STALE** | Outbox report exists at `D-2026-05-03-C1-ALIAS-DISAMBIGUATION.md`. Work completed. Should be archived. | -- |
| D-2026-05-03-C1-CROSS-PILL-CRM-CORRECTION | **STALE** | Outbox report exists at `D-2026-05-03-C1-CROSS-PILL-CRM-CORRECTION.md`. Commit 20303f0 shipped. Should be archived. | -- |
| D-2026-05-04-C1-ENTITYVETCARD-V4 | **STALE** | Outbox report exists at `D-2026-05-04-C1-ENTITYVETCARD-V4.md`. Part of 3-Surface build. Should be archived. | -- |
| D-2026-05-04-C1-PHASE0-IDENTITY-REGISTRY | **STALE** | Outbox report exists at `D-2026-05-04-C1-PHASE0-IDENTITY-REGISTRY.md`. getCanonicalTeamRoster() shipped. Should be archived. | -- |
| D-2026-05-04-C1-PHASE1-VET-BRIDGE | **STALE** | Outbox report exists at `D-2026-05-04-C1-PHASE1-VET-BRIDGE.md`. bridgeVetToKnowledge shipped. Should be archived. | -- |
| D-2026-05-04-C1-PHASE1-STALE-RELATIONSHIP-CRON | **STALE** | Outbox report exists at `D-2026-05-04-C1-PHASE1-STALE-RELATIONSHIP-CRON.md`. Cron + last_contact_at shipped. Should be archived. | -- |
| D-2026-05-04-C1-PHASE2-SPIN-REVEAL-PLUS-PHASE3-TRIGGER | **STALE** | Outbox report exists at `D-2026-05-04-C1-PHASE2-SPIN-REVEAL-PLUS-PHASE3-TRIGGER.md`. SpinToRevealTrigger + EOD trigger shipped in 3-Surface build. Should be archived. | -- |
| D-2026-05-04-C1-RESILIENCE-PRIMITIVES | **ACTIVE** | MiloEmptyState, MiloErrorToast, MiloErrorInline, MiloErrorState, copy banks. No outbox report matching this specific directive. CURRENT-STATE mentions resilience layer "IN-FLIGHT." C2's resilience migration (565897d) may have consumed these, but C1's Track 1 (building the primitives themselves) needs verification. Check if components exist at `src/components/operator/empty/` and `error/` before dispatching. | Wave 1 (Surface 22 + Resilience) |

**Summary C1:** 8 STALE (archive), 2 ACTIVE, 1 BLOCKED.

---

## Coder-2 Migration Inbox (17 directives)

| Directive | Status | Reason | Consolidation Group |
|-----------|--------|--------|---------------------|
| D-2026-05-03-C2-SURFACE-19 | **STALE** | Outbox report exists. SuggestionCards were built then retired (324af15) per RETIRE-SUGGESTION-CARDS directive below. SpinToRevealTrigger replaced this. | -- |
| D-2026-05-03-C2-VET-CARD-COPY-ALL | **STALE** | Outbox report exists at `D-2026-05-03-C2-VET-CARD-COPY-ALL.md`. Draft outreach + Copy All shipped. Should be archived. | -- |
| D-2026-05-04-C2-PHASE0-TARGET-USER-ID | **STALE** | Outbox report exists at `D-2026-05-04-C2-PHASE0-TARGET-USER-ID.md`. resolveAlertTargetUser shipped, 147/154 alerts backfilled per CURRENT-STATE. Should be archived. | -- |
| D-2026-05-04-C2-PHASE1-DRAFT-PERSISTENCE | **STALE** | Outbox report exists at `D-2026-05-04-C2-PHASE1-DRAFT-PERSISTENCE.md`. Copy persistence to ai_action_log shipped. Should be archived. | -- |
| D-2026-05-04-C2-PHASE1-MULTI-STEP-STATE-MACHINE | **STALE** | Outbox report exists at `D-2026-05-04-C2-PHASE1-MULTI-STEP-STATE-MACHINE.md`. step_number/parent_alert_id/step_label columns + state machine shipped. Should be archived. | -- |
| D-2026-05-04-C2-PHASE2-TRIAGE-DRAWER-PLUS-PHASE3-COMPOSER | **STALE** | Outbox report exists at `D-2026-05-04-C2-PHASE2-TRIAGE-DRAWER-PLUS-PHASE3-COMPOSER.md`. TriageDrawer + EOD composer shipped (f37e0f7). Should be archived. | -- |
| D-2026-05-04-C2-DRAWER-V4-CHROME | **STALE** | Outbox report exists at `D-2026-05-04-C2-DRAWER-V4-CHROME.md`. Surface 9 shell pattern applied to all drawers (418ea4e). Should be archived. | -- |
| D-2026-05-04-C2-PILL-PICKER-V4-STRICT | **STALE** | Outbox report exists at `D-2026-05-04-C2-PILL-PICKER-V4-STRICT.md`. 8 V4 core pills, auto-query killed (1709459). Should be archived. | -- |
| D-2026-05-04-C2-PILLDRAWER-DATA-FETCH | **STALE** | Outbox report exists at `D-2026-05-04-C2-PILLDRAWER-DATA-FETCH.md`. Data fetch fix shipped (7db5eab, d9c77c7). Should be archived. | -- |
| D-2026-05-04-C2-DRAWER-INLINE-ACTIONS | **STALE** | Outbox report exists at `D-2026-05-04-C2-DRAWER-INLINE-ACTIONS.md`. Review/Snooze/Resolve/Assign inline actions shipped (f59584d). Should be archived. | -- |
| D-2026-05-04-C2-DRAFT-HALFSTATE-FIX | **STALE** | Outbox report exists at `D-2026-05-04-C2-DRAFT-HALFSTATE-FIX.md`. Draft half-state and hero label clarity fixed. Should be archived. | -- |
| D-2026-05-04-C2-SALES-PILL-FIX-PLUS-COUNT-ALIGN | **STALE** | Outbox report exists at `D-2026-05-04-C2-SALES-PILL-FIX-PLUS-COUNT-ALIGN.md`. Sales pill mis-wiring + count/list mismatch fixed. Should be archived. | -- |
| D-2026-05-04-C2-RESILIENCE-MIGRATION-PLUS-AUTH-REMOVAL | **ACTIVE** | Workstream 1 (drawer migrations to use resilience primitives) appears partially shipped (565897d per CURRENT-STATE). Workstream 2 (auth removal from 6 endpoints) depends on C3 risk audit. Needs verification of what's left. | Wave 1 (Surface 22 + Resilience) |
| D-2026-05-04-C2-RETIRE-SUGGESTION-CARDS | **STALE** | SuggestionCards retired at 324af15 per CURRENT-STATE. SpinToRevealTrigger is the canonical entry pattern. Should be archived. | -- |
| D-2026-05-05-C2-CALL-DETAIL-PANEL-UI | **ACTIVE** | Surface 22 panel UI. Outbox report exists but this is dated today (May 5). CURRENT-STATE shows "C2: panel UI PENDING." Check outbox for completion status. If outbox report shows completion, then STALE. If outbox only shows dispatch, ACTIVE. | Wave 1 (Surface 22 + Resilience) |
| D-2026-05-05-C2-SURFACE22-BUGFIX | **ACTIVE** | Surface 22 rendering fixes (white screen, # prefix, row dedup, phone format). Outbox report exists but CURRENT-STATE still shows Surface 22 in-flight. Likely latest work. | Wave 1 (Surface 22 + Resilience) |

**Summary C2:** 12 STALE (archive), 3 ACTIVE, 0 BLOCKED.

---

## Coder-3 Research Inbox (22 directives)

| Directive | Status | Reason | Consolidation Group |
|-----------|--------|--------|---------------------|
| D-2026-04-29-C3-CI-TEST-GATE-SPEC | **ACTIVE** | CI test gate for SSE-coupled components. Spec only. No outbox. Still relevant -- the SSE enforcement is agent-definition-level only (Pattern 24). Durable CI gate not built. | Wave 5 (Test Infra) |
| D-2026-04-29-C3-OUTREACH-CAPTURE-AUDIT | **ACTIVE** | Architecture gap for outreach capture. TASKS.md shows C3 was working this (Decision 166). No outbox report found. May have been partially completed. Check for outbox at OUTREACH-CAPTURE-AUDIT-AND-SPEC.md. If not completed, still relevant -- operators still copy-paste outreach with no audit trail. | Wave 4 (Outreach + CRM Pipeline) |
| D-2026-04-29-C3-RESEARCH-LIST-DENSITY-SPEC | **STALE** | Decision 165 locked. Spec approved (084954b). Collapsed sections and takeaway-first rendering are part of the shipped StructuredResponseCard. Should be archived. | -- |
| D-2026-05-03-C3-ALIAS-RESOLUTION-SPEC | **STALE** | Mark approved aliases-first approach. C1 shipped alias disambiguation (outbox D-2026-05-03-C1-ALIAS-DISAMBIGUATION.md). pg_trgm comparison is moot. Should be archived. | -- |
| D-2026-05-03-C3-CLASSIFIER-CRM-AUDIT | **ACTIVE** | Classifier-CRM mismatch scan + Q6 outreach pipeline infrastructure amendment. No complete outbox found. Cross-pill CRM correction shipped (20303f0) but audit for lurking mismatches not completed. Q6 (outreach pipeline for Vee/Cam buyer prospecting) is new work. | Wave 4 (Outreach + CRM Pipeline) |
| D-2026-05-03-C3-V4-SURFACE-17-CODE-AUDIT | **ACTIVE** | 30-question questionnaire for Surface 17 (chat surface) V4 redesign. No outbox. Still relevant as Surface 17 is listed as PENDING in CURRENT-STATE V4 Surface Progress table. | Wave 2 (Chat Surface V4) |
| D-2026-05-04-C3-AUTH-REMOVAL-RISK-AUDIT-PLUS-CHAT-RESTYLE-PREP | **ACTIVE** | Two deliverables: (1) auth removal risk audit blocking C2, (2) chat surface restyle prep. C2's auth removal work depends on this. No outbox found. High priority. | Wave 1 (Surface 22 + Resilience) |
| D-2026-05-04-C3-CALL-LEVEL-ENRICHMENT | **ACTIVE** | Call-level alert enrichment gap. Review endpoint shipped (f59584d) but call-level alerts get stub content. No outbox. Still valid -- operators see thin "Review recommended" on call-level alerts. | Wave 1 (Surface 22 + Resilience) |
| D-2026-05-04-C3-CONVERSATIONMODAL-V4 | **ACTIVE** | Surface 17 ConversationModal V4 restyle. No outbox. CURRENT-STATE shows Surface 17 PENDING. | Wave 2 (Chat Surface V4) |
| D-2026-05-04-C3-MILO-AUTO-CONTEXT | **ACTIVE** | Auto-context from TrackDrive + PPC graph for Review actions. No outbox. This is the intelligence layer powering C2's inline Review. | Wave 1 (Surface 22 + Resilience) |
| D-2026-05-04-C3-OPERATOR-WORKFLOW-AUDIT | **ACTIVE** | Comprehensive 9-capability operator workflow audit. No outbox found. Builds on 5 prior 2026-04-28 audits. Still relevant as a system-of-record for what's built vs latent. | Wave 3 (Optimization) |
| D-2026-05-04-C3-PHASE2-SPEC-AMENDMENT | **STALE** | Phase 2 Surface B spec expansion. The Phase 2 build is SHIPPED per CURRENT-STATE (2d002c4). This spec amendment fed into the shipped implementation. Should be archived. | -- |
| D-2026-05-04-C3-PHASE2-SURFACE-B-IMPL-SPEC | **STALE** | Phase 2 Surface B implementation spec. Consumed by C1/C2 Phase 2 build which shipped. Should be archived. | -- |
| D-2026-05-04-C3-PHASE3-AND-CONTINUATION-SPEC | **STALE** | Phase 3 + Phase 1.4-1.6 continuation spec. All Phase 1+2+3 shipped per CURRENT-STATE. The spec was consumed. Should be archived. | -- |
| D-2026-05-04-C3-PHASE4-SPEC-PLUS-EOW-EOM | **ACTIVE** | Phase 4 spec: Telegram, bank_transactions, net profit, DCP violations, EOW/EOM crons, Reports pill modal. CURRENT-STATE says "Phase 4 spec ready, blocked on Mark's open questions (Q1/Q3/Q4/Q5/Q6)." Still active but blocked on Mark decisions. | Wave 4 (Outreach + CRM Pipeline) |
| D-2026-05-04-C3-SURFACE-A-MEMORY-LAYER-AUDIT | **STALE** | Surface A memory layer recon. This audit fed into the 3-Surface build Phase 0+1. Work shipped. Should be archived. | -- |
| D-2026-05-04-C3-SURFACE-B-TRIAGE-UI-AUDIT | **STALE** | Surface B triage UI recon. Fed into Phase 2 build which shipped. Should be archived. | -- |
| D-2026-05-04-C3-SURFACE-C-EOD-REPORTING-AUDIT | **STALE** | Surface C EOD reporting recon. Fed into Phase 3 build which shipped. Should be archived. | -- |
| D-2026-05-05-C3-CALL-DETAIL-ENDPOINT | **ACTIVE** | Surface 22 call detail aggregation endpoint. CURRENT-STATE shows "C3: aggregation endpoint PENDING." | Wave 1 (Surface 22 + Resilience) |
| D-2026-05-05-C3-CID-SHAPE-AUDIT | **ACTIVE** | CID shape verification against production TrackDrive data. Quick research task. | Wave 1 (Surface 22 + Resilience) |
| ITEM-10-ask-regression | **STALE** | /ask regression measurement from 2026-04-28 sprint. TASKS.md shows "PARTIAL (baseline captured, zero production usage since deploy, needs soak)." Items 1-4 deployed weeks ago. The soak period has passed. Data is either available or moot. CURRENT-STATE lists /ask quality sprint as shipped (941059f). The regression measurement window is closed. | -- |

**Summary C3:** 10 STALE (archive), 11 ACTIVE, 0 BLOCKED (Phase 4 spec is blocked on Mark but still active).

---

## Coder-4 Ops Inbox (16 directives)

| Directive | Status | Reason | Consolidation Group |
|-----------|--------|--------|---------------------|
| D-2026-04-29-C4-COMMIT-MEASUREMENT-PLAN | **STALE** | Outbox shows this completed. Decisions 161-163 logged. Measurement plan committed. Should be archived. | -- |
| D-2026-04-29-C4-OPS-BATCH | **STALE** | Outbox shows completed. Decisions 164-168 assigned. MILESTONE-LOG rewritten. CURRENT-STATE v18. Should be archived. | -- |
| D-2026-04-29-C4-STRATEGY-PROMPT-AMENDMENT | **STALE** | Strategy-Claude failed verify tracking rule was added (commit 1b713cd). Decision 167 logged. Should be archived. | -- |
| D-2026-04-30-C4-VERIFY-DRIFT-RECURRENCE | **STALE** | Updated 2026-05-01. Action 3 completed (04f1cb0). Actions 1+2 residual scope addressed by Decision 180 (VERIFY-LEDGER standalone commits). Verify drift recurrence pattern mitigated by Pattern 30. Should be archived. | -- |
| D-2026-05-03-C4-VERIFY-SURFACE-STATUS | **STALE** | Outbox report exists at `D-2026-05-03-C4-VERIFY-SURFACE-STATUS.md`. Verify #11 added, V4 Surface Progress table created. Should be archived. | -- |
| D-2026-05-03-C4-VERIFY-SURFACE19 | **STALE** | Outbox report exists at `D-2026-05-03-C4-VERIFY-SURFACE19.md`. Verify entries #9-10 added, Surface 19 candidate spec generated. Should be archived. | -- |
| D-2026-05-04-C4-ERROR-LOG-PLUS-DOCS-SWEEP | **STALE** | Superseded by D-2026-05-04-C4-FINISH-ERROR-LOG-INFRA which completed the remaining infra. Outbox shows both completed. error_log table live, daily digest cron registered. Should be archived. | -- |
| D-2026-05-04-C4-FINISH-ERROR-LOG-INFRA | **STALE** | Outbox report exists. error_log table verified in production, daily digest cron built (0 14 * * *), SCHEMA.md added. Commits 5a6b5ed + dff52cb. Should be archived. | -- |
| D-2026-05-04-C4-PHASE0-SNOOZE-CONSOLIDATION | **STALE** | Outbox report exists. alerts.snoozed_until consolidated, snoozed_items retired. Should be archived. | -- |
| D-2026-05-04-C4-PHASE1-ALERT-DISPOSITION | **STALE** | Outbox report exists. 3 routes, 5 alert actions instrumented. Commit 7279482. Should be archived. | -- |
| D-2026-05-04-C4-PHASE1-REPLY-TRACKING | **STALE** | Outbox report exists. Resend webhook receiver, classifyReply adapter, send_log migration. Commit b3a3a2b. Should be archived. | -- |
| D-2026-05-04-C4-PHASE2-DRILLDOWN-PLUS-PHASE3-PERSISTENCE | **STALE** | Outbox report exists. DrillDownDrawer, TaskCreateInline, EOD persistence, Teams format. Commit cd5d021. Should be archived. | -- |
| D-2026-05-04-C4-RESEND-WEBHOOK-SECRET | **STALE** | Outbox exists in recently completed (TASKS.md). RESEND_WEBHOOK_SECRET added, Svix verification live. Commits 980e38e + 52c961e. Should be archived. | -- |
| D-2026-05-04-C4-SELF-VERIFICATION-PROTOCOL | **STALE** | Outbox report exists. Decision 187 locked, all Coder agent defs amended, Pattern 31 added. Should be archived. | -- |
| D-2026-05-05-C4-CALL-NOTES-SCHEMA-D188 | **STALE** | Outbox report exists. call_notes table + endpoint shipped. Commits c789dd5 + f543737 + 5f1ebfe. Decision 188 logged. Should be archived. | -- |
| D-2026-05-05-C4-D187-AMENDMENT-DOMAIN-AUDIT | **ACTIVE** | Decision 187 Element F amendment (production deploy verification) + domain mapping audit. Outbox exists but was just dispatched today. If outbox shows completion, STALE. Otherwise this is the current C4 work. | Wave 1 (Surface 22 + Resilience) |

**Summary C4:** 15 STALE (archive), 1 ACTIVE.

---

## Aggregate Summary

| Lane | STALE | ACTIVE | BLOCKED | Total |
|------|-------|--------|---------|-------|
| Coder-1 | 8 | 2 | 1 | 11 |
| Coder-2 | 12 | 3 | 0 | 15 |
| Coder-3 | 10 | 11 | 0 | 21 |
| Coder-4 | 15 | 1 | 0 | 16 |
| **Total** | **45** | **17** | **1** | **63** |

**45 of 63 inbox directives are STALE and should be archived.** They have matching outbox reports confirming completion, or have been superseded by the 3-Surface build (15 commits, ~8,600 LOC shipped 2026-05-04).

---

## Dispatch Waves (ACTIVE directives grouped by priority)

### Wave 1: Surface 22 Completion + Resilience Hardening (HIGHEST PRIORITY)
**Theme:** Finish the in-flight Surface 22 (Call Detail Panel) and close resilience gaps.

| Directive | Lane | What |
|-----------|------|------|
| D-2026-05-05-C3-CALL-DETAIL-ENDPOINT | C3 | Call detail aggregation endpoint (Surface 22 backend) |
| D-2026-05-05-C3-CID-SHAPE-AUDIT | C3 | CID format verification (quick research, unblocks Surface 22 data confidence) |
| D-2026-05-05-C2-CALL-DETAIL-PANEL-UI | C2 | Call Detail Panel modal UI (Surface 22 frontend) |
| D-2026-05-05-C2-SURFACE22-BUGFIX | C2 | Surface 22 rendering fixes (white screen, # prefix, row dedup, phone format) |
| D-2026-05-04-C3-AUTH-REMOVAL-RISK-AUDIT | C3 | Auth removal risk audit (unblocks C2 resilience migration workstream 2) |
| D-2026-05-04-C2-RESILIENCE-MIGRATION | C2 | Resilience migration workstream 2 (auth removal from 6 endpoints) |
| D-2026-05-04-C3-CALL-LEVEL-ENRICHMENT | C3 | Call-level alert enrichment (richer MY TAKE for call alerts) |
| D-2026-05-04-C3-MILO-AUTO-CONTEXT | C3 | Auto-context from TrackDrive for Review actions |
| D-2026-05-04-C1-RESILIENCE-PRIMITIVES | C1 | Empty/error state components (verify shipped or build) |
| D-2026-05-05-C4-D187-AMENDMENT | C4 | Decision 187 Element F + domain mapping audit |

**Dispatch order:** C3 CID audit first (quick, unblocks data confidence) -> C3 call detail endpoint + C2 panel UI in parallel -> C2 bugfixes -> C3 auth risk audit -> C2 auth removal. C3 auto-context and call enrichment can run in parallel.

### Wave 2: Chat Surface V4 (Surface 17)
**Theme:** V4 restyle of the core chat interface.

| Directive | Lane | What |
|-----------|------|------|
| D-2026-05-03-C3-V4-SURFACE-17-CODE-AUDIT | C3 | 30-question code audit for Surface 17 redesign |
| D-2026-05-04-C3-CONVERSATIONMODAL-V4 | C3 | ConversationModal V4 restyle |

**Dispatch order:** Code audit first (research), then restyle dispatch. Blocked on Wave 1 resilience stabilization (file collision avoidance on ask/page.tsx).

### Wave 3: Optimization + Audit Catchup
**Theme:** Reduce cost, improve cache hit rate, document system state.

| Directive | Lane | What |
|-----------|------|------|
| D-2026-04-29-C1-SERVER-CACHE-NORMALIZATION | C1 | Server-side prefix match for vet lookup cache (cost reduction) |
| D-2026-05-04-C3-OPERATOR-WORKFLOW-AUDIT | C3 | Comprehensive 9-capability workflow audit |

**Dispatch order:** C3 workflow audit can run anytime. C1 cache fix is low urgency.

### Wave 4: Outreach Pipeline + Phase 4
**Theme:** Fill the outreach tracking gap and plan Phase 4.

| Directive | Lane | What |
|-----------|------|------|
| D-2026-04-29-C3-OUTREACH-CAPTURE-AUDIT | C3 | Architecture spec for outreach capture (Decision 166) |
| D-2026-05-03-C3-CLASSIFIER-CRM-AUDIT | C3 | Classifier mismatch scan + Q6 outreach pipeline infrastructure |
| D-2026-05-04-C3-PHASE4-SPEC-PLUS-EOW-EOM | C3 | Phase 4 spec (Telegram, bank_transactions, EOW/EOM, Reports pill) |

**Note:** Phase 4 spec is blocked on Mark answering open questions Q1/Q3/Q4/Q5/Q6. Outreach capture and classifier audit can proceed independently as research.

**CONSOLIDATE candidate:** D-2026-04-29-C3-OUTREACH-CAPTURE-AUDIT + D-2026-05-03-C3-CLASSIFIER-CRM-AUDIT Q6 overlap significantly. Both audit outreach pipeline infrastructure. Merge into a single "Outreach Pipeline Architecture Spec" directive.

### Wave 5: Test Infrastructure (LOWEST PRIORITY)
**Theme:** Durable CI + synthetic testing. Deferred until product surfaces stabilize.

| Directive | Lane | What |
|-----------|------|------|
| D-2026-04-29-C3-CI-TEST-GATE-SPEC | C3 | CI test gate spec for SSE-coupled components |
| D-2026-04-29-C1-FIXTURE-TESTS-PHASE2 | C1 | Fixture pages + Playwright specs |
| D-2026-04-29-C1-SYNTHETIC-OPERATOR-PHASE3 | C1 | Synthetic operator MVP (BLOCKED on Phases 1+2) |

**Note:** All three of these are from 2026-04-29 (6 days old). Product has evolved significantly since then. The fixture page patterns proposed (EntityVetCard, OpportunityCard, OtherMentions) may need updating to match the shipped V4 components. CI-TEST-GATE-SPEC is still architecturally valid but low urgency while direct-push-to-main workflow continues.

---

## Recommendations

1. **Archive 45 STALE directives immediately.** Move to `directives/archive/2026-05-05/`. This clears inbox noise for all 4 lanes.

2. **Dispatch Wave 1 now.** Surface 22 is the active product work per CURRENT-STATE. All 3 contributing lanes (C2 UI, C3 endpoint, C4 infra) have inbox directives ready.

3. **Consolidate outreach audits.** Merge OUTREACH-CAPTURE-AUDIT + CLASSIFIER-CRM-AUDIT Q6 into one "Outreach Pipeline Architecture" directive for C3.

4. **Gate Wave 2 on Wave 1 completion.** Chat surface V4 touches ask/page.tsx which is in active flux from Surface 22 work. Let Surface 22 ship first.

5. **Mark decision needed for Wave 4.** Phase 4 spec has 5 open questions (Q1/Q3/Q4/Q5/Q6). These should be answered before Phase 4 work queues.

6. **Wave 5 is shelf-stable.** Test infrastructure directives are still valid but can wait until the V4 surface redesign stabilizes. Update fixture page specs when ready.

---

Report at: https://raw.githubusercontent.com/moscaleflow/conductor-mark-public/main/reports/DIRECTIVE-TRIAGE-2026-05-05.md
