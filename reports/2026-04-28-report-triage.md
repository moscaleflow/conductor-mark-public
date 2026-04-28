# Report Triage — 2026-04-28

**Author:** Coder-4 (Ops Lane)
**Reports triaged:** 28 (18 Coder-1, 9 Coder-3, 1 Coder-4)
**Repos affected:** milo-for-ppc (primary), milo-engine, milo-outreach (audit-only)

---

## Shipped clean (no Mark attention needed)

These are COMPLETE with passing builds, no schema ambiguity, and no product decisions deferred.

1. **C1-003 Vet card dismiss button** (cd0c4ba) -- X button in EntityVetCard top-right corner, session-only dismiss with fade transition. No data deletion. Build clean.

2. **C1-007 ?dev=true middleware bypass removed** (266f4e5) -- Security fix. Deleted the 3-line block that let any visitor bypass ALL middleware auth via `?dev=true`. Full grep confirms zero remaining instances. Build clean.

3. **C1-008 Tautological self-reference bleed fix** (2a66437) -- Single STYLE paragraph added to vet prompt. Tells model to distinguish paste allegations from independently researched findings. Build clean.

4. **C1-010 Pipeline board UX (D-013)** -- NO CODE CHANGE NEEDED. All 4 reported issues (horizontal scroll, chat FAB hidden, notifications hidden, header text) were already fixed in current codebase. Gate: mark-visual-verify (Mark should open /pipeline-board and confirm).

5. **C1-011 Pipeline board cleanup A+C** -- Only remaining item (drag guard when detail panel open) now implemented. All other items already applied in prior sessions. A3 seed data cleanup still blocked on entity names from Mark (see "blocked" section). Build clean.

6. **C1-012 EntityDetailPanel vet_results wiring** -- ALREADY_COMPLETE. API route, TypeScript interface, and Vet Intelligence rendering section all present in current codebase. No changes needed.

7. **C1-013 Collapsible sections** -- ALREADY_COMPLETE. CollapsibleSection component, expandedSections state, full-bar clickable with chevron rotation, all 15 sections wrapped. Actions expanded by default.

8. **C1-014 Blacklisted/dormant pipeline views** -- ALREADY_COMPLETE. isFlatView toggle, blacklisted/dormant options in view dropdown, count badges all present.

9. **C1-015 Fuzzy search** -- ALREADY_COMPLETE. Fuse.js fuzzy search on both pipeline pages with memoized instances.

10. **C1-016 Blacklist consolidation** -- ALREADY_COMPLETE. /api/blacklist reads/writes blacklist_entries, soft-deletes with is_active=false.

11. **C1-017 DnD fix** -- ALREADY_COMPLETE. All 3 fixes applied: isDropAnimating check, isDragDisabled when selectedEntity open, DragDropContext keyed on filter/sort/search state.

12. **C1-018 Resend outreach send** -- ALREADY_COMPLETE. /api/outreach/send route exists. EntityDetailPanel has handleSendOutreach with send/sent/error states.

13. **C4-001 Archive stale relay** -- 9 bootstrap relay files archived. TASKS.md updated. Dashboard zeroed out stale reports.

---

## Needs Mark's attention

These require a decision, visual verification, or strategic judgment.

### Security (review urgently)

14. **C1-006 auth-server.ts query-param bypass removed** (5298b04) -- CRITICAL security fix. Removed `?userId=X&isAdmin=true` query-param fallback from resolveRequestUser() that allowed unauthenticated privilege escalation on 5 API routes. Also fixed /api/notifications/count missing 401 guard and /api/operator internal fetch to use cookie forwarding instead of query-param auth.
    - **Decision needed:** This commit is LOCAL ONLY, not pushed. Mark should push this immediately.
    - **Additional flag in report:** middleware `?dev=true` bypass was noted as separate concern -- already fixed in C1-007 above.

### Schema changes (review before push)

15. **C1-009 Entity profile step 1 -- Vetted column** (bdbad4d) -- Applied migration `20260425100001_fix_pipeline_source_and_stages.sql` to shared Supabase. Adds `milo-vet` and `milo_auto` to prospect_pipeline source CHECK constraint. Purple Vetted column added as leftmost column on pipeline board. `vetted` added to stage API VALID_STAGES.
    - **Decision needed:** Migration already applied to shared Supabase. Commit is local only. Mark should confirm the UI placement (leftmost column) and push.
    - **Gate:** mark-visual-verify. Screenshot at `/Users/markymark/Desktop/pipeline-vetted-column-2026-04-25.png`.

### Product decisions pending

16. **C1-004 Vet verdict quality fix** (b4dd3d9) -- Prompt change allowing associated entity signals to inform target verdict without reporting them as findings. Structural change is correct, but live API re-vet was NOT run to confirm model behavior change.
    - **Decision needed:** Should a live re-vet be run against the "Matthew Barner / Top Class Leads LLC" test case to confirm the model now downgrades Barner's verdict?

17. **C1-002 Vet entity-mixing fix** (9ce9c26) -- Prompt isolation + associated_entities structured field. Production test shows bleed eliminated for the 3-entity test case. Michael correctly classified as secondary (no standalone card). 
    - **Decision needed:** The directive expected Michael to get his own primary card. Coder-1 chose to let Haiku classify Michael as secondary (no last name, "his guy" reference). Mark should confirm this is the desired behavior, or request extraction rules adjustment.
    - **Gate:** mark-visual-verify. Screenshot at `/Users/markymark/Desktop/vet-entity-isolation-fix-2026-04-25.png`.

18. **C1-019 E2E test infrastructure + pipeline board tests** -- 16/16 tests pass against localhost, 1/16 against production (data-testid attributes not deployed yet). Playwright storageState auth, 4 test files, 11 data-testid attributes added to pipeline-board.
    - **Decision needed:** DB stage constraint mismatch -- seed uses only currently-valid stages (vetted, outreach, qualifying, activation). Migration `20260425000001_vet_results.sql` adds more stages but has not been applied to production. Mark should confirm when this migration ships.

### Audit findings requiring product decisions

19. **C3-001 Vet bleed audit** -- 79% of multi-entity vets had some cross-entity contamination; 21% SEVERE (verdict/recommendation contaminated). Coder-1's fix addresses ~58% of bleed instances. Remaining gap: tautological self-reference (6 instances) where model restates paste context as findings. Also found: `vet_results.chat_log_id` is NULL for ALL rows (backfill was fire-and-forget with `void`), `response_text` truncated at 10K chars.
    - **Decision needed:** The remaining tautological bleed was addressed by C1-008, but both fixes need live regression testing against the 7 test cases defined in this audit.

20. **C3-006 Blacklist + fuzzy search audit** -- Found two-table problem: `blacklist` table vs `blacklist_entries` table. "Flag as Scam" writes to `blacklist` but screening reads from `blacklist_entries`. Data drift risk.
    - **Decision needed:** Should the `blacklist` table be retired entirely in favor of `blacklist_entries`? Report recommends yes. (Note: C1-016 report says consolidation is ALREADY_COMPLETE -- verify these are reconciled.)

21. **C3-007 DnD audit** -- Root cause analysis for card overlap: same-column drop silently discarded, custom `transition` property overrides library's drop animation, selectedEntity guard swallows drag events. All 4 fixes now applied per C1-017.
    - **Decision needed:** Should manual in-column card reorder be supported? Report recommends NO (sort-derived order is correct for this use case). Mark should confirm.

22. **C3-008 Outreach + Resend integration spec** -- Full audit of draft/send flow. Resend already installed (v6.12.2). No /api/outreach/send route existed at time of audit (now exists per C1-018). 
    - **Decision needed:** Verify `theleadpenguin.com` domain is verified in Resend dashboard. Verify RESEND_API_KEY is set in production .env.

23. **C3-009 Onboarding flow audit** -- 75% wired, 25% integration plumbing. Found 2 CONFIRMED BUGS:
    - **BUG 1 (CRITICAL):** Activation gate bypass via PATCH route -- operator can drag entity directly from Onboarding to Active on kanban board, bypassing MSA/tax/IO document requirement. Gate only exists on POST route.
    - **BUG 2 (CONFIRMED):** handleGenerateDocs sends PATCH to /api/pipeline/{id}/stage but that route only exports POST. Next.js returns 405 silently. Stage advance fails but operator sees "Documents generated" success message.
    - **Decision needed:** Both bugs need prioritized fixes. Also: DID provisioning and Config validation are placeholders (steps 8-9 of 12), buyer onboarding path does not exist.

24. **C3-020 Sales pipeline audit** -- Comprehensive V4 Sales Drawer readiness audit. Found: no @milo/sales or @milo/outreach packages exist. prospect_pipeline is sole source of truth. No "won/lost" deal semantics. No calendar integration.
    - **Decisions needed (3):**
      - V4 Sales Drawer should read from prospect_pipeline, not @milo/crm counterparties (they drift)
      - "Move to Dormant" should accept a reason field (new migration)
      - Outreach compose should be extracted from the 4,477-line EntityDetailPanel into a composable component

### Research specs (ready for Coder-1 when prioritized)

25. **C3-002 Entity profile + kanban directives** -- Coder-1-ready spec for "Vetted column on pipeline board + source constraint fix." Already executed by C1-009 above. Remaining items (entity type mapping for non-publisher/buyer, column config extraction) deferred to future directives.

26. **C3-004 Pipeline three directive specs** -- Three detailed Coder-1-ready specs (A: board cleanup, B: EntityDetailPanel vet wiring, C: board polish). A and C largely executed. B executed (C1-012). Seed data cleanup (A3) still blocked on entity names from Mark.

27. **C3-010 E2E test audit** -- Full infrastructure assessment. Recommended storageState auth, Playwright config upgrade, npm test scripts, data-testid attributes. All implemented by C1-019. 
    - **Decision needed:** Adopt password-auth test user as standard E2E strategy (production is magic-link only). This is the only password-auth account in the system.

---

## Stuck or blocked

28. **C1-011 Seed data cleanup (A3)** -- BLOCKED. Coder-1 needs the 4 entity names or UUIDs for d146 seed data that was inserted directly into shared Supabase. No hardcoded IDs in the codebase. Mark must provide these.

---

## Stats

| Metric | Count |
|--------|-------|
| Total reports triaged | 28 |
| Shipped clean (no action) | 13 |
| Needs Mark's attention | 14 |
| Blocked | 1 |
| Coder-1 commits (unique SHAs) | 9 (9ce9c26, cd0c4ba, b4dd3d9, cd138bf, 5298b04, 266f4e5, 2a66437, bdbad4d, + E2E infra) |
| ALREADY_COMPLETE (no code change) | 7 (D010, D012-D018) |
| Confirmed bugs found | 5 (activation gate bypass, PATCH/POST method mismatch, chat_log_id backfill, response_text truncation, entity dedup) |
| Security fixes | 2 (query-param auth bypass, ?dev=true bypass) |
| Schema migrations applied | 1 (source CHECK constraint) |
| Build status | All PASS clean |
| Live API verification pending | 2 (verdict quality prompt, tautological bleed prompt) |
| Visual verification gates | 3 (vet entity isolation, vet card dismiss, pipeline vetted column) |

### Priority ranking for Mark's review

1. **URGENT:** Push C1-006 auth security fix (5298b04) -- unauthenticated privilege escalation is live in production
2. **URGENT:** Fix activation gate bypass (C3-009 BUG 1) -- operators can skip document checks via kanban drag
3. **HIGH:** Fix handleGenerateDocs PATCH/POST mismatch (C3-009 BUG 2) -- onboarding stage advance silently fails
4. **HIGH:** Push C1-009 entity profile step 1 (migration already applied to DB)
5. **MEDIUM:** Run live regression tests for vet prompt changes (C1-002, C1-004, C1-008)
6. **MEDIUM:** Retire `blacklist` table in favor of `blacklist_entries` (C3-006)
7. **LOW:** Provide seed data entity names for cleanup (C1-011 A3)
8. **LOW:** Confirm Michael-as-secondary behavior (C1-002)
9. **LOW:** V4 Sales Drawer architecture decisions (C3-020)

---

Report at: https://raw.githubusercontent.com/moscaleflow/conductor-mark-public/main/reports/2026-04-28-report-triage.md
