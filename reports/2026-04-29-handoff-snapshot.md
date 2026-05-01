# Handoff Snapshot — 2026-04-29

Cold PM pickup format. Read this to resume coordination from zero context.

---

## Lane states

| Coder | Status | Current work |
|---|---|---|
| Coder-1 | **active** | Vet collapse fix #3 (trace-first debug, 8aea054). Queued behind: research_list render, Amendment D render |
| Coder-2 | **active** | Chat logs archive cron, then theleadpenguin.com domain detach |
| Coder-3 | **active** | Outreach capture audit + stress-test |
| Coder-4 | **active** | This ops batch (D-2026-04-29-C4-OPS-BATCH) |

---

## In-flight directives with gates

### Verify status uses three-state distinction (Decision 167):
- **shipped not yet verified** — code deployed, no visual verification attempted
- **shipped and verified** — code deployed + visual verification passed
- **shipped prior verify FAILED** — code deployed + visual verification attempted and failed

| Directive | Coder | Gate | Status |
|---|---|---|---|
| TrackDrive prompt wiring (de26f3b) | C1/C2 | mark-visual-verify | **shipped not yet verified** |
| Phase 1 opportunity matching (5212b44) | C1 | mark-visual-verify | **shipped not yet verified** |
| Vet collapse fix #3 | C1 | mark-visual-verify | **shipped prior verify FAILED** (549037f, 933c8a9). Fix #3 trace-first in progress. |
| research_list render | C1 | mark-visual-verify | **not started** |
| Amendment D render | C1 | mark-visual-verify | **not started** |
| Outreach capture audit | C3 | mark-approval | **running** |
| CI test gate spec (98ceec0) | C3 | mark-approval | **complete, pending review** |

---

## Outstanding Mark approvals

1. **CI test gate spec** — Coder-3 outbox (conductor-mark 98ceec0). Defines SSE-coupled component testing requirements.
2. **Outreach capture audit** — pending Coder-3 completion, then Mark approval of recommended architecture.
3. **TrackDrive prompt wiring visual verify** — shipped (de26f3b), needs visual confirm that buyer/publisher pills display rate data.
4. **Phase 1 opportunity matching visual verify** — shipped (5212b44), needs visual confirm of OpportunityCard rendering.

---

## Mark pending action items (from sprint)

- Add MILO_OUTREACH_API_KEY + MILO_OUTREACH_API_URL to milo-for-ppc production Vercel env
- Add campaign_ops to Malvin's user_profiles.roles for contract pills to appear

---

## Recent decisions (today)

All assigned by Coder-4 ops batch sweep:

| # | Decision | Status |
|---|---|---|
| 164 | TrackDrive Option A — wire existing payout/rate data into pills (32290eb + de26f3b) | LOCKED |
| 165 | research_list density spec — collapsed sections, takeaway-first (084954b) | LOCKED |
| 166 | Outreach capture architectural gap — spec phase required | PROPOSED |
| 167 | Failed verify tracking — no silent overwriting of prior failed verifies (1b713cd) | LOCKED |
| 168 | Pattern 24 SSE testing enforcement — promoted to coder-1 agent def (f1e7033) | LOCKED |

Earlier today (assigned in prior sweeps):

| # | Decision | Status |
|---|---|---|
| 161 | 30-day vet freshness window — collapsed-by-default UX | DECIDED |
| 162 | TLP tenant domain set to milo-outreach.vercel.app (CAN-SPAM fix) | LOCKED |
| 163 | theleadpenguin.com domain — Option B for unsubscribe URL routing | DECIDED |

---

## Recent process amendments

1. **Pattern 24 enforcement (Decision 168, f1e7033).** SSE-coupled component testing moved from drift guard documentation to mandatory pre-commit check in coder-1 agent definition. Any component rendering SSE data must have end-to-end data path verified before commit.
2. **Failed verify tracking (Decision 167, 1b713cd).** Strategy-Claude prompt amended: failed verify status persists until a new fix ships AND passes. Three-state distinction enforced.
3. **research_list density spec approved (Decision 165, 084954b).** KEY TAKEAWAY + ACTIONABLE ITEMS expanded, detail sections collapsed by default.
4. **No-time-estimates expansion (Decision 156).** Lane reports use status only (running/complete/blocked). No duration hints, no relative time estimates.

---

## Critical context warnings

1. **Vet collapse has failed 2 fix attempts (549037f, 933c8a9).** Fix #3 is trace-first -- trace instrumentation committed (8aea054) before any code change. Do NOT accept a fix without trace evidence showing the SSE data path propagation.
2. **Strategy-Claude reported failed verify as green.** Decision 167 prevents recurrence. When reviewing status, verify the three-state distinction is maintained for all verify gates.
3. **TrackDrive data enrichment shipped but prompt wiring not yet visual-verified.** Commit de26f3b wired contractedRates into pill context. Needs visual confirm that buyer/publisher pills actually display rate data.
4. **CAN-SPAM resolved via Option B (tenant domain update).** theleadpenguin.com domain detach approved, queued for Coder-2. GoHighLevel marketing page unaffected.
5. **Phase 1 opportunity matching shipped (5212b44).** Includes signal classifier, CRM/pipeline cross-reference, OpportunityCard. GIN index applied. Measurement instrumentation live (746e0e6). Needs visual verify.
6. **Two Strategy-Claude prompt amendments in one day.** Prompt-rule layer still being calibrated. Current sprint surface area exposing coordination edge cases. Each amendment addresses a real observed failure.

---

## What shipped today (summary)

- Measurement instrumentation (746e0e6)
- TrackDrive enrichment (32290eb) + prompt wiring (de26f3b) — Decision 164
- Phase 1 opportunity matching (5212b44) + GIN index (311c40f)
- StructuredResponseCard migration for all 3 pills (08feab9, 480c275, ea8f146, 4ee0524)
- CAN-SPAM fix (Decision 162-163)
- research_list density spec (084954b) — Decision 165
- Pattern 24 enforcement in agent def (f1e7033) — Decision 168
- Failed verify tracking in Strategy-Claude prompt (1b713cd) — Decision 167
- Vet collapse trace instrumentation (8aea054)

---

## What is still broken

- Vet collapse: 2 failed fixes. Fix #3 trace-first in progress.

Report at: https://raw.githubusercontent.com/moscaleflow/conductor-mark-public/main/reports/2026-04-29-handoff-snapshot.md
