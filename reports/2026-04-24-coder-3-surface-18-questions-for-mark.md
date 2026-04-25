# Surface 18 Open Questions — Consolidated for Mark Batch Review

**Coder-3 Research | 2026-04-24**
**Source:** Decision 107 (Surface 18 minimal build spec, commit `5c43bd5`)
**Purpose:** Pre-resolve 11 open questions before Coder-1 fires the implementation directive. If Mark answers the 3 HIGH questions, Coder-1 can proceed immediately. MEDIUM/LOW questions ship with Coder-3's recommended defaults — Mark can override during or after build.

---

## Priority Summary

| Priority | Count | Decision timing |
|---|---|---|
| HIGH | 3 | Must answer before Coder-1 starts |
| MEDIUM | 6 | Can answer during build or accept recommended default |
| LOW | 2 | Post-v1 polish |

---

## HIGH — Blocks Coder-1 Build

### Q-01: Should `fileUpload` replace `contractFile` or coexist?

- **Context:** The current /ask route accepts `contractFile: { name, data }` for contract PDF/DOCX drops (D104). Surface 18 needs a general-purpose field `fileUpload: { name, data, mime }` that handles contracts, images, and future file types. The question is whether to replace the existing field (breaking change to the transport layer) or keep both (backward compatibility, more code).
- **Spec section:** §9, question 8
- **Current spec position:** REPLACE recommended
- **Options:**
  - **A. REPLACE** — `fileUpload` becomes the sole transport field. Route A detects contracts by MIME, not by field name. If `contractFile` is still sent (old client cache), backend treats it as `fileUpload` with `mime: 'application/pdf'`. Clean, one code path.
  - **B. COEXIST** — Keep `contractFile` for backward compat, add `fileUpload` for new routes. Two code paths. `contractFile` takes priority if both present. More defensive but messier long-term.
  - **C. RENAME ONLY** — Keep the same `{ name, data }` shape, just add `mime` to the existing `contractFile` field and rename it. Minimal diff, but the name "contractFile" becomes misleading when carrying LinkedIn screenshots.
- **Coder-3 recommendation:** **A. REPLACE.** `fileUpload` is a strict superset. The backward-compat shim (detect `contractFile` → treat as `fileUpload`) is 3 lines. D104 shipped recently (same day) — no long-lived caches to worry about. One field, one code path, clean.
- **Priority:** HIGH — determines the request body shape for all routes. If Coder-1 builds on the wrong shape, every route needs refactoring.
- **When Mark needs to decide:** Before build

---

### Q-02: LinkedIn screenshot routing — Milo decides automatically or always asks?

- **Context:** When a LinkedIn profile screenshot is dropped, the classifier extracts name + title + company. The question is whether Milo auto-routes to Sales/Publisher/Buyer based on the extracted title, or always asks the operator which pill to use. This determines whether Coder-1 builds keyword-based pill routing logic or a mandatory confirmation flow.
- **Spec section:** §9, question 4
- **Current spec position:** AUTO-DECIDE recommended
- **Options:**
  - **A. AUTO-DECIDE** — Vision classifier extracts title. Keyword matching: buyer/demand/advertiser → Buyer pill. Publisher/affiliate/media-buying/supply → Publisher pill. Ambiguous → Sales pill (default). Override link always visible. Fast: operator drops, Milo routes, draft appears.
  - **B. ALWAYS ASK** — After extraction, Milo asks: "I pulled [Name] at [Company]. Route to: [Sales | Publisher | Buyer]?" Operator clicks. Slower but guaranteed correct routing.
  - **C. AUTO-DECIDE WITH THRESHOLD** — Auto-route if title keyword confidence is high. Ask if title is ambiguous (e.g., "Director of Partnerships" could be buyer or publisher). Hybrid: fast for clear cases, safe for gray areas.
- **Coder-3 recommendation:** **A. AUTO-DECIDE.** The override link is always visible, so wrong routing is one click to fix. Asking every time adds friction to a workflow that should feel like magic (drop screenshot → Milo acts). The classifier can default to Sales for ambiguous titles — Sales is the broadest pill and handles outreach to any role. D106 voice audit (HIGH-03) proposed out-of-scope deflection for all pills — the same defensive layer protects against misrouted LinkedIn queries.
- **Priority:** HIGH — determines Route B's core architecture. Auto-decide requires keyword mapping logic. Always-ask requires confirmation UI flow. Different code paths.
- **When Mark needs to decide:** Before build

---

### Q-03: Should `fileUpload` field auto-submit like contracts, or wait for operator message?

- **Context:** Contract drops on Vet currently auto-submit with "Vet this." (`ask/page.tsx:453-457`) — operator doesn't need to type anything. Should LinkedIn and scam-chat drops auto-submit the same way? Or should they wait for the operator to add context ("Evaluate this person for publisher recruitment" vs just dropping the file)?
- **Spec section:** §9, question 6
- **Current spec position:** Auto-submit recommended
- **Options:**
  - **A. AUTO-SUBMIT ALL ROUTES** — Drop file → classifier runs → response appears. Same UX as contracts. Default messages: LinkedIn → "I dropped a LinkedIn profile", scam chat → "I dropped a scam chat screenshot". Fast, zero-friction.
  - **B. WAIT FOR MESSAGE** — Drop file → file attaches to input → operator types context → submit. Slower but operator can add context ("Evaluate for publisher" or "Check if this person is on the blacklist"). More precise routing.
  - **C. AUTO-SUBMIT CONTRACTS, WAIT FOR IMAGES** — Contracts are unambiguous (analyze it). Images benefit from context. Hybrid: contracts auto-submit (existing behavior preserved), images wait for operator message.
- **Coder-3 recommendation:** **A. AUTO-SUBMIT ALL ROUTES.** The whole point of Surface 18 is "drop and go." If the operator wants to add context, they can type before dropping (the input field persists). Auto-submit with a default message keeps the UX uniform across all file types. The override mechanism handles misroutes. If the operator regularly needs to add context to image drops, that's signal to revisit in v2 — but start with the fastest path.
- **Priority:** HIGH — determines whether the client auto-submits or waits. Different event handling code paths on the client.
- **When Mark needs to decide:** Before build

---

## MEDIUM — Surfaces During Build

### Q-04: Confidence score visibility in classifier banner

- **Context:** The classifier produces a confidence score (0.0–1.0). The question is whether operators see "I see a contract from Granite Media (87% confident)" or just "I see a contract from Granite Media."
- **Spec section:** §9, question 1
- **Current spec position:** HIDE recommended
- **Options:**
  - **A. HIDE** — Operators see only the natural language banner. Confidence logged to `context_metadata` for admin analysis on /review. Cleaner UX, less cognitive load.
  - **B. SHOW** — Operators see the confidence score. Transparency builds trust. But numbers like "0.87" feel robotic and operators may distrust anything below 0.95.
  - **C. SHOW AS QUALITATIVE** — No number, but the banner language shifts: "I'm pretty sure this is a contract" (high) vs "I think this might be a contract" (low). Voice-consistent per D106.
- **Coder-3 recommendation:** **A. HIDE.** Operators don't need to see numbers. The low-confidence confirmation prompt (< 0.85) already handles uncertainty by asking for confirmation. High-confidence responses just proceed. The /review page shows confidence for admin analysis. Cross-ref D106: Milo's voice should be confident ("I see a contract") not hedging ("I'm 87% sure this is a contract").
- **Priority:** MEDIUM — doesn't change architecture. CSS/copy change. Coder-1 can build with HIDE and toggle later.
- **When Mark needs to decide:** During build

---

### Q-05: Override UX — inline buttons or dropdown?

- **Context:** When the classifier routes a file, the operator needs to be able to correct it. The question is UX shape: inline text link in the chat ("Wrong? Route to Vet | Sales | Publisher | Buyer") or a dropdown selector.
- **Spec section:** §9, question 2
- **Current spec position:** INLINE recommended
- **Options:**
  - **A. INLINE text link** — Small "[Wrong? Route to ___]" beneath the classified banner with clickable pill names. Feels like chat, not a form. Matches /ask's conversational surface.
  - **B. DROPDOWN** — Select element with pill options. More compact. But feels like a form control, not a chat interaction. Inconsistent with /ask's conversational UX.
  - **C. INLINE BUTTONS** — Styled pill buttons (matching the pill selector at the top of /ask). Not a text link, not a dropdown — actual buttons. More discoverable than a text link, more chat-native than a dropdown.
- **Coder-3 recommendation:** **C. INLINE BUTTONS.** Text links are too subtle — operators might miss them. Dropdowns feel like forms. Small styled pill buttons (like the existing pill selector, but smaller, beneath the banner) are the right middle ground. Reuses the existing pill selector component pattern. Touch-friendly on mobile (44px targets per §4c).
- **Priority:** MEDIUM — UI component shape. Coder-1 can build with inline buttons and adjust later.
- **When Mark needs to decide:** During build

---

### Q-06: /ask only or /operator too for v1?

- **Context:** Surface 18 could also work on /operator surfaces (entity detail panel, contract-review page). Adding /operator doubles the scope — different file handling, different contexts, different UI patterns.
- **Spec section:** §9, question 3
- **Current spec position:** /ask only recommended
- **Options:**
  - **A. /ask only** — Ship Surface 18 on /ask. /operator keeps its existing file handling. Unify in v2 if the pattern proves valuable.
  - **B. /ask + /operator** — Ship on both surfaces. /operator file drops use the same classifier but render results in the operator UI context (entity detail panel, contract-review page). More scope, more value.
- **Coder-3 recommendation:** **A. /ask only.** /operator has its own file handling architecture (contract-review page, entity detail panel). Adding Surface 18 to /operator means building a second rendering context for every route. Ship v1 on /ask, prove the pattern, then port the classifier to /operator in v2 if operators want it there.
- **Priority:** MEDIUM — determines scope. If Mark wants /operator, Coder-1 needs to plan for 2× the UI work. But the recommendation (/ask only) is the safe default.
- **When Mark needs to decide:** During build

---

### Q-07: Scam-chat screenshot — auto-blacklist or always confirm?

- **Context:** When a scam-chat screenshot mentions a negative entity, should Milo auto-add to the blacklist if confidence is very high (> 0.95), or always require operator confirmation?
- **Spec section:** §9, question 5
- **Current spec position:** ALWAYS CONFIRM recommended
- **Options:**
  - **A. ALWAYS CONFIRM** — "Add to blacklist?" with one-click confirm for every entity. Slower but safe. Blacklisting is irreversible in practice (entity is blocked from doing business, reputation signal).
  - **B. AUTO-BLACKLIST > 0.95** — If the classifier is very confident AND the entity name matches blacklist patterns (e.g., already flagged in scam chat), auto-add. Operator sees "Added [entity] to blacklist" notification. Faster but riskier — OCR could misread a name.
  - **C. AUTO-DRAFT, OPERATOR COMMITS** — Milo pre-fills the blacklist entry form (entity name, reason, evidence from screenshot) but doesn't submit. Operator reviews and clicks "Confirm." Middle ground: Milo does the work, operator has the final say.
- **Coder-3 recommendation:** **A. ALWAYS CONFIRM.** Blacklisting is a high-consequence action. Even at 0.99 confidence, vision OCR on a phone screenshot could misread "Sky Marketing" as "Sky Markets" — different entity, wrong blacklist. One-click confirm is fast enough (< 1 second operator overhead). The D101 plug-in audit noted that /ask currently bypasses @milo/blacklist's fuzzy matching — Surface 18 should use `@milo/blacklist.check()` which catches near-misses, but writing to the blacklist should always be human-gated.
- **Priority:** MEDIUM — affects Route C action affordances. Default (always confirm) is safe. Coder-1 can build with confirm and Mark can upgrade to auto later if accuracy proves out.
- **When Mark needs to decide:** During build

---

### Q-08: Image file size limit — 5MB or 10MB?

- **Context:** Contract files have a 5MB limit (existing). Images could be up to 10MB for high-res screenshots. Larger files mean slower base64 encoding, larger request payloads, and higher vision API costs.
- **Spec section:** §9, question 7
- **Current spec position:** 5MB recommended
- **Options:**
  - **A. 5MB** — Match contract limit. Phone screenshots are typically 1-3MB. 5MB covers all normal use cases.
  - **B. 10MB** — Allow high-res captures. Some operators screenshot on Retina/HiDPI displays with 3-5MB images. 10MB provides headroom.
  - **C. 5MB WITH CLIENT RESIZE** — Accept up to 10MB from the picker, but resize to max 2000px width before base64 encoding. Reduces actual payload size while accepting large inputs. More client code, but optimal for API costs.
- **Coder-3 recommendation:** **A. 5MB.** Start conservative. If operators hit the limit, Coder-1 adds client-side resize later (Option C). Phone screenshots from the Teams/Telegram apps that operators actually use are 500KB-2MB. 5MB is generous. Revisit only if operators report "file too large" errors in production.
- **Priority:** MEDIUM — one constant in the client. Trivial to change.
- **When Mark needs to decide:** During build

---

### Q-09: Vision classifier tier — Sonnet or Haiku?

- **Context:** The image classifier needs to extract entity names from screenshots. Sonnet (standard tier) is more reliable for OCR but costs more. Haiku (quick tier) is faster and cheaper but may miss names in low-res screenshots.
- **Spec section:** §9, question 11
- **Current spec position:** START WITH SONNET recommended
- **Options:**
  - **A. SONNET** — More reliable OCR and UI pattern recognition. ~$0.03-0.05 per classification. ~2s latency.
  - **B. HAIKU** — Faster (~0.8s), cheaper (~$0.005 per classification), but may fail on low-res or complex screenshots. Higher override rate risk.
  - **C. SONNET FIRST, BENCHMARK, DOWNGRADE** — Start with Sonnet. After 50+ real classifications, measure accuracy. If Haiku hits > 85% entity extraction accuracy on the same test set, downgrade. Data-driven decision.
- **Coder-3 recommendation:** **C. SONNET FIRST, BENCHMARK, DOWNGRADE.** D101 plug-in audit specifically flagged that /ask bypasses @milo/ai-client's tier system. Surface 18 should use `callClaudeVision(tier='standard')` — which resolves to Sonnet via the tier config. If cost becomes an issue after launch, benchmark Haiku and downgrade. The tier system makes this a config change, not a code change.
- **Priority:** MEDIUM — tier selection is a config value in the classifier call. Coder-1 builds with `tier='standard'`, Mark can change the tier later.
- **When Mark needs to decide:** After v1 ship (based on cost/accuracy data)

---

## LOW — Post-v1 Polish

### Q-10: CRM lead source value for LinkedIn drops

- **Context:** The `LeadSource` enum is `'manual' | 'scrape' | 'referral' | 'inbound' | 'research'`. There's no `'linkedin-drop'` value. The question is whether to add a new enum value (schema change) or tag via metadata.
- **Spec section:** §9, question 9
- **Current spec position:** Use `'research'` with metadata tag
- **Options:**
  - **A. Use `'research'` + metadata** — `source: 'research'`, `metadata: { surface18_source: 'linkedin-drop' }`. Zero schema changes. Filtering by source requires querying metadata JSONB.
  - **B. Add `'linkedin-drop'` to enum** — Schema change to `crm_leads` source column. Clean filtering. But requires Decision 38 SCHEMA.md amendment process + migration.
- **Coder-3 recommendation:** **A. Use `'research'` + metadata.** Zero schema changes aligns with Surface 18's "zero new primitives, zero new tables" constraint. The metadata tag captures the source for analytics. If filtering by `linkedin-drop` becomes a frequent need, add the enum value in a future directive. The `metadata` JSONB field exists specifically for this — freeform context that doesn't warrant a schema change yet.
- **Priority:** LOW — `'research'` with metadata works. No build blocker, no user-facing difference.
- **When Mark needs to decide:** After v1 ship (if filtering by source matters)

---

### Q-11: Multi-file drops — v1 support or defer?

- **Context:** What if an operator drops 3 files at once? The current handler takes only `e.dataTransfer?.files[0]`. Multi-file would need sequential classification + merged or separate responses.
- **Spec section:** §9, question 10
- **Current spec position:** DEFER to v2
- **Options:**
  - **A. DEFER** — v1 is single-file only. If operator drops multiple files, only the first is processed. Clear error: "I can handle one file at a time. Drop them one by one."
  - **B. SEQUENTIAL** — Accept multiple files, classify each sequentially, return multiple responses. More complex client + backend state management.
  - **C. FIRST FILE + WARN** — Process the first file, show a toast: "Got [filename1]. I can only handle one file at a time — drop the others separately." Polite single-file with visibility.
- **Coder-3 recommendation:** **A. DEFER with C's copy.** Single-file only in v1, but use Option C's friendly message rather than silently dropping extra files. "Got [filename]. I can only handle one at a time — drop the others separately." This is a UI copy decision, not an architecture decision. Defer multi-file to v2 when usage patterns reveal whether batch drops are common.
- **Priority:** LOW — explicitly deferred to v2 in the spec. Not a build consideration.
- **When Mark needs to decide:** After v1 ship (if operators request batch drops)

---

## Cross-References

### D101 (Plug-in gap audit) overlap

- **Q-09 (vision tier):** D101 flagged that /ask bypasses `@milo/ai-client` entirely (raw Anthropic SDK). Surface 18's classifier MUST use `callClaudeVision` from `@milo/ai-client` — not raw `sdk.messages.create()`. This gives tier policy, MODEL_QUIRKS protection, and retry logic for free. Q-09's recommendation (use `tier='standard'`) aligns with D101's recommendation to route all AI calls through the primitive.
- **Q-07 (blacklist auto-action):** D101 flagged that /ask's blacklist pre-check uses raw Supabase queries, missing `@milo/blacklist`'s fuzzy matching. Surface 18's blacklist screening for Routes B + C MUST use `@milo/blacklist.check()` — not direct Supabase queries. This is already spec'd correctly in D107 §2 Route B/C.

### D106 (Voice consistency audit) overlap

- **Q-04 (confidence visibility):** D106 HIGH-02 proposed anti-hedging across all pills ("Be definitive, not tentative"). Showing confidence scores in the classified banner would force hedging language ("I'm 87% confident"). HIDE aligns with Milo's confident voice — "I see a contract from Granite Media" is Milo; "I'm 87% sure this is a contract" is not.
- **Q-02 (LinkedIn auto-routing):** D106 HIGH-04 proposed posture anchors ("operator-first" for Sales, "quality-first" for Publisher, "TLP-first" for Buyer). If LinkedIn auto-routing sends a publisher profile to the Buyer pill by mistake, the posture anchor would shape the response incorrectly. The override mechanism mitigates this, but it reinforces why the keyword mapping in Q-02 needs to be conservative (default to Sales for ambiguous titles).

---

## Decision Protocol for Mark

**Fast path:** Answer the 3 HIGH questions. Coder-1 starts immediately with MEDIUM/LOW defaults.

**Full path:** Answer all 11. Every question has a Coder-3 recommendation — Mark can approve with "yes" or override with a specific choice.

**Shortcut:** If Mark agrees with all Coder-3 recommendations, a single "approved" resolves all 11. Coder-1 builds with the recommended defaults.

---

## Recommended Defaults Summary

| Q# | Question | Recommendation | Priority |
|---|---|---|---|
| Q-01 | fileUpload vs contractFile | A. REPLACE | HIGH |
| Q-02 | LinkedIn auto-routing | A. AUTO-DECIDE | HIGH |
| Q-03 | Auto-submit behavior | A. AUTO-SUBMIT ALL | HIGH |
| Q-04 | Confidence visibility | A. HIDE | MEDIUM |
| Q-05 | Override UX shape | C. INLINE BUTTONS | MEDIUM |
| Q-06 | /ask only scope | A. /ASK ONLY | MEDIUM |
| Q-07 | Blacklist auto-action | A. ALWAYS CONFIRM | MEDIUM |
| Q-08 | Image size limit | A. 5MB | MEDIUM |
| Q-09 | Vision tier | C. SONNET FIRST, BENCHMARK | MEDIUM |
| Q-10 | CRM lead source | A. 'research' + metadata | LOW |
| Q-11 | Multi-file drops | A. DEFER + friendly copy | LOW |

---

Report at: (to be pushed)
