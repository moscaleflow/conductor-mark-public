# MOP Analyzer Reasoning-Stripped Format Spec for /synth Snapshot 2

**Coder-3 Research | 2026-04-24**
**Scope:** Pure spec. No code. Defines the stripped output format Mark pastes into /synth for Snapshot 2 collection (Gap 4).
**Cross-references:** D102a/D117 (Snapshot 2 plan, Gap 4 = TOP-1 priority), D106 (voice consistency audit), Vet Part 1.5 IssueCard format.

---

## A. Why Stripping Matters

1. **Operator review time.** The full MOP analyzer output for a single issue includes 5-7 fields spanning ~200 words. Tiffani doesn't need `legal_reasoning` or `negotiation_leverage` to draft a redline — she needs the clause, the problem, and the fix.

2. **Persona library signal density.** The current Contracts persona query #9 ("strip the reasoning, quick copy-paste for tita") references a stripped format that was never captured. The persona library v1 inferred the format. Confirming it with a real sample closes the Contracts thinness flag.

3. **Alignment with Vet Part 1.5.** The IssueCard component already renders a stripped view — `plain_summary` (1-2 sentences), `suggested_revision`, `clause_reference`. The card hides `problem_explanation` behind an expand click and `legal_reasoning` behind "Explain Further." The stripped format for /synth should match what operators see in the collapsed card state, not the full expanded state.

4. **Voice consistency (D106).** D106 HIGH-02 flagged anti-hedging gaps. The reasoning chain in `problem_explanation` often hedges ("could," "may," "if they claim") while `plain_summary` is direct ("They can delay your rejections forever and you're stuck paying"). Stripping to `plain_summary` voice reinforces the anti-hedging direction.

---

## B. Current State — What's in MOP Analyzer Output Today

### Two pipelines produce analysis output

| Pipeline | Prompt | Model | Storage | Output shape |
|---|---|---|---|---|
| **Legacy** (`/api/contract/analyze-bg`) | `contract-analysis-prompt.ts` (495 lines, 60+ rules inline) | Opus (`tier: 'judgment'`) | `contract_documents.analysis_result` | `FlaggedClause[]` with `plain_english`, `impact_to_us`, `recommendation` |
| **@milo/contract-analysis engine** | `system-prompt.ts` (230 lines) + base registry (45 rules) | Sonnet (`tier: 'structured'`) | `contract_analyses` table | `ContractIssue[]` with `plain_summary`, `problem_explanation`, `suggested_revision` |

Both pipelines produce per-issue output with this field breakdown:

### Per-issue fields (engine pipeline — current)

| Field | Purpose | Avg length | Strippable? |
|---|---|---|---|
| `rule_id` | Registry rule match (e.g., `BUY_FIN_NO_REJECTION_WINDOW`) | 20-40 chars | KEEP (reference) |
| `risk_level` | CRITICAL / HIGH / MEDIUM / LOW | 4-8 chars | KEEP (severity) |
| `category` | PAYMENT / LIABILITY / TERMINATION / IP / COMPLIANCE / SCOPE / OTHER | 5-12 chars | KEEP (grouping) |
| `clause_reference` | Section number (e.g., "Section 5.4") | 10-20 chars | KEEP (location) |
| `title` | Short issue name (e.g., "No Defined Rejection Window") | 20-50 chars | KEEP (label) |
| `original_text` | Verbatim quote from contract | 10-200 chars | KEEP (evidence) |
| `plain_summary` | 1-2 sentence operator-facing finding | 30-80 words | KEEP (the finding) |
| `suggested_revision` | Ready-to-use replacement clause | 30-150 words | KEEP (the fix) |
| `problem_explanation` | Multi-paragraph reasoning chain with "Additionally:" blocks | 80-250 words | **STRIP** |
| `legal_reasoning` | Brief legal basis | 20-60 words | **STRIP** |
| `negotiation_leverage` | Tactical advice for counterparty discussion | 30-80 words | **STRIP** |
| `score_impact` | Numeric scoring weight | 2-3 chars | **STRIP** |
| `merged_rule_ids` | Array of rules merged into this issue | array | **STRIP** |
| `rider_refs` | Rider section references | array | KEEP (location) |

### Where reasoning lives

The reasoning chain is NOT in a dedicated field. It is spread across three fields:

1. **`problem_explanation`** — the main reasoning chain. Contains the "why this is bad" logic with multi-paragraph "Additionally:" blocks that chain related observations. Example: 3 paragraphs for a single rejection-window issue covering (1) no deadline, (2) no remedy, (3) no process.

2. **`legal_reasoning`** — brief legal basis. Usually one sentence citing a legal principle or standard.

3. **`negotiation_leverage`** — tactical advice. Contains what to say to the counterparty, with suggested phrasing.

These three fields account for ~60% of the per-issue word count. Stripping them cuts each issue from ~400 words to ~150 words.

---

## C. Proposed Stripped Format

### What stays (the "operator card" — matches IssueCard collapsed state)

```
[RISK_LEVEL] [CATEGORY] — [TITLE]
Section: [clause_reference] | Rider: [rider_refs]

CURRENT TEXT:
[original_text]

FINDING:
[plain_summary]

REDLINE:
[suggested_revision]
```

### What goes

- `problem_explanation` — reasoning chain, not operator-facing
- `legal_reasoning` — legal basis, useful for lawyers but not for Tiffani's redline paste
- `negotiation_leverage` — tactical advice, belongs in the Sales/Buyer pill, not in the redline
- `score_impact` — internal scoring math
- `merged_rule_ids` — internal pipeline metadata
- `rule_id` — kept only as a small parenthetical reference, not a separate section

### Format rules

1. One issue per block, separated by `---`
2. Risk level as bracket tag: `[CRITICAL]`, `[HIGH]`, `[MEDIUM]`, `[LOW]`
3. Category in ALL CAPS after risk level
4. `CURRENT TEXT` uses verbatim quote — no paraphrasing
5. `FINDING` is the `plain_summary` field — direct, uses "you" and "they," no hedging
6. `REDLINE` is the `suggested_revision` field — ready-to-paste contract language
7. If `original_text` is "Not addressed" (missing clause), skip CURRENT TEXT section and label FINDING as "MISSING CLAUSE:"
8. Header line at top: `[DOCUMENT NAME] — [ROLE] Analysis — [OVERALL_RISK] Risk — [N] Issues`

### Alignment with IssueCard

| Stripped format field | IssueCard field | Config key |
|---|---|---|
| Risk level bracket | `SeverityBadge` | `severity` |
| Title | `issue.title` (bold header) | — |
| Clause reference | `issue.clause_reference` | `riderRef` |
| Finding | `issue.plain_summary` (collapsed preview) | `plainEnglish` |
| Redline | `issue.suggested_revision` | `recommendation` |
| Current text | `issue.original_text` | — (shown in expanded state) |

The stripped format is the IssueCard's collapsed state plus `original_text` and `suggested_revision` — exactly what an operator needs to copy-paste for redlining.

---

## D. Extraction Prompt for /synth

### Prompt Mark types into /synth chat

```
Here's a real MOP contract analysis output, stripped to the operator-facing format —
the version I'd copy-paste to Tiffani for redlining. No reasoning chains, no scoring math,
just findings and fixes:

[paste stripped output here]

This is from a [buyer/publisher] MSA analysis. Extract patterns for the Vet and Contracts
personas:
1. What voice does the FINDING section use? (directness, pronouns, hedging level)
2. What format does the REDLINE section follow? (party references, defined terms, clause structure)
3. Are there recurring category patterns that should become persona queries?
4. Does this match the IssueCard format in Vet Part 1.5?
```

### What Mark pastes

Mark takes a real MOP analysis (from the contract-review page or /proof), strips it to the format in Section C, and pastes. The stripping can be done:

- **Manually:** Copy issue cards from the contract-review page, remove the expanded reasoning sections
- **Via /ask:** "Strip this analysis to redline format" — the Vet pill's CONTRACT VET MODE already knows not to repeat per-issue breakdowns, but a new command could produce the stripped format
- **Via export button (future):** A "Copy for redline" button on the contract-review page that outputs the stripped format to clipboard

For Snapshot 2, Mark does it manually. The export button is a post-v1 feature.

---

## E. Expected Yield

### Persona library v2 size impact

- **Current:** Contracts persona has 30 queries in v1, flagged THIN (format inferred, no sourced stripped samples)
- **With stripped MOP sample:** +3-5 new queries grounded in real operator format
  - Query confirming stripped format structure (Section C template)
  - Query testing FINDING voice (direct, no hedging, "you" and "they")
  - Query testing REDLINE structure (legal clause format, party placeholders)
  - Query testing MISSING CLAUSE handling (no original text → different format)
  - Query testing multi-issue summary header
- **New query total:** 33-35 Contracts persona queries (from 30)

### Voice consistency benefit (D106 cross-reference)

| D106 Finding | Stripped Format Impact |
|---|---|
| HIGH-02: Anti-hedging gap | `plain_summary` field is already anti-hedged ("They can delay your rejections forever" vs problem_explanation's "could sit on your rejection requests"). Stripping reinforces the target voice. |
| HIGH-03: Out-of-scope deflection | Stripped format has no room for off-topic content — it's structured. Confirms the direction. |
| HIGH-04: Posture anchors | `plain_summary` voice IS the Vet posture — skeptical, direct, operator-first. Sourcing real samples grounds the posture anchor in evidence. |

### Vet Part 1.5 alignment

The stripped format exactly matches the IssueCard collapsed view that Vet Part 1.5 renders. Confirming this format means:
- Persona queries test the SAME format operators see in the UI
- Voice consistency checks run against the SAME output shape
- No format drift between what the model produces and what the persona library tests

### Gaps closed

- **Contracts persona thinness flag** — CLOSED. Format moves from "inferred" to "sourced."
- **D98 query #9 format mismatch risk** — RESOLVED. Real sample confirms or corrects the assumed format.

---

## F. Sample BEFORE/AFTER

### Source: Real buyer MSA analysis (from proof page data)

---

### BEFORE: Full MOP Output (Issue 1 of 11)

```
TITLE: No Defined Rejection Window
RULE_ID: BUY_FIN_NO_REJECTION_WINDOW
CATEGORY: PAYMENT
RISK_LEVEL: CRITICAL
SCORE_IMPACT: 40
CLAUSE_REFERENCE: Section 5.4
RIDER_REFS: Buyer Rider §1, Buyer Rider §2
MERGED_RULE_IDS: BUY_FIN_NO_REJECTION_WINDOW, BUY_FIN_NO_REPLACEMENT_OR_CREDIT,
                 BUY_FIN_REJECTION_PROCESS_NOT_DEFINED

ORIGINAL TEXT:
"Invalid, fraudulent, or non qualified Leads/Calls may be rejected."

PLAIN SUMMARY:
No deadline for rejecting bad leads. They can delay your rejections forever and
you're stuck paying. You need a 7-day rejection window minimum or you lose all
leverage.

PROBLEM EXPLANATION:
There's no time limit on when you can reject bad leads or calls. They could sit
on your rejection requests for months while you're stuck paying for garbage
traffic. You need a hard deadline or they control when (or if) you ever get credit.

Additionally: The contract says you can reject bad leads but doesn't say what
happens next. Do you get a credit? Replacement leads? A refund? Without a remedy,
rejection is meaningless — you still paid for garbage.

Additionally: How do you actually reject a lead? Email? Portal? What evidence do
you need? How long do they have to respond? None of this is defined, so every
rejection becomes a negotiation.

LEGAL REASONING:
UCC §2-602 requires rejection within a reasonable time. Without a defined window,
the buyer loses the right to reject and must accept nonconforming goods.

NEGOTIATION LEVERAGE:
Industry standard is 7-30 days for lead/call rejection. Ask: 'How long do you
need to validate a lead? We need the same window to validate what we're buying.
Let's make it mutual and reasonable.'

SUGGESTED REVISION:
Company shall submit rejection notices via email to [designated email] within
seven (7) calendar days of delivery, including: (a) Lead/Call ID, (b) date/time
of delivery, (c) specific rejection reason per IO quality standards, and (d)
supporting evidence (e.g., call recording, duplicate check, consent verification).
Buyer Network #1 shall respond to rejection notices within three (3) business days
with acceptance or dispute. Disputed rejections shall be resolved through good
faith negotiation within ten (10) business days.
```

**Word count: ~310 words**

---

### AFTER: Stripped Format (same issue)

```
[CRITICAL] PAYMENT — No Defined Rejection Window
Section: 5.4 | Rider: Buyer Rider §1, Buyer Rider §2

CURRENT TEXT:
"Invalid, fraudulent, or non qualified Leads/Calls may be rejected."

FINDING:
No deadline for rejecting bad leads. They can delay your rejections forever and
you're stuck paying. You need a 7-day rejection window minimum or you lose all
leverage.

REDLINE:
Company shall submit rejection notices via email to [designated email] within
seven (7) calendar days of delivery, including: (a) Lead/Call ID, (b) date/time
of delivery, (c) specific rejection reason per IO quality standards, and (d)
supporting evidence (e.g., call recording, duplicate check, consent verification).
Buyer Network #1 shall respond to rejection notices within three (3) business days
with acceptance or dispute. Disputed rejections shall be resolved through good
faith negotiation within ten (10) business days.
```

**Word count: ~130 words (58% reduction)**

---

### BEFORE: Full MOP Output (Issue 2 of 11 — missing clause)

```
TITLE: Data Ownership Not Defined
RULE_ID: BUY_DATA_OWNERSHIP_UNCLEAR
CATEGORY: IP
RISK_LEVEL: HIGH
SCORE_IMPACT: 38
CLAUSE_REFERENCE: Not addressed
RIDER_REFS: Buyer Rider §4

ORIGINAL TEXT:
Not addressed

PLAIN SUMMARY:
Contract doesn't say who owns lead data and recordings after purchase. If they
own it, you can't defend lawsuits or verify compliance. You need ownership transfer.

PROBLEM EXPLANATION:
Who owns the lead data, call recordings, and consent logs after you buy them? If
they claim ownership, you can't use the data to defend TCPA lawsuits or verify
compliance. You need clear ownership transfer upon payment.

NEGOTIATION LEVERAGE:
Ask: 'Once we pay for a lead, who owns it? We need to own the data to defend
ourselves legally. Ownership should transfer on payment — that's standard.'

SUGGESTED REVISION:
Upon full payment for Leads/Calls, all right, title, and interest in the delivered
Leads/Calls, including associated consumer data, call recordings, consent records,
and metadata, shall transfer to Company. Buyer Network #1 retains no ownership
rights in delivered and paid-for Leads/Calls. Company may use, store, and process
such data for any lawful business purpose including compliance verification, TCPA
defense, and customer relationship management.
```

**Word count: ~190 words**

---

### AFTER: Stripped Format (missing clause variant)

```
[HIGH] IP — Data Ownership Not Defined
Rider: Buyer Rider §4

MISSING CLAUSE:
Contract doesn't say who owns lead data and recordings after purchase. If they
own it, you can't defend lawsuits or verify compliance. You need ownership transfer.

REDLINE:
Upon full payment for Leads/Calls, all right, title, and interest in the delivered
Leads/Calls, including associated consumer data, call recordings, consent records,
and metadata, shall transfer to Company. Buyer Network #1 retains no ownership
rights in delivered and paid-for Leads/Calls. Company may use, store, and process
such data for any lawful business purpose including compliance verification, TCPA
defense, and customer relationship management.
```

**Word count: ~100 words (47% reduction)**

---

### Full stripped output (header + 2 sample issues)

This is what Mark pastes into /synth:

```
Buyer MSA Analysis — CRITICAL Risk — 11 Issues

---

[CRITICAL] PAYMENT — No Defined Rejection Window
Section: 5.4 | Rider: Buyer Rider §1, Buyer Rider §2

CURRENT TEXT:
"Invalid, fraudulent, or non qualified Leads/Calls may be rejected."

FINDING:
No deadline for rejecting bad leads. They can delay your rejections forever and
you're stuck paying. You need a 7-day rejection window minimum or you lose all
leverage.

REDLINE:
Company shall submit rejection notices via email to [designated email] within
seven (7) calendar days of delivery, including: (a) Lead/Call ID, (b) date/time
of delivery, (c) specific rejection reason per IO quality standards, and (d)
supporting evidence (e.g., call recording, duplicate check, consent verification).
Buyer Network #1 shall respond to rejection notices within three (3) business days
with acceptance or dispute. Disputed rejections shall be resolved through good
faith negotiation within ten (10) business days.

---

[HIGH] IP — Data Ownership Not Defined
Rider: Buyer Rider §4

MISSING CLAUSE:
Contract doesn't say who owns lead data and recordings after purchase. If they
own it, you can't defend lawsuits or verify compliance. You need ownership transfer.

REDLINE:
Upon full payment for Leads/Calls, all right, title, and interest in the delivered
Leads/Calls, including associated consumer data, call recordings, consent records,
and metadata, shall transfer to Company. Buyer Network #1 retains no ownership
rights in delivered and paid-for Leads/Calls. Company may use, store, and process
such data for any lawful business purpose including compliance verification, TCPA
defense, and customer relationship management.

---

[... 9 more issues ...]
```

---

Cross-references: Decision 117/D102a (Snapshot 2 plan, Gap 4), Decision 106 (voice consistency audit — HIGH-02 anti-hedging, HIGH-04 posture anchors), Vet Part 1.5 IssueCard format (`perIssueFields` config).
