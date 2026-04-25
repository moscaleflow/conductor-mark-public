# Vet Pill Voice Audit Post-Part 1.5 — D106 HIGH Refinement Assessment

**Coder-3 Research | 2026-04-24**
**Scope:** Pure audit. No code. Assess whether D106's 5 HIGH refinements still apply after Part 1.5 shifted Vet from wall-of-text to card-first output.
**Cross-references:** D106 (voice consistency audit), D116 (Vet Part 1), Part 1.5 commit 10f6e8d.

---

## Part 1.5 Prompt Delta

Part 1.5 (commit 10f6e8d) changed the CONTRACT VET MODE section of `vet.ts`. The key shift:

| Part 1 instruction | Part 1.5 instruction |
|---|---|
| "Translate findings into your voice. You found these issues. Own them." | "The cards handle detail. Give a 3-4 line verdict. You're the closing voice." |
| Milo produces full RED FLAGS / GREEN FLAGS / CONFIDENCE breakdown for contracts | Milo produces 3-4 line verdict only — cards handle per-issue detail |
| ~200-400 word contract response | ~50-80 word contract response |

The entity vet flow (non-contract queries) is **unchanged** — still produces the full RED FLAGS / GREEN FLAGS / CONFIDENCE format.

---

## D106 HIGH Refinement Assessment

### HIGH-01: Normalize experience claims across all 4 pills

**D106 target:** Publisher L36 "recruited 500 publishers, because I have" / Buyer L51 "managed 200 buyer accounts, because I have"

**Verdict: APPLIES (unchanged)**

This proposal targets Publisher and Buyer pills, not Vet. The Vet pill never had contradictory experience claims — it has "You've seen thousands of entities come through this industry" which is appropriately vague and non-contradictory. Part 1.5 didn't touch this line, and it remains consistent.

The core problem (cross-pill identity fracture from specific contradictory numbers) is entirely outside the Vet pill and entirely outside Part 1.5's scope. Still applies as-written to Publisher + Buyer.

---

### HIGH-02: Add shared anti-hedging instruction to Sales, Publisher, Buyer

**D106 target:** Only Vet has "'I have concerns' is weak." Other 3 pills have zero anti-hedging.

**Verdict: APPLIES (unchanged)**

This proposal targets Sales, Publisher, and Buyer — NOT Vet. The Vet pill's anti-hedging line (STYLE section L53: `Be direct. "This smells wrong" is fine. "I have concerns" is weak.`) is present and unchanged post-Part 1.5.

However, a **new voice concern emerges**: the 3-4 line contract verdict is so short that hedging in that context is less likely — but Milo's entity vet responses (the full RED/GREEN/CONFIDENCE format) still benefit from anti-hedging since those are the longer prose outputs.

Still applies as-written. No Part 1.5 impact on this proposal.

---

### HIGH-03: Add out-of-scope deflection to ALL 4 pills

**D106 target:** No pill has deflection instructions for off-topic queries.

**Verdict: APPLIES (unchanged)**

Part 1.5 only modified the CONTRACT VET MODE section. The broader Vet prompt still has no out-of-scope deflection. If someone drops a contract and then asks "what's the weather?", Milo has no instruction to redirect in character.

The card-first format doesn't address this at all — it only changes how contract analysis responses look, not how off-topic queries are handled.

Still applies as-written. Recommended text unchanged: "Not my department. I do vetting. Give me something in that lane."

---

### HIGH-04: Add posture anchor to Sales, Publisher, Buyer preambles

**D106 target:** Only Vet has "Your default posture is skepticism." Other 3 pills lack emotional anchors.

**Verdict: APPLIES (unchanged)**

This proposal targets Sales, Publisher, and Buyer — NOT Vet. The Vet posture anchor (L5-6: `Fraud is the base rate in pay-per-call. Your default posture is skepticism.`) is present and unchanged post-Part 1.5.

The proposed anchors for other pills ("operator-first" for Sales, "quality-first" for Publisher, "TLP-first" for Buyer) are independent of Vet's format change.

Still applies as-written.

---

### HIGH-05: Add TLP voice register awareness to Sales pill

**D106 target:** Sales pill has no TLP team voice registers (po tita, penguin culture, FAB team voice).

**Verdict: APPLIES (unchanged)**

This proposal targets Sales only. The Vet pill has never needed voice register awareness — vetting doesn't involve mimicking team communication styles.

Part 1.5 has zero impact on this proposal.

---

### Summary: D106 HIGH Refinements vs Part 1.5

| Proposal | Target Pill(s) | Verdict | Rationale |
|---|---|---|---|
| HIGH-01: Normalize experience claims | Publisher, Buyer | **APPLIES** | Targets non-Vet pills, untouched by Part 1.5 |
| HIGH-02: Anti-hedging | Sales, Publisher, Buyer | **APPLIES** | Vet's anti-hedging unchanged, other pills still lack it |
| HIGH-03: Out-of-scope deflection | All 4 | **APPLIES** | Card format doesn't address off-topic handling |
| HIGH-04: Posture anchors | Sales, Publisher, Buyer | **APPLIES** | Vet's anchor unchanged, other pills still lack them |
| HIGH-05: TLP voice registers | Sales | **APPLIES** | Targets Sales only, irrelevant to Part 1.5 |

**Result: 5 APPLIES, 0 PARTIAL, 0 OBSOLETE.**

All 5 HIGH proposals survive Part 1.5 because they either target non-Vet pills or target aspects of the Vet prompt (identity, style, deflection) that Part 1.5 didn't modify. Part 1.5 only changed the CONTRACT VET MODE section — the broader voice infrastructure is untouched.

---

## New Voice Concerns Post-Part 1.5

### Concern 1: "You're the closing voice" is stage direction, not character

**Severity: HIGH**

Part 1 had: *"Do NOT dump the raw analysis data. Translate it into your voice. You found these issues. Own them."*

Part 1.5 replaced it with: *"Keep it to 3-4 lines max. The detail lives in the cards. You're the closing voice."*

"Own them" was peak Milo — it told the model to feel ownership over the findings, producing verdicts like "This MSA is a minefield." "You're the closing voice" is an instruction about output format, not about character. It reads like a director's note, not a personality injection.

**Proposed fix:** Replace "You're the closing voice." with:

```
You found these issues. The cards lay them out. Your job is the gut check — 
would you sign this? Say so like you mean it.
```

This preserves Part 1.5's "don't repeat the cards" instruction while restoring the ownership voice that made Milo's contract verdicts distinctive.

---

### Concern 2: Per-issue card voice — `plain_summary` is Milo voice, but `problem_explanation` may not be

**Severity: MEDIUM**

IssueCard renders two text fields that operators read:
- **Collapsed (always visible):** `plain_summary` — "No deadline for rejecting bad leads. They can delay your rejections forever and you're stuck paying." This IS Milo voice: direct, uses "you" and "they," no hedging.
- **Expanded (click to see):** `problem_explanation` — multi-paragraph reasoning with "Additionally:" chains. This is analytical but NOT distinctly Milo. It reads more like a legal brief than a cocky intelligence layer.

The issue: `problem_explanation` is Claude's reasoning output, shaped by the analysis system prompt (in route.ts L327-336), NOT by the vet.ts personality prompt. The analysis prompt says to produce structured JSON — it has no voice instructions. So the expanded card view reads as "generic contract AI" rather than "Milo."

**Proposed fix:** Add a one-line Milo voice instruction to the analysis system prompt in route.ts:

```
Write problem_explanation and plain_summary in a direct, operator-facing voice. 
Use "you" and "they." No legalese, no hedging. Say what's wrong and why it matters 
to a pay-per-call network operator.
```

This doesn't require vet.ts changes — it's an analysis prompt refinement. The `plain_summary` field is already good, but `problem_explanation` needs the same voice treatment since operators see it when they expand a card.

---

### Concern 3: Milo personality lost in the gap between cards and verdict

**Severity: MEDIUM**

In Part 1, the entire response was Milo's voice — RED FLAGS, GREEN FLAGS, CONFIDENCE RATING, closing verdict. The personality was distributed across 200-400 words.

In Part 1.5, cards are structured UI elements (severity badges, clause references, action buttons) with NO Milo personality — they're data display. Milo's personality is concentrated into 3-4 lines of verdict text that streams in AFTER the cards.

The visual flow is: `[structured cards — neutral voice] → [Milo verdict — 3-4 lines]`.

If the cards handle 90% of the content and Milo's text is 10%, there's a personality dilution problem. The operator gets a thorough card breakdown in generic voice, then a brief Milo sign-off. The "slightly cocky, direct" personality that makes Milo distinct from ChatGPT-wrapped-in-a-contract-tool is squeezed into 50 words.

**Proposed fix (pick one):**

**Option A — Milo greeting before cards:** Add a 1-line Milo intro that streams BEFORE the cards render. Example: "Alright, I ran CrossBlade's MSA through the engine. 11 issues, 3 critical. Here's the breakdown:" This gives Milo ownership from the start, not just the ending.

**Option B — Card header voice:** The IssueListExpandable header currently shows a neutral summary bar. Add a brief Milo-voiced subheader like: "3 are deal-breakers. Start with those." This injects personality into the card container without changing per-issue content.

**Option C — Verdict anchor line:** Add a closing signature pattern to the CONTRACT VET MODE instructions. After the confidence rating + recommendation, allow a 1-sentence Milo-ism: "Don't let them tell you this is standard — none of this is standard." This anchors the personality without expanding the word count.

**Recommendation:** Option A + Option C combined. Milo bookends the cards — intro before, sign-off after. Cards stay neutral/structured in between.

---

### Concern 4: `formatAnalysisContext` only passes CRITICAL + HIGH issues

**Severity: LOW (voice-adjacent, more of a capability gap)**

The `formatAnalysisContext()` function in route.ts (L198-219) only includes CRITICAL and HIGH issues in the KEY ISSUES list that Claude sees. MEDIUM and LOW issues are counted but their titles/details are omitted.

If a contract has 0 critical/high but 6 medium issues, Milo's verdict has no issue names to reference — just "Total Issues: 6 (0 critical, 0 high, 6 medium, 0 low)." The verdict becomes vague: "6 medium issues across this contract" vs. the specific "3 critical issues in the liability and termination sections" that the example in the prompt shows.

**Proposed fix:** Include MEDIUM issue titles (not details) in the context:

```
KEY ISSUES:
- [CRITICAL] No Defined Rejection Window (Section 5.4): ...
- [HIGH] Data Ownership Not Defined: ...

ALSO FLAGGED:
- [MEDIUM] Auto-Renewal Without Notice (Section 12.1)
- [MEDIUM] Unilateral Rate Change (Section 7.3)
```

This gives Milo enough context to reference medium issues by name without bloating the prompt.

---

### Concern 5: Explain-further route uses raw Anthropic SDK

**Severity: LOW (technical hygiene, not voice)**

Both `/api/ask/explain-further` and `/api/ask/refine-redline` instantiate `new Anthropic()` directly instead of using `@milo/ai-client`. This means:
- No MODEL_QUIRKS protection (Pattern 15)
- No tier-based routing
- No spend tracking
- Hardcoded model: `claude-sonnet-4-6-20250514`

The explain-further route DOES have Milo voice instructions ("Be direct, slightly cocky, and useful. Not cute, not corporate.") so the voice is present. But if ai-client's model reference changes (e.g., Sonnet version bump), these routes won't follow.

This is a D101-style wireup gap, not a voice concern. Flagging for Coder-1 awareness.

---

### Concern 6: web_search tool still active in contract vet mode

**Severity: LOW**

When a contract is dropped, the vet prompt's tool configuration still includes `web_search` with `max_uses: 5`. But the CONTRACT VET MODE instruction says to give a 3-4 line verdict. Milo might web-search the counterparty name ("Let me search for CrossBlade Media...") when it should just summarize the analysis.

Whether this actually happens depends on Claude's judgment, but the prompt doesn't explicitly say "do NOT web search in contract vet mode." If it does fire, it adds latency to the verdict and the operator sees a "Searching..." status while waiting for a 3-line response.

**Proposed fix:** Add to CONTRACT VET MODE: "Do not web search — the analysis engine already ran. Just give your verdict based on what the engine found."

---

## Emoji Marker Migration Check

**D106 confirmed:** Vet prompt uses `🚩` in RED FLAGS header and `✅` in GREEN FLAGS header as LOCKED format markers.

**Post-Part 1.5 status:**
- **Entity vet (non-contract):** Unchanged. Full RED FLAGS 🚩 / GREEN FLAGS ✅ / CONFIDENCE format still produced. Emoji markers preserved.
- **Contract vet:** Cards use `SeverityBadge` component (color-coded pill: red CRITICAL, orange HIGH, yellow MEDIUM, blue LOW). No emoji markers. This is correct — cards are UI components, not text output. The emoji markers only applied to the text-based RED/GREEN format.
- **Verdict text:** The 3-4 line verdict does NOT use emoji markers. The CONFIDENCE RATING line has no emoji. This is correct — the verdict is a brief closing, not a formatted list.

**Conclusion:** 🚩/✅ markers are preserved for entity vetting (where they apply) and correctly absent from card-based contract vetting (where SeverityBadge handles severity display). No migration needed.

---

## Top 3 Recommended Vet Voice Refinements (Part 1.5 state)

Ranked by impact on operator experience:

### 1. Restore ownership voice in CONTRACT VET MODE (HIGH)

**Current:** "Keep it to 3-4 lines max. The detail lives in the cards. You're the closing voice."

**Proposed:**
```
Keep it to 3-4 lines max. The detail lives in the cards.
You found these issues. The cards lay them out. Your job is the gut check — 
would you sign this? Say so like you mean it.
```

**Why:** "You're the closing voice" is format instruction. "Would you sign this? Say so like you mean it" is character. This single edit restores the personality density that Part 1 had in "Own them" without re-introducing wall-of-text.

### 2. Bookend the cards with Milo intro + sign-off (MEDIUM)

**Proposed:** Update the analysis/streaming flow so Milo produces a 1-line intro BEFORE cards render:
- Intro (streams first): "Alright, I ran [counterparty]'s [document_type] through the engine. [N] issues, [N] critical. Here's the breakdown:"
- Cards render (UI components, no Milo text)
- Verdict (streams after): the 3-4 line gut check

**Why:** Without the intro, the operator sees cards appear silently, then Milo's voice shows up only at the end. With the intro, Milo "owns" the entire experience from first word to last.

**Implementation note:** This requires a prompt change (add intro instruction to CONTRACT VET MODE) plus a rendering change (stream intro text, emit analysis SSE, stream verdict). Coder-1 scopes this.

### 3. Suppress web_search in contract vet mode (LOW)

**Proposed:** Add to CONTRACT VET MODE:
```
Do not web search in contract vet mode. The analysis engine already examined 
the document. Your verdict is based on what the engine found, not on web results 
about the counterparty. If the operator wants entity vetting, they'll ask separately.
```

**Why:** Prevents latency-adding web searches during what should be a fast 3-4 line verdict. Also prevents voice confusion — a web-searching Milo sounds like entity-vet Milo, not contract-vet Milo.

---

## Summary

| D106 HIGH | Verdict | Rationale |
|---|---|---|
| HIGH-01: Normalize experience claims | **APPLIES** | Targets Publisher + Buyer, untouched |
| HIGH-02: Anti-hedging | **APPLIES** | Targets Sales/Publisher/Buyer, Vet's anti-hedging intact |
| HIGH-03: Out-of-scope deflection | **APPLIES** | Card format doesn't address off-topic |
| HIGH-04: Posture anchors | **APPLIES** | Targets non-Vet pills |
| HIGH-05: TLP voice registers | **APPLIES** | Targets Sales only |

**All 5 D106 HIGH refinements survive Part 1.5.** None are obsolete because none targeted the CONTRACT VET MODE section that Part 1.5 changed.

**3 new voice refinements proposed** for current Part 1.5 state, all targeting the contract vet experience specifically.

---

Cross-references: Decision 106 (voice consistency audit), Decision 116 (Vet Part 1), Part 1.5 commit 10f6e8d, D118 (MOP stripped format — aligns with card field structure).
