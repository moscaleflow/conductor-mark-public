# V4 Shell Refinement: Footer Sync Status with Troubleshoot Escalation

> Coder-3 Spec | D152 | For Coder-1 execution
> Source: D85 lock — "Footer 'Live · synced' status escalates to Milo troubleshoot card if sync degrades"
> Build order: 2 of 6

---

## User-observable behavior

**Before:** The operator page has no footer. No visibility into whether background systems (crons, TrackDrive webhooks, health checks) are healthy.

**After:** A fixed footer bar at the bottom of `/operator` shows sync health:

**Healthy state:**
- Left-aligned: green dot + "Live · synced" in muted text (#636366, 12px)
- Right-aligned: timestamp of last health check ("Checked 3m ago")
- Footer background: transparent or `#000` to match page

**Degraded state:**
- Left-aligned: amber dot + "Degraded · 2 services slow" in amber text (#ff9f0a)
- "Troubleshoot" link appears right-aligned
- Clicking "Troubleshoot" dispatches `milo:open-chat-with-message` with: "Milo, something looks degraded — run a health check and tell me what's wrong."

**Critical state:**
- Left-aligned: red dot + "Offline · TrackDrive API down" in red text (#ff453a)
- "Troubleshoot" link becomes more prominent (white text, underlined)
- Same click behavior — opens Milo with diagnostic prompt

**Acceptance criteria:**
- Footer is visible on `/operator` for both admin and non-admin users
- Footer fetches `/api/health-check` on mount and every 5 minutes
- Footer does not block page render (async fetch, shows "Checking..." until first response)
- Dot color matches severity: green (#30d158) / amber (#ff9f0a) / red (#ff453a)
- "Troubleshoot" only appears when status is degraded or critical
- Clicking "Troubleshoot" opens Milo conversation (same `openChatWith` pattern as pill clicks)
- Footer is sticky at bottom of viewport (not bottom of content)
- Footer does not overlap PillDrawer (z-index below drawer's z-index)
- Mobile: footer text truncates gracefully; "Troubleshoot" stays visible

---

## File-level scope

| File | Change | ~LOC |
|---|---|---|
| `src/components/operator/SyncFooter.tsx` | **New file.** Renders footer bar. Fetches `/api/health-check` on mount + interval. Maps `HealthReport.overall` to dot color + label. Renders "Troubleshoot" link on degraded/critical. | ~80 |
| `src/app/operator/page.tsx` | Import and render `<SyncFooter />` below the main content div, outside the `maxWidth: 860` container. Pass `onTroubleshoot` callback that calls `openChatWith`. | ~5 |

**Total: ~85 LOC across 2 files.**

---

## Data dependencies

**Existing API — no new schema or routes needed.**

`GET /api/health-check` already returns:
```typescript
interface HealthReport {
  timestamp: string;
  overall: 'healthy' | 'degraded' | 'critical';
  quietHours: boolean;
  checks: HealthCheck[];  // 7 checks: MOP, TrackDrive, ConvoQC, sync freshness, enrichment, briefing, Teams
  alertsSent: boolean;
}
```

The footer only needs `overall` and `checks[].name` + `checks[].status` for the label text. No new data.

**Auth consideration:** `/api/health-check` currently has no auth gate — it's called by the cron and returns structured data. The footer calls it from the client. If auth is needed, add a session check. **Assumption: no auth gate needed** — the response contains no sensitive data (just service names and up/down status).

---

## Visual requirements

**No HTML design reference exists.** D144 never landed.

**Assumed layout (flagged for Mark):**
```
┌────────────────────────────────────────────────────┐
│ ● Live · synced                     Checked 3m ago │
└────────────────────────────────────────────────────┘
```
Degraded:
```
┌────────────────────────────────────────────────────┐
│ ● Degraded · 2 services slow          Troubleshoot │
└────────────────────────────────────────────────────┘
```

- Height: 36px
- Background: `#000` (matches page)
- Border-top: `0.5px solid #1c1c1e` (subtle separator)
- Fixed to bottom of viewport: `position: fixed; bottom: 0; left: 0; right: 0`
- z-index: below PillDrawer (PillDrawer is z-50; footer should be z-10)

---

## Blast radius

**Low.** New component + 5 lines in operator page. No API changes. No schema changes.

**Risk 1:** `/api/health-check` runs 7 parallel checks including external API pings (MOP, TrackDrive, ConvoQC). Calling it every 5 minutes from every open operator tab could amplify external API traffic. **Mitigation:** Add a client-side dedup (only fetch if last fetch was >4 minutes ago) and consider a lightweight `/api/health-check/summary` endpoint that returns cached results from the last cron run instead of re-running all checks.

**Risk 2:** Health check returns `critical` when any service is down. An operator seeing "Offline" might panic when it's just the ConvoQC API being slow. **Mitigation:** The label should name the specific degraded service, not just say "Offline."

---

## Build order rationale

Ranked 2 of 6 because:
- High visibility — Mark specifically called this out in the D85 design session
- No design reference needed — the footer is simple enough to ship with the assumed layout and iterate
- Zero dependencies on other refinements
- The health check API already exists and is production-proven
- Provides immediate operational value (operators see system health without asking Milo)

---

## Open questions

**Q1 (assumed answer flagged):** Should the footer poll `/api/health-check` directly (re-runs all 7 checks) or read a cached summary from the last cron run? **Assumed: poll directly** for v4.0 simplicity. If external API amplification becomes a concern, add a `/api/health-check/summary` cache endpoint in v4.1.

**Q2 (assumed answer flagged):** Should the footer appear on admin executive view (Top5Frame) or only on operator briefing view? **Assumed: both** — Mark sees the admin view and should also see sync health.
