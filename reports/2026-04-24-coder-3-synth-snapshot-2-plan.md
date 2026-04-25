# /synth Snapshot 2 Collection Plan

**Coder-3 Research | 2026-04-24**
**Inputs:** Persona libraries v1 (180 queries, commit 5f4970a), D98 pill prompt audit, D100 contracts page audit, D101 primitive plug-in gap audit, /synth Snapshot 1 handoff (6 dumps + Dump C1)

---

## Gap Inventory (sorted by priority: HIGH first)

---

### Gap 1: Mark's Loom — "Contract Review and Key Considerations for Buyers"

**Priority:** HIGH
**Effort for Mark:** MEDIUM (transcribe Loom or dictate heuristics from memory)

**Why HIGH:** The Contracts persona's decision framework (query #26: "Malvin isn't sure which redlines to accept and which to push back on") is currently inferred from pattern analysis of 13 contracts in Dump C1. Mark's Loom is the actual training material Malvin watches. Without it, the decision heuristic tree (ALWAYS redline / USUALLY redline / CASE-BY-CASE / ACCEPT) is reconstructed, not sourced. D98 pill prompt audit's proposed contracts pill would need this framework as its backbone.

**Closes:** Contracts persona thinness flag ("Reasoning-stripped format inferred. No live negotiation thread captured end-to-end."). Also closes the gap between the 495-line `contract-analysis-prompt.ts` and what Mark actually teaches operators — any discrepancy is a prompt drift risk.

**Cross-reference:** D100 audit found two parallel analysis pipelines. Mark's Loom heuristics would definitively answer which pipeline's rules are canonical — the 60+ rule prompt or the operator-facing training.

**Extraction prompt:**
```
Here's a transcript of my Loom "Contract Review and Key Considerations for Buyers": [paste transcript or dictate from memory].

I want to extract:
1. The decision heuristics you teach — which clauses to always redline, which to accept, which are case-by-case
2. Any buyer-specific review patterns that differ from publisher review
3. Any rules of thumb you give about when to escalate vs. handle independently
4. The priority ordering you recommend for a new reviewer scanning a contract for the first time

Tag each heuristic with which persona it feeds: Contracts, Buyer, or both.
```

**Expected yield:**
- 8-12 new Contracts persona queries (decision framework scenarios)
- 3-5 new Buyer persona queries (buyer-specific contract considerations)
- Definitive heuristic tree replacing the inferred version in persona library v1
- Closes 1 thinness flag (Contracts)

**Dependencies:** None. Can be collected first.

---

### Gap 2: Cam and Vee LinkedIn DMs — outbound + inbound

**Priority:** HIGH
**Effort for Mark:** LOW-MEDIUM (screenshot or copy-paste LinkedIn DMs)

**Why HIGH:** Sales persona is rated MEDIUM thinness specifically because "Zero email voice. Zero LinkedIn DM voice." All 30 Sales queries are based on Teams/chat register. LinkedIn DMs are a primary outreach channel for TLP — Cam and Vee use it daily. The voice register is meaningfully different (more professional, tighter, fewer in-jokes). D98 pill prompt audit noted Sales pill missing the internal vs. external voice distinction. LinkedIn DMs would provide the external register anchor.

**Closes:** Sales persona thinness flag ("LOW for email/LinkedIn"). Also closes D98 pill audit HIGH-02 (Sales): "Add internal vs. external voice distinction" — LinkedIn DMs ARE the external voice.

**Extraction prompt:**
```
Here are LinkedIn DM threads from Cam and Vee — both outbound pitches to new prospects and inbound responses to interested leads: [paste threads].

Extract:
1. The outbound pitch structure — how they open, what they lead with, how long the message is
2. The inbound response pattern — how they reply to someone who reached out first
3. Voice differences from Teams chat — formality level, emoji usage, sign-off patterns
4. Any follow-up cadence (do they send a second message? How soon? What does the follow-up say?)
5. Successful vs unsuccessful threads if both are visible — what worked?

Tag each pattern as: outbound_cold, outbound_warm, inbound_response, follow_up.
```

**Expected yield:**
- 8-10 new Sales persona queries (LinkedIn-specific drafts and responses)
- 2-3 new voice patterns (LinkedIn register vs. Teams register)
- 3-5 entity references (new prospects from LinkedIn)
- Sales persona jumps from MEDIUM to HIGH thinness
- Closes 1 thinness flag (Sales)

**Dependencies:** None. Can be collected first.

---

### Gap 3: Real TLP dispute arc (filing → investigation → resolution)

**Priority:** HIGH
**Effort for Mark:** MEDIUM (gather a complete dispute thread with timestamps and resolution)

**Why HIGH:** Buyer persona dispute handling (queries #4-6) is entirely framework-based — the pill prompt says "acknowledge → investigate → resolve or push back with data" but no real dispute thread was captured. The Lead Buyer Hub arc in Dump 6 shows underperformance handling but not a formal dispute (complaints, credits, clawback negotiations). D98 pill audit noted the buyer pill's dispute framework is abstract. A real dispute arc would ground it.

**Closes:** Buyer persona thinness flag ("No real dispute arc. No call recording analysis."). Also provides evidence for D100's missing contract status `rejected` — a dispute that escalates to contract-level action.

**Cross-reference:** D101 found @milo/blacklist has severity tracking that /ask doesn't use. A dispute arc would show what threshold triggers a severity escalation vs. a warning, grounding the blacklist integration gap.

**Extraction prompt:**
```
Here's a real dispute thread — from the first buyer complaint through investigation to resolution: [paste the full thread with timestamps].

Extract:
1. What triggered the dispute (quality, duration, duplicates, fraud, nonpayment)
2. What data Tiffani/Cam/Malvin pulled to investigate (call recordings? duration logs? TrackDrive data?)
3. How the verdict was reached — data-backed or judgment call?
4. What the resolution was (credit issued, publisher warned, publisher blacklisted, buyer mollified, escalated to Mark)
5. The tone at each stage — how does TLP voice change from "acknowledging" to "investigating" to "delivering the verdict"?
6. If the dispute involved a publisher: what happened to the publisher relationship afterward?

Tag each step: dispute_trigger, investigation, verdict, resolution, aftermath.
```

**Expected yield:**
- 5-8 new Buyer persona queries (real dispute scenarios replacing framework queries)
- 2-3 new Adversarial persona queries (if the dispute involved fraud detection)
- Real investigation methodology (what data sources, in what order)
- Closes 1 thinness flag (Buyer)

**Dependencies:** None. Can be collected independently.

---

### Gap 4: MOP analyzer output — reasoning-stripped format

**Priority:** HIGH
**Effort for Mark:** LOW (single paste of a real stripped output)

**Why HIGH:** Contracts persona query #9 ("strip the reasoning, quick copy-paste for tita") is based on Mark's confirmed preference but the actual stripped format was never captured. The D98 pill audit proposed a contracts pill that would need to produce both formats (with-reasoning and stripped). Without a real example of the stripped output, the format is invented, not sourced. Low effort + high impact = immediate win.

**Closes:** Contracts persona gap ("Reasoning-stripped format inferred"). Directly confirms whether the current MOP format in persona library #9 matches reality.

**Extraction prompt:**
```
Here's a real MOP analyzer output with the reasoning stripped — the version you'd copy-paste directly to Tiffani for redlining: [paste the stripped output].

I want to confirm:
1. The exact structure — is it literally just "Section / Current Text / Proposed Revision" with no other fields?
2. Are there any headers, numbering, or separator lines?
3. Does Tiffani modify these before sending to the counterparty, or are they sent verbatim?
4. Is there a specific number of items that's typical (5? 10? 15?) or does it vary?
```

**Expected yield:**
- 2-3 Contracts persona queries refined (format confirmation)
- Definitive format spec for the contracts pill's stripped output mode
- Closes 1 thinness flag (Contracts)

**Dependencies:** None. Fastest gap to close.

---

### Gap 5: Malvin publisher recruitment pitch end-to-end

**Priority:** MEDIUM
**Effort for Mark:** MEDIUM (gather a multi-message thread from Malvin's outreach)

**Why MEDIUM:** Publisher persona has Malvin's offer card voice and "copy tita noted" register captured, but not his full recruitment flow (initial outreach → follow-up → negotiation → close or reject). The persona library's recruitment draft queries (#9-12) are composites using the Hormozi framework from the prompt, not Malvin's actual outreach. This would close the gap between the framework and the operator's real-world application of it.

**Closes:** Publisher persona thinness flag ("Malvin end-to-end pitch missing"). Would also reveal whether Malvin actually follows the Hormozi framework or has his own adapted version — if the latter, D98's publisher pill needs updating.

**Extraction prompt:**
```
Here's a Malvin publisher recruitment thread from first contact to outcome: [paste the full thread].

Extract:
1. Opening message — how does he lead? Does he use the Hormozi framework (lead with what THEY get) or his own approach?
2. Follow-up pattern — how many follow-ups? What cadence? How does tone change between messages?
3. Negotiation phase — how does he handle pricing pushback? Terms pushback?
4. Close or rejection — what was the outcome? If close, what sealed it? If rejection, what was the reason?
5. Voice characteristics — formality level, Tagalog/English register mixing, emoji usage

Tag each message as: opening, follow_up_1, follow_up_2, negotiation, close/reject.
```

**Expected yield:**
- 5-7 new Publisher persona queries (real recruitment flow replacing composites)
- Malvin's actual recruitment voice vs. Hormozi framework (confirms or revises D98 publisher pill proposals)
- 1-2 entity references (the publisher being recruited)
- Closes 1 thinness flag (Publisher)

**Dependencies:** None.

---

### Gap 6: Email thread samples (Sales/Contracts voice in email register)

**Priority:** MEDIUM
**Effort for Mark:** MEDIUM (find and redact an email thread)

**Why MEDIUM:** All 180 queries in persona library v1 are based on Teams/chat voice. Email register is more formal, longer-form, and uses different conventions (subject lines, sign-offs, CC patterns). Sales and Contracts personas need email register samples to prevent Milo from drafting emails that read like chat messages. D98 pill audit noted this gap for Sales but it equally applies to Contracts (counterparty redline responses are often email-based).

**Closes:** Sales thinness ("Zero email voice"). Partially closes Contracts gap (counterparty communication often email-based).

**Extraction prompt:**
```
Here are 2-3 email threads — at least one sales outreach/follow-up and one contract-related exchange with a counterparty: [paste threads with headers].

Extract:
1. Subject line conventions — how does TLP subject their emails?
2. Opening patterns — "Hi [name]," vs "Hey [name]" vs no greeting
3. Body structure — paragraph length, bullet usage, formality level
4. Sign-off patterns — "Best," "Thanks," "- Mark," etc.
5. Voice shift from chat — where does the email voice differ from the Teams chat voice?
6. CC patterns — who gets CC'd and when?

Tag each email as: sales_outbound, sales_follow_up, contracts_redline_response, contracts_initial_send.
```

**Expected yield:**
- 4-6 new Sales persona queries (email-specific drafts)
- 2-3 new Contracts persona queries (email-formatted redline responses)
- Email register spec for pill prompts (D98 follow-up)
- Partially closes 2 thinness flags (Sales email, Contracts communication format)

**Dependencies:** None.

---

### Gap 7: Publisher onboarding flow (pub-side, first contact → live traffic)

**Priority:** MEDIUM
**Effort for Mark:** MEDIUM (document or screenshot the onboarding steps)

**Why MEDIUM:** Buyer onboarding is captured (Dump 6: Tiffani sells → Cam processes → MSA/W9 → setup card → compliance gate → go-live). Publisher onboarding is the mirror flow but was never documented. D101 found @milo/onboarding primitive exists with startRun/getStatus but only 3/4 exports are used — publisher onboarding steps would clarify what the remaining `completeStep` export should track.

**Closes:** Publisher persona thinness ("Publisher-side onboarding lifecycle coverage missing"). Also informs the @milo/onboarding primitive integration in D101.

**Extraction prompt:**
```
Walk me through the publisher onboarding flow from first contact to live traffic: [dictate or paste the steps].

Extract:
1. First contact — who reaches out? Publisher or TLP?
2. Qualification — what does TLP check before proceeding? (Vet? Blacklist? Traffic source verification?)
3. Paperwork — what docs does TLP send? What docs does the publisher send back? (MSA, W9, IO)
4. Technical setup — DID assignment, ping URL, TrackDrive/Ringba integration, test calls
5. Compliance gate — what must pass before go-live? (Creative review? Consent language review?)
6. Go-live — who flips the switch? What's the confirmation process?
7. Post-go-live — first week monitoring, first payment, ramp-up cadence

Tag each step with: who_does_it (Tiffani/Cam/Malvin/Mark/publisher), typical_timeline, blocking_dependency.
```

**Expected yield:**
- 4-5 new Publisher persona queries (onboarding-specific scenarios)
- Process documentation that informs @milo/onboarding primitive step definitions
- Closes 1 thinness flag (Publisher)

**Dependencies:** None.

---

### Gap 8: Tiffani DM-level private buyer negotiations

**Priority:** MEDIUM
**Effort for Mark:** HIGH (private channel content, requires Tiffani's involvement or Mark relaying)

**Why MEDIUM (not HIGH):** This would significantly enrich the Buyer persona's account management queries, but the existing Buyer persona is already HIGH fidelity for account management (Victor laconic, Bill Borneman IO-grade, FAB diagnostic). The private DM content would add depth, not fill a void. The HIGH effort (private channels, possibly requiring Tiffani's consent to share) makes the ROI lower per unit of Mark's time.

**Closes:** Buyer persona thinness ("No DM-level private buyer negotiations"). Would also enrich the Sales persona (Tiffani's selling voice in private is likely different from Teams).

**Extraction prompt:**
```
Here are DMs between Tiffani and a key buyer account — the private-channel version of the account management relationship: [paste with anonymization if needed].

Extract:
1. How does Tiffani's voice change in private vs. group chat? (More candid? More strategic? More profane?)
2. What topics go to DM vs. group? (Payment issues? Pricing negotiations? Complaints?)
3. Relationship management patterns — how does she handle a difficult ask privately?
4. Any intelligence she shares with the buyer that wouldn't be said publicly?
5. Escalation pattern — when does a DM conversation move to a call or to Mark?

Tag each pattern as: private_candor, private_negotiation, private_intelligence, escalation_trigger.
```

**Expected yield:**
- 3-5 new Buyer persona queries (private-channel account management)
- Tiffani's private voice register (may differ significantly from group chat)
- Escalation trigger patterns (when does an operator loop in Mark)
- Partially closes 1 thinness flag (Buyer)

**Dependencies:** Requires Tiffani's awareness or consent. Cannot be collected unilaterally by Mark.

---

### Gap 9: ConvoQC fraud analysis output

**Priority:** MEDIUM
**Effort for Mark:** LOW (single paste of a QC output report)

**Why MEDIUM:** Adversarial persona has good coverage of industry-observable scam patterns but lacks the QC-side signal — what does a fraud detection output look like when TLP's own QC system flags a call? D101 found /ask bypasses @milo/ai-client — if a ConvoQC integration is planned, the output format would define what Milo shows operators when fraud is detected in-call.

**Closes:** Adversarial persona thinness ("Could use QC-side signal"). Would also inform the Vet pill if fraud detection outputs are surfaced during entity vetting.

**Extraction prompt:**
```
Here's a sample ConvoQC output — what the system produces when it flags a call as potentially fraudulent: [paste the output].

Extract:
1. What fields does the output include? (caller ID, duration, disposition, confidence score, fraud signals)
2. What fraud signals does ConvoQC specifically detect? (scripted responses, background noise, call center indicators, duration manipulation)
3. How does the operator act on this output? (blacklist publisher? dispute call? investigate further?)
4. What's the false positive rate like? (Does the team trust ConvoQC output or verify independently?)
5. Output format — is it structured JSON, email alert, dashboard card, or chat message?

Tag each signal type as: fraud_confirmed, fraud_suspected, quality_issue, false_positive.
```

**Expected yield:**
- 3-4 new Adversarial persona queries (QC-originated fraud detection)
- 1-2 new Buyer persona queries (acting on QC alerts)
- Output format spec for potential ConvoQC → Milo integration
- Closes 1 thinness flag (Adversarial)

**Dependencies:** None.

---

### Gap 10: TCPA compliance pushback exchanges

**Priority:** MEDIUM
**Effort for Mark:** MEDIUM (find a compliance-focused contract negotiation thread)

**Why MEDIUM:** Contracts persona compliance queries are generic — the persona library covers TCPA clauses in contracts (#10: "TCPA record retention," indemnity checks in PUB checks G1-G3) but no real compliance-specific negotiation thread was captured. D100 contracts audit found the 495-line prompt has extensive TCPA/compliance rules but the persona library has no queries testing how operators negotiate compliance terms in practice.

**Closes:** Contracts persona gap ("compliance-specific contract negotiations not captured"). Also validates whether the 17 mandatory checks in `contract-analysis-prompt.ts` match real operator priorities.

**Extraction prompt:**
```
Here's a contract negotiation thread where TCPA compliance was a key sticking point: [paste thread].

Extract:
1. What specific TCPA clause was in dispute? (Record retention period? Liability shift? Consent proof access?)
2. What was TLP's position? What was the counterparty's position?
3. How was it resolved? (Accepted as-is? Modified? Removed?)
4. Did TLP use any industry standards or legal references to support their position?
5. How does this compare to the standard TCPA checks in TLP's contract review template?

Tag each item as: tcpa_retention, tcpa_liability, tcpa_consent, tcpa_indemnity, resolution.
```

**Expected yield:**
- 3-4 new Contracts persona queries (compliance-specific negotiation)
- Real-world validation of `contract-analysis-prompt.ts` TCPA rules
- Closes 1 thinness flag (Contracts)

**Dependencies:** None, but benefits from Gap 1 (Mark's Loom) landing first for full context.

---

### Gap 11: Call audit / DQ redline conversations

**Priority:** LOW
**Effort for Mark:** MEDIUM (gather an internal call quality audit thread)

**Why LOW:** Buyer persona quality-related queries are framework-based but the framework is sound — "pull call recordings, check durations, compare to thresholds" is operationally correct even without sourced examples. The existing persona library already covers the diagnostic pattern (FAB diagnostic voice from Dump 6, Victor/Nir underperformance arc). Real call audit data would add depth but wouldn't change the framework or reveal gaps.

**Extraction prompt:**
```
Here's an internal call audit or DQ (disqualification) review thread — where the team reviewed call quality and made decisions on specific calls: [paste thread].

Extract:
1. What triggered the audit? (Buyer complaint? Routine QC? Pattern detection?)
2. What data was reviewed? (Call recordings? Duration logs? IVR pass rates? Disposition codes?)
3. How were individual calls adjudicated? (Qualified? DQ'd? Disputed?)
4. What was the aggregate finding? (Publisher quality issue? Buyer-side problem? Routing error?)
5. What action was taken? (Credits issued? Publisher warned? Routing adjusted?)

Tag each element as: trigger, data_reviewed, call_adjudication, aggregate_finding, action_taken.
```

**Expected yield:**
- 2-3 new Buyer persona queries (real audit scenarios)
- Call quality terminology confirmation (DQ criteria, disposition codes)
- Partial enrichment of Buyer persona dispute handling

**Dependencies:** None.

---

### Gap 12: Telegram scam chat content

**Priority:** LOW
**Effort for Mark:** LOW (screenshot or copy-paste a Telegram scam attempt)

**Why LOW:** Adversarial persona already covers the Alex Edward WhatsApp/data-dialer pattern (Dump 2) and social engineering patterns from Dumps 4-5. Real Telegram content would increase fidelity but the pattern is already well-characterized. The persona library already has adversarial query #20 (social engineering via authority claims) which is synthetic but structurally sound.

**Extraction prompt:**
```
Here's a Telegram scam chat — someone trying to sell TLP something or impersonate a legitimate operator: [paste or screenshot].

Extract:
1. Opening message — how did they initiate? (Cold DM? Group chat crossover? Referral claim?)
2. Pitch content — what were they selling? (Data? Dialer services? "Premium leads"?)
3. Pressure tactics — urgency, exclusivity claims, "act now" language?
4. Red flag signals — poor English, refusal to use official channels, pushing for WhatsApp/Telegram only
5. How did TLP respond? (Ignored? Engaged to gather intel? Reported?)

Tag each signal as: opening_tactic, pitch_content, pressure_signal, language_tell, response_pattern.
```

**Expected yield:**
- 2-3 new Adversarial persona queries (real Telegram scam patterns)
- Enriched communication channel signal library
- Closes 1 thinness flag (Adversarial — but already LOW thinness)

**Dependencies:** None.

---

## TOP 3 RECOMMENDED FOR IMMEDIATE COLLECTION

| Rank | Gap | Why first | Effort | Yield |
|---|---|---|---|---|
| 1 | **Gap 4: MOP analyzer reasoning-stripped format** | Lowest effort (single paste), closes a format gap that blocks Contracts pill implementation. 5-minute task for Mark. | LOW | 2-3 refined queries + definitive format spec |
| 2 | **Gap 2: Cam/Vee LinkedIn DMs** | Highest persona-quality jump (Sales MEDIUM → HIGH). LinkedIn is a daily outreach channel — this is the biggest voice gap. | LOW-MEDIUM | 8-10 new queries, 2-3 voice patterns |
| 3 | **Gap 1: Mark's Loom transcript** | Contracts persona decision framework moves from inferred to sourced. Also validates whether the 60+ rule prompt matches operator training. Resolves a D100 audit question. | MEDIUM | 8-12 new queries, definitive heuristic tree |

---

## ORDER OF COLLECTION

Mark works through gaps in this order (optimized for: effort ascending within priority tiers, dependency chains respected):

1. **Gap 4** — MOP stripped format (5 min, single paste)
2. **Gap 2** — Cam/Vee LinkedIn DMs (15-30 min, copy-paste threads)
3. **Gap 1** — Mark's Loom transcript (30-60 min, transcribe or dictate)
4. **Gap 3** — Real dispute arc (30 min, find and paste thread)
5. **Gap 9** — ConvoQC output (10 min, single paste)
6. **Gap 12** — Telegram scam content (5 min, screenshot)
7. **Gap 5** — Malvin end-to-end pitch (20 min, gather thread)
8. **Gap 6** — Email thread samples (20 min, find and redact)
9. **Gap 7** — Publisher onboarding flow (30 min, document steps)
10. **Gap 10** — TCPA compliance pushback (30 min, find thread)
11. **Gap 11** — Call audit / DQ redlines (20 min, gather thread)
12. **Gap 8** — Tiffani private DMs (variable, requires Tiffani involvement)

---

## ESTIMATED PERSONA LIBRARY V2 SIZE

| Persona | V1 count | Estimated new queries from Snapshot 2 | V2 projected |
|---|---|---|---|
| Vet | 40 | +2-3 (QC-originated fraud, minimal new) | 42-43 |
| Sales | 30 | +12-16 (LinkedIn DMs, email register) | 42-46 |
| Publisher | 30 | +9-12 (Malvin pitch, onboarding, recruitment) | 39-42 |
| Buyer | 30 | +8-13 (dispute arc, DM negotiations, QC alerts) | 38-43 |
| Contracts | 30 | +13-19 (Loom heuristics, format confirmation, TCPA, email responses) | 43-49 |
| Adversarial | 20 | +5-7 (Telegram, QC fraud, compliance probing) | 25-27 |
| **Total** | **180** | **+49-70** | **229-250** |

**V2 thinness projection:** All 6 personas reach LOW thinness (currently only Vet and Adversarial are LOW). Sales jumps most dramatically (MEDIUM → LOW) with LinkedIn DMs landing.

---

Report at: https://raw.githubusercontent.com/moscaleflow/conductor-mark-public/main/reports/2026-04-24-coder-3-synth-snapshot-2-plan.md
