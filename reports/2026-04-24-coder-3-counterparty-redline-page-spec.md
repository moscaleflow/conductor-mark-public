# Counterparty-Facing Redline Page Spec

**Coder-3 Research | 2026-04-24**
**Purpose:** Spec for Coder-1 to implement as Vet wire-up Part 2. Defines the page a counterparty (e.g., Randy at Granite Media) sees when they click a redline review link from TLP.
**No code. Pure spec.**

---

## A. AUTH MODEL

### Token-only access, no login

The counterparty has no account on justmilo.app. Authentication is via a one-time token embedded in the URL. Two URL formats resolve to the same page:

| Format | Source | Example |
|---|---|---|
| Full token URL | `@milo/contract-negotiation` `sendRound()` → `reviewUrl` | `https://tlp.justmilo.app/review/{uuid-token}` |
| Short code URL | `@milo/contract-negotiation` `sendRound()` → `shortUrl` | `https://tlp.justmilo.app/r/{8-char-code}` |

**Resolution flow:**
1. Short code URL → API route resolves `negotiation_links.short_code` → redirect to full token URL
2. Full token URL → page calls `client.loadReview(token)` which:
   - Looks up `negotiation_links` by token + tenant_id
   - Checks expiration (`expires_at < now()` → expired)
   - Increments `view_count`, updates `first_viewed_at`/`last_viewed_at`
   - Returns `ReviewData` (negotiation, analysis, broker decisions, prior counterparty decisions, rounds)

**State matrix:**

| Token state | Page behavior |
|---|---|
| Valid, status=`counterparty_review` | Full interactive page (decisions enabled) |
| Valid, counterparty already submitted | Read-only view with prior decisions shown, "Decisions submitted on {date}" banner |
| Expired | 410 page with Milo-voice copy |
| Invalid/not found | 404 page, no information leak |
| Negotiation cancelled | Read-only with "This negotiation was cancelled by TLP" |
| Negotiation finalized | Read-only with "This contract is finalized. Both sides agreed." |

**Expiration UX:**
```
410 GONE

"This link expired. It happens -- contracts move fast in this business."

"Need more time? Hit the button and I'll let TLP know."

[Request Extension]  (mailto: link or form → sends email to TLP operator)
```

**Already-acted-on UX:**
```
"You already sent your decisions on April 22, 2026."

"Here's what you submitted:"
[read-only list of decisions]

"TLP is reviewing. They'll be in touch."
```

**"Extend 7 days" button:**
- Visible on expired links and links expiring within 48 hours
- Click triggers `mailto:` to the operator email stored on `negotiation_records.metadata.operator_email` (or fallback to `verticalConfig.company.ownerEmail`)
- Subject: "Extension request — {counterparty_name} — {document_type}"
- Body pre-filled: "I need a few more days to review. Can you extend?"
- **OPEN QUESTION 1:** Should "Extend" write to the DB directly (self-service) or always go through TLP? Mark decides.

---

## B. PAGE HEADER

### Layout

```
+------------------------------------------------------------------+
|                                                                    |
|  Granite Media -- Buyer IO -- Version 2                           |
|  Sent by TLP on Apr 20, 2026. Expires Apr 27, 2026.              |
|                                                                    |
|  "I marked up your contract. Here's what TLP wants to change."    |
|                                                                    |
+------------------------------------------------------------------+
```

### Fields

| Field | Source | Fallback |
|---|---|---|
| Counterparty name | `ReviewData.negotiation.counterparty_name` | "Your Company" |
| Document type | `ReviewData.negotiation.document_type` → human label (`msa_buyer` → "Buyer MSA") | Raw value |
| Version | `ReviewData.negotiation.current_round` | "1" |
| Sent date | Broker round's `submitted_at` from `ReviewData.rounds` | `negotiation.created_at` |
| Expires date | `negotiation_links.expires_at` | "No expiration" |

### Milo intro copy (static, not AI-generated)

```
"I marked up your contract. Here's what TLP wants to change.
Look it over. Decide on each item. Send it back when you're done."
```

**OPEN QUESTION 2:** Should the header show TLP's logo/branding or Milo's branding? Directive says "Optional company logo / branding (defer to Part 3+ if complex)." Recommend: show `verticalConfig.company.name` ("TLP Compliance") as text, no logo upload system yet. Milo penguin icon as page favicon.

---

## C. REDLINE BODY

### Rendering strategy

The redline body is NOT raw HTML from `@milo/contract-analysis`. It is a structured issue-by-issue display, because:
1. The counterparty needs to make decisions per-issue, not per-paragraph
2. `ReviewData.analysis.issues` provides the structured data (`ContractIssue[]` from `analysis_records`)
3. `ReviewData.brokerDecisions` tells us what TLP decided on each issue — this frames the counterparty's view

### Issue display per item

Each issue from `analysis.issues` that TLP marked as `rejected` or `edited` or `counter` is shown (TLP `accepted` issues are NOT shown — the counterparty doesn't need to see clauses TLP agreed to keep).

```
+------------------------------------------------------------------+
| Section 5.2 -- Clawback Window                          [HIGH]   |
|                                                                    |
|  CURRENT TEXT (what you sent):                                     |
|  "Company may reverse any payment within 60 calendar days of      |
|   the initial payment date, for any reason."                       |
|                                                                    |
|  TLP'S PROPOSED CHANGE:                                            |
|  "Company may reverse payment within 14 calendar days of the      |
|   initial payment date, for documented quality failures only.      |
|   Leads not rejected within 14 days are final and billable."       |
|                                                                    |
|  WHY:                                                              |
|  "60 days is way too long. You could reverse a payment 2 months   |
|   after we delivered. 14 days with documented reasons is fair      |
|   for both sides."                                                 |
|                                                                    |
|  [Accept]  [Reject]  [Counter-propose]                            |
+------------------------------------------------------------------+
```

### Field mapping from ReviewData

| Display field | Source |
|---|---|
| Section reference | `issue.clause_reference` |
| Title | `issue.title` |
| Severity badge | `issue.risk_level` via `SeverityBadge` component |
| Current text | `issue.original_text` |
| TLP's proposed change | `issue.suggested_revision` (or `broker_decision.edited_text` if broker edited) |
| Why | `issue.plain_summary` or `issue.problem_explanation` |

### Section anchors

Each issue renders with an `id` attribute (`issue-{id}`) so the sidebar can link-jump. The sidebar shows issue titles grouped by severity (reuse `IssueSidebar` pattern).

### Mobile responsive

- On screens <768px: full-width cards, no sidebar, severity badges inline
- Section anchors become a collapsible "Jump to..." dropdown at the top
- Decision buttons stack vertically on <400px

---

## D. PER-ISSUE DECISION UI

### Three actions per issue

```
+------------------------------------------------------+
|  [Accept]      [Reject]      [Counter-propose]       |
+------------------------------------------------------+
```

| Action | Behavior | Visual state |
|---|---|---|
| Accept | Counterparty agrees to TLP's proposed change | Green highlight, checkmark |
| Reject | Counterparty rejects TLP's change, keeps original | Red highlight, X mark |
| Counter-propose | Opens text area for alternative wording | Blue highlight, edit icon |

### Counter-propose expansion

```
+------------------------------------------------------+
| Counter-propose:                                      |
|                                                        |
| +--------------------------------------------------+ |
| | "Company may reverse payment within 21 calendar  | |
| | days of the initial payment date, for documented  | |
| | quality failures only."                           | |
| +--------------------------------------------------+ |
|                                                        |
| Optional note to TLP:                                 |
| +--------------------------------------------------+ |
| | "21 days is our standard. Meet in the middle?"    | |
| +--------------------------------------------------+ |
|                                                        |
| [Save counter-proposal]   [Cancel]                    |
+------------------------------------------------------+
```

### State persistence

- All decisions stored in `localStorage` under key `milo-redline-{token}`
- Shape: `{ decisions: Record<issue_id, { status: 'accepted'|'rejected'|'counter', edited_text?: string, notes?: string }>, lastSaved: ISO string }`
- On page load: check localStorage for existing state, restore if found
- Visual indicator: "You have unsaved decisions from {date}. [Resume] [Start over]"
- Decisions are NOT sent to server until final submit

### Progress tracking

- Sticky bar at bottom shows: "{N} of {total} decided" with a progress bar
- Issues without a decision have a subtle pulse animation to draw attention

---

## E. SUBMIT FLOW

### Pre-submit state

The "Send decisions to TLP" button lives in the sticky bottom bar. It is:
- **Disabled** until at least 1 decision is made
- **Enabled but shows warning** if some issues are undecided: "You haven't decided on {N} items. They'll be marked as 'no response.'"

### Confirmation modal

```
+----------------------------------------------------------+
|                                                            |
|  Ready to send this back to TLP?                          |
|                                                            |
|  4 accepted, 2 rejected, 1 counter-proposal               |
|  3 items with no response (counted as no objection)        |
|                                                            |
|  TLP will review and get back to you.                     |
|                                                            |
|  [Send it]                          [Keep reviewing]       |
|                                                            |
+----------------------------------------------------------+
```

### On submit

1. POST to `/api/negotiation/submit` with:
   ```
   {
     token: string,
     decisions: NegotiationDecision[],
     actorName: string (from optional "Your name" field),
     overallNotes: string (from optional "Anything else for TLP?" field),
   }
   ```
2. API route calls `client.submitCounterpartyRound(token, submission)` which:
   - Validates token + expiration
   - Creates a `negotiation_rounds` row with `actor_type='counterparty'`
   - Transitions negotiation status `counterparty_review` → `broker_review`
   - Fires webhook to TLP (`negotiation.round_submitted`)
3. On success → show success state
4. On failure → show error with retry button

### Success state

```
+----------------------------------------------------------+
|                                                            |
|  "Done. TLP got your decisions."                          |
|                                                            |
|  "They'll review what you sent back and either accept,    |
|   counter, or call you. Usually takes a day or two."      |
|                                                            |
|  Your decisions:                                           |
|  4 accepted, 2 rejected, 1 counter-proposal               |
|                                                            |
|  [Download your response as PDF]   [Close this tab]       |
|                                                            |
+----------------------------------------------------------+
```

### Idempotency

- If the counterparty submits, then hits browser back and submits again:
  - `submitCounterpartyRound` calls `assertTransition('broker_review', 'counterparty_submit')` which fails because status is already `broker_review` (the first submit transitioned it)
  - API returns 409 Conflict with body: `{ error: "You already submitted your decisions." }`
  - Page shows: "You already sent this. TLP has it."

---

## F. ACTIONS PANEL

Rendered as a collapsible section below the issue list (mobile) or as a right sidebar panel (desktop >1200px).

### Actions

| Action | Availability | Implementation |
|---|---|---|
| **Download as Word** | Always | Gap: no DOCX generator exists. Spec: endpoint that takes `analysis.issues` + `brokerDecisions` → DOCX via `docx` npm package. **OPEN QUESTION 3:** Build in Part 2 or defer to Part 3? |
| **Sign as-is** | Only if counterparty has made zero decisions | Wires to `@milo/contract-signing`. Creates a `signing_documents` row, redirects to signing page. **OPEN QUESTION 4:** Does the signing page exist yet? D101 says contract-signing is "completely unwired." If not, defer. |
| **Request a call** | Always | `mailto:` link to operator email. Subject: "Call request — {counterparty_name} — {document_type}". Milo copy: "Sometimes it's easier to just talk. Hit this and TLP will set something up." |
| **Forward to my lawyer** | Always | Copies the review URL to clipboard + shows a pre-written email template: "Subject: Contract review — {document_type}. Body: TLP sent me this redline. Can you review? {url}" |

### Actions wireframe

```
+----------------------------------------------+
|  ACTIONS                                      |
|                                                |
|  [Download as Word]  (gap — defer to Part 3)  |
|  [Sign as-is]        (only if 0 decisions)    |
|  [Request a call]    (mailto)                  |
|  [Forward to lawyer] (copy link + template)   |
+----------------------------------------------+
```

---

## G. MILO PRESENCE

### Recommended: defer chat widget, ship textbox

A live `/ask` chat widget scoped to this contract would be powerful but complex (requires: WebSocket/SSE, contract context injection, rate limiting on unauthenticated page, abuse potential). **Not worth the complexity for Part 2.**

Ship instead: a "Question for TLP?" textarea at the bottom of the page.

```
+------------------------------------------------------+
|  Question for TLP?                                    |
|                                                        |
|  +--------------------------------------------------+ |
|  | "The indemnification clause -- does this cover    | |
|  | sub-affiliates too?"                              | |
|  +--------------------------------------------------+ |
|                                                        |
|  [Send to TLP]                                        |
|                                                        |
|  "I'll pass this along. They usually respond within  |
|   a business day."                                    |
+------------------------------------------------------+
```

On submit: appends the question to `negotiation_records.metadata.counterparty_questions[]` and fires the webhook. TLP sees it in the operator dashboard.

**OPEN QUESTION 5:** Should the question field be available before or after submitting decisions? Recommend: available at any time, independent of decision submission.

---

## H. EDGE CASES

### H1. v2 → v3 chain

When the counterparty submits on v2, TLP reviews and creates v3 (new round via `sendRound()`). The v2 link behavior:

- v2 token stays valid but page shows read-only state
- Banner: "TLP reviewed your response and sent you a new version."
- Link to v3: "Here's the latest: {v3 review URL}"
- v3 has its own token, own expiration, own decision state

Implementation: `loadReview()` already returns the full `rounds` array. If the latest round is `actor_type='broker'` and `round_number > link.roundNumber`, the page knows a newer version exists. Check `negotiation_links` for a link with `round_number = latest_round` to get the v3 URL.

### H2. Expired link

See Section A. 410 page with extension request button.

### H3. Double submit

See Section E (idempotency). State machine prevents it: `assertTransition('broker_review', 'counterparty_submit')` fails.

### H4. localStorage restoration

- On load: check `localStorage['milo-redline-{token}']`
- If found and not yet submitted: show banner "You were in the middle of reviewing this. Pick up where you left off?"
- [Resume] restores decisions, [Start over] clears localStorage
- If found but already submitted (server says status != `counterparty_review`): clear localStorage, show read-only

### H5. Mid-decision page close

- `beforeunload` event: "You have unsaved decisions. Leave anyway?"
- Only triggers if decisions exist in state but haven't been submitted

### H6. Very long contracts (30+ issues)

- Lazy render: only render issues in viewport + 5 above/below
- Sidebar navigation becomes essential
- "Jump to next undecided" button in sticky bar

### H7. Zero issues shown

If TLP accepted all issues (nothing to show counterparty), the page displays:
```
"TLP reviewed your contract and accepted everything as-is. No changes needed."

"If you want to proceed to signing: [Sign now]"
```

---

## I. INSTRUMENTATION

### Events tracked

| Event | Storage | Fields |
|---|---|---|
| Link opened | `negotiation_links.view_count`, `first_viewed_at`, `last_viewed_at` | Already in `loadReview()` |
| Section expanded | `negotiation_analytics` (new table or append to `metadata`) | `{ event: 'section_expanded', issue_id, timestamp }` |
| Time on page | `negotiation_analytics` | `{ event: 'page_time', seconds, timestamp }` (sent on `beforeunload`) |
| Decision made (pre-submit) | `negotiation_analytics` | `{ event: 'decision_made', issue_id, status, timestamp }` |
| Decisions submitted | `negotiation_rounds` (already tracked) | Full submission |
| Extension requested | `negotiation_analytics` | `{ event: 'extension_requested', timestamp }` |

### Operator-facing analytics (shown on `/contract-review/[id]` detail page)

```
Counterparty Activity:
- Opened link 3 times (first: Apr 21, last: Apr 23)
- Total time viewing: 12 minutes
- Viewed "Indemnification" section 3 times
- Made 4 decisions before submitting (2 changed after initial selection)
- Submitted on Apr 23 at 2:14 PM
```

**OPEN QUESTION 6:** Store analytics in `negotiation_records.metadata` (simple, no new table) or a dedicated `negotiation_analytics` table (cleaner, queryable)? Recommend: `metadata` for Part 2, migrate to dedicated table if analytics features grow.

---

## J. COPY (Milo Voice)

All copy follows VISION-MILO-V1-FUNNEL.md Section 11. First person, slightly cocky, useful, confident. Never "we," never corporate.

### Key copy strings

| Context | Copy |
|---|---|
| Page intro | "I marked up your contract. Here's what TLP wants to change." |
| Section heading | "Here's what needs your attention:" |
| Per-issue "why" label | "Why this matters:" |
| Counter-propose prompt | "Got a better idea? Write it here." |
| Progress bar | "{N} of {total} reviewed. Keep going." |
| All decided | "You've reviewed everything. Send it back when you're ready." |
| Submit button | "Send decisions to TLP" |
| Confirm modal title | "Ready to send this back to TLP?" |
| Confirm modal body | "They'll review what you sent and respond. Usually takes a day or two." |
| Confirm button | "Send it" |
| Cancel button | "Keep reviewing" |
| Success title | "Done. TLP got your decisions." |
| Success body | "They'll review what you sent back and either accept, counter, or call you." |
| Expired title | "This link expired." |
| Expired body | "It happens -- contracts move fast. Need more time?" |
| Extension button | "Ask TLP for more time" |
| Already submitted | "You already sent your decisions on {date}." |
| v3 available | "TLP reviewed your response and sent a new version." |
| Question prompt | "Question for TLP?" |
| Question sent | "I'll pass this along. They usually respond within a business day." |
| Undecided warning | "You haven't decided on {N} items. They'll be marked as 'no response.'" |
| Double submit | "You already sent this. TLP has it." |
| Request a call | "Sometimes it's easier to just talk." |
| Forward to lawyer | "Want your lawyer to look at this? Here's a link you can forward." |
| Error state | "Something went wrong on my end. Try again -- if it keeps happening, email TLP directly." |
| Zero issues | "TLP accepted everything as-is. No changes needed from you." |

### Anti-patterns (NEVER use)

- "We at TLP..."
- "Our system detected..."
- "Click submit when ready"
- "Thank you for your review"
- "Please review the following items"
- "Your response has been recorded"

---

## K. STATE PERSISTENCE

### Client-side (localStorage)

**Key:** `milo-redline-{token}` (token from URL)

**Shape:**
```typescript
interface RedlineLocalState {
  decisions: Record<number, {
    status: 'accepted' | 'rejected' | 'counter';
    edited_text?: string;
    notes?: string;
  }>;
  overallNotes: string;
  actorName: string;
  lastSaved: string; // ISO timestamp
  version: 1;        // schema version for future migrations
}
```

**Lifecycle:**
1. Page load → check localStorage for key
2. If found → validate `version` field, show restore prompt
3. User makes decision → immediate localStorage write (debounced 500ms)
4. User submits → clear localStorage after server confirms
5. Page shows read-only (already submitted / expired) → clear localStorage

### Server-side (Supabase)

**Tables touched:**

| Table | Read/Write | Purpose |
|---|---|---|
| `negotiation_links` | Read + Update | Token lookup, view tracking, expiration check |
| `negotiation_records` | Read + Update | Negotiation state, status transitions |
| `negotiation_rounds` | Read + Insert | Round history, counterparty submission |
| `analysis_records` | Read | Issue data for display |

**Counterparty submission shape (mirrors `CounterpartySubmission` from `@milo/contract-negotiation`):**
```typescript
{
  actorName: string;          // "Randy Johnson" — from optional name field
  decisions: Array<{
    issue_id: number;         // matches analysis_records.issues[].id
    status: 'accepted' | 'rejected' | 'counter';
    edited_text?: string;     // only for counter-proposals
    notes?: string;           // optional per-issue note
  }>;
  overallNotes?: string;      // "Overall, we're close. Let's talk about Section 5."
  actorIp?: string;           // captured server-side from request headers
  actorUserAgent?: string;    // captured server-side
}
```

---

## Component Reuse Map

| Existing component | Reuse opportunity | Modifications needed |
|---|---|---|
| `SeverityBadge` (`src/components/contract/SeverityBadge.tsx`) | Severity indicator on each issue card | None — works as-is with `RiskLevel` from `@milo/contract-analysis` |
| `IssueSidebar` (`src/components/contract/IssueSidebar.tsx`) | Section navigation sidebar | Minor: add click handler for anchor scrolling, add "undecided" indicator per issue |
| `IssueCard` (`src/components/contract/IssueCard.tsx`) | Issue display card | Significant: replace FeedbackThumbs with Accept/Reject/Counter buttons, add counter-propose textarea, add decision state display |
| `ContractAnalysisDisplay` (`src/components/shared/ContractAnalysisDisplay.tsx`) | Decision button pattern | Extract `DecisionButton` sub-component. Note: this component uses lowercase `'high'|'medium'|'low'` risk levels (missing `'critical'`). Must use `@milo/contract-analysis` UPPERCASE types instead. |
| `verticalConfig.analysis.severityColors` | Color tokens for severity badges | None — already defined |
| `@milo/contract-negotiation` `loadReview()` | Token resolution + data loading | None — use directly |
| `@milo/contract-negotiation` `submitCounterpartyRound()` | Decision submission | None — use directly |
| `@milo/contract-analysis` `ContractIssue` type | Issue data shape | None — canonical type |
| `/c/[slug]/route.ts` | Short URL resolution pattern | Adapt for `/r/[code]` route — same pattern, different table (`negotiation_links` instead of `contract_documents`) |

### New components needed

| Component | Purpose |
|---|---|
| `RedlineIssueCard` | Per-issue display with decision buttons (Accept/Reject/Counter) + counter-propose textarea |
| `RedlineProgressBar` | Sticky bottom bar with decision count + submit button |
| `RedlineConfirmModal` | Submit confirmation with decision summary |
| `RedlineSuccessState` | Post-submit success display |
| `RedlineExpiredState` | 410 expired link display |
| `RedlineReadOnlyBanner` | Banner for already-submitted / cancelled / finalized states |

---

## Page Component Tree

```
/review/[token]/page.tsx (server component — token resolution)
  └─ RedlineReviewPage (client component — all interactive state)
      ├─ RedlineHeader
      │   ├─ counterparty name, doc type, version
      │   ├─ date sent, expiration
      │   └─ Milo intro copy
      ├─ RedlineReadOnlyBanner (conditional: expired/submitted/cancelled)
      ├─ RedlineSidebar (desktop only, >1200px)
      │   └─ IssueSidebar (reused, with decision indicators)
      ├─ RedlineIssueList
      │   └─ RedlineIssueCard (per issue)
      │       ├─ SeverityBadge (reused)
      │       ├─ original text display
      │       ├─ proposed change display
      │       ├─ "why" explanation
      │       └─ decision buttons (Accept / Reject / Counter)
      │           └─ CounterProposeTextarea (conditional)
      ├─ RedlineQuestionBox ("Question for TLP?")
      ├─ RedlineActionsPanel
      │   ├─ Download as Word (gap — shows disabled with tooltip)
      │   ├─ Sign as-is (conditional)
      │   ├─ Request a call (mailto)
      │   └─ Forward to lawyer (copy link)
      ├─ RedlineProgressBar (sticky bottom)
      │   ├─ progress count + bar
      │   └─ [Send decisions to TLP] button
      └─ RedlineConfirmModal (conditional)
          └─ decision summary + confirm/cancel
```

---

## Route Structure

| Route | Type | Purpose |
|---|---|---|
| `/review/[token]/page.tsx` | Next.js page (server component shell + client component body) | Full redline review page |
| `/r/[code]/route.ts` | Next.js route handler (GET) | Short code → redirect to `/review/{token}` |
| `/api/negotiation/submit/route.ts` | Next.js API route (POST) | Counterparty decision submission |
| `/api/negotiation/question/route.ts` | Next.js API route (POST) | Counterparty question submission |
| `/api/negotiation/analytics/route.ts` | Next.js API route (POST) | Beacon endpoint for instrumentation events |

---

## Open Questions (Coder-1 needs Mark to decide before building)

1. **Extension model:** Should "Extend 7 days" be self-service (counterparty clicks, link auto-extends) or request-based (sends email to TLP, TLP manually extends)? Recommend request-based for Part 2.

2. **Header branding:** Show TLP logo (requires logo upload system) or text-only "TLP Compliance" with Milo penguin favicon? Recommend text-only for Part 2.

3. **DOCX download:** Build the Word export in Part 2 or defer to Part 3? No DOCX generator exists in either primitive. The `docx` npm package could generate from structured issue data. Recommend defer — the PDF from `@milo/contract-signing` certificate generator could be adapted faster.

4. **"Sign as-is" availability:** Does the signing page exist yet? D101 audit says @milo/contract-signing is "completely unwired" in milo-for-ppc. If no signing page, this button must be hidden or show "Coming soon." Recommend: hide entirely for Part 2.

5. **Question field timing:** Should counterparties be able to ask questions before submitting decisions, after, or both? Recommend both — the field is always visible, independent of decision state.

6. **Analytics storage:** Append counterparty analytics to `negotiation_records.metadata` (simple, no migration) or create a `negotiation_analytics` table (cleaner, queryable)? Recommend `metadata` for Part 2.

7. **Counterparty name field:** The submit flow includes `actorName`. Should this be required (counterparty must type their name) or optional? Recommend required — TLP needs to know who reviewed.

8. **Mobile breakpoint strategy:** Use the existing dark-mode design system from milo-for-ppc, or a lighter counterparty-facing theme? The counterparty has never seen the operator dashboard — they don't expect dark mode. Recommend: light theme for this page only, using neutral colors. The operator dashboard stays dark.

9. **Undecided issue handling:** If counterparty submits with undecided issues, should those be recorded as `'pending'` (explicitly undecided) or omitted from the submission? Recommend `'pending'` — TLP should see that the counterparty deliberately skipped items vs. accepted them.

---

Report at: https://raw.githubusercontent.com/moscaleflow/conductor-mark-public/main/reports/2026-04-24-coder-3-counterparty-redline-page-spec.md
