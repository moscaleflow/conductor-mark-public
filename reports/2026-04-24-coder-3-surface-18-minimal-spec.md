# Surface 18 Minimal Build Spec — File-Drop Classifier-Router

**Coder-3 Research | 2026-04-24**
**Scope:** Pure spec. No code. Coder-1 implements from this in a future directive.
**Constraint:** Zero new primitives, zero new database tables. Pure plug-in over existing `@milo/*` packages.

---

## 1. Existing Infrastructure Inventory

Before speccing the new wiring, here is exactly what exists and what each piece's API looks like.

### 1a. Current /ask file handling (as shipped in D104)

The `/ask` page currently supports file drops, but ONLY for contracts on the Vet pill:

| Stage | Current behavior | Code location |
|---|---|---|
| Client detection | `.pdf` or `.docx` by filename regex | `ask/page.tsx:417` — `const isContract = /\.(pdf\|docx)$/i.test(file.name)` |
| Size limit | 5MB for contracts, 100KB for other files | `ask/page.tsx:419` |
| Encoding | Contract: `readAsArrayBuffer` → base64. Other: `readAsText` | `ask/page.tsx:425-449` |
| Transport | `contractFile: { name, data }` in JSON body | `ask/page.tsx:494` |
| Routing | Hard-coded: only fires if `pill === 'vet' && contractFile?.data` | `api/ask/route.ts:335` |
| Processing | `extractContractText(buffer, name)` → `createPpcAnalysisEngine(role).analyze(text)` | `api/ask/route.ts:338-346` |
| Classification | None — assumes PDF/DOCX on Vet is always a contract | — |
| Image support | Not supported as file drops (only text files under 100KB) | `ask/page.tsx:440-449` |
| Auto-submit | Triggers only when `selectedPill === 'vet'` | `ask/page.tsx:453` |

**Key constraint:** The current system has no image upload path. Surface 18 must add image handling (base64 encoding, MIME detection, size limits) to the client.

### 1b. Primitive APIs available

| Primitive | Method | Signature | Notes |
|---|---|---|---|
| `@milo/ai-client` | `callClaudeVision` | `(prompt, { imageBase64, mediaType, tier, system, maxTokens, temperature })` → `string` | Supports `image/png`, `image/jpeg`, `image/gif`, `image/webp`. Returns raw text. No structured output — must parse JSON from response. |
| `@milo/contract-analysis` | `createAnalysisEngine` | `(role: 'publisher'\|'buyer')` → engine with `.analyze(text, opts)` | Already wired in D104 Vet contract drop. |
| `@milo/contract-analysis` | `extractText` | `(buffer: Buffer, filename: string)` → `string` | Handles PDF (pdf-parse) and DOCX (mammoth). |
| `@milo/blacklist` | `check` | `(client, { company_name, contact_names?, email?, linkedin_url? })` → `{ matches, blocked, confidence, reason }` | `ScreenInput` accepts company_name (required), contact_names, email, ip_address, linkedin_url — all optional except company_name. |
| `@milo/crm` | `createLead` | `(client, LeadInsert)` → `Lead` | `LeadInsert` requires: `company`. Optional: `contact_name`, `contact_email`, `linkedin_url`, `website`, `industry`, `role`, `source` (enum: manual/scrape/referral/inbound/research), `notes`, `metadata`. |
| `@milo/crm` | `listCounterparties` | `(client, opts?)` → `Counterparty[]` | Can check if entity already exists before creating. |
| `@milo/feedback` | `recordFeedback` | `(client, FeedbackSignalInsert)` → `FeedbackSignal` | For routing accuracy signals. |
| `@milo/ai-client` | `callClaude` (text) | `(prompt, { tier, system, messages, tools })` → `{ text, usage }` | For non-vision classification (PDF content sniffing after text extraction). |

### 1c. Existing /ask auto-classifier

The current auto-classifier (`pill === 'auto'`) uses a text-based `classifyInput(message, sdk)` call that returns `{ pill, confidence: 'high'\|'low' }`. For low confidence, it returns a clarifier message instead of proceeding. This same pattern extends to Surface 18's file classifier.

### 1d. chat_logs schema (current columns relevant to Surface 18)

| Column | Type | Current use | Surface 18 reuse |
|---|---|---|---|
| `pill` | TEXT | Which pill handled the query | Already stores final destination |
| `input_text` | TEXT | User's text message | Add file context |
| `attachments` | JSONB | Currently `[]` in most cases | Store `file_meta` here |
| `context_metadata` | JSONB | CRM context from D102 | Store `classifier_decision` here |
| `analysis_id` | TEXT | Contract analysis ID from D104 | Already used for contracts |
| `admin_flag` | BOOLEAN | Empty response flag | Repurpose not viable — different semantics |

**New columns needed (on existing chat_logs table — NOT a new table):**
- `operator_override` BOOLEAN DEFAULT FALSE — did operator change routing?
- `final_destination` TEXT — where the file actually went after override

`file_meta` goes into existing `attachments` JSONB. `classifier_decision` goes into existing `context_metadata` JSONB (extend the shape). This avoids creating unnecessary columns while capturing all instrumentation data.

---

## 2. Routing Table

### Route A: Contract PDF / DOCX

| Column | Value |
|---|---|
| **Detection rule** | MIME `application/pdf` or `application/vnd.openxmlformats-officedocument.wordprocessingml.document`. Client-side extension check: `.pdf` or `.docx`. Size limit: 5MB (existing). |
| **Classification step** | Text extraction via `@milo/contract-analysis.extractText(buffer, filename)`. Then `callClaude` with classifier prompt: "Given this extracted text from a document, is this a legal agreement, contract, MSA, IO, amendment, or terms of service? Or is it something else (invoice, report, letter, resume)? Return JSON: `{type, confidence, counterparty_name, document_type}`." If `type === 'contract'` and `confidence > 0.85`, route directly. |
| **Destination** | Vet pill → `createAnalysisEngine(role).analyze(text)` (existing D104 pipeline). |
| **Drafted action** | Full RED/GREEN analysis, expandable issue list, analysis_id persisted. If Part 2 ships: redline link button. Counterparty name extracted by classifier surfaces in header: "I see a contract from [Granite Media]. Running the full analysis." |
| **Fallback** | If classifier says "not a contract" (e.g., invoice, report): fall through to text-based Vet with extracted text appended as context. Milo says: "This doesn't look like a contract — it's a [type]. Here's what I see:" and treats it as a text-based Vet query. |
| **Effort** | **S** — mostly wired already. Classifier prompt is new; everything else is D104 infrastructure. |

### Route B: LinkedIn Profile Screenshot (PNG/JPG)

| Column | Value |
|---|---|
| **Detection rule** | MIME `image/png` or `image/jpeg`. Size limit: 10MB (images are larger than documents). Client-side: detect image MIME, read as base64 via `readAsDataURL`. |
| **Classification step** | `callClaudeVision` with classifier system prompt (see §3). Vision model extracts: `{type: 'linkedin_profile', name, title, company, location, profile_url, confidence}`. Detection signals: LinkedIn UI patterns (blue nav bar, profile photo placement, "Experience" / "About" sections, linkedin.com URL visible in browser chrome). |
| **Destination** | **Pill selection (LOCKED — Decision 112, Q-HIGH-2):** Milo infers from extracted job title + company context. Title keyword mapping: buyer/advertiser/demand/brand/agency → Buyer pill. publisher/affiliate/media-buying/supply/traffic-broker/network → Publisher pill. Ambiguous or no clear signal → Sales pill (default). Inference shown to operator: "Looks like a Sales lead based on title" with one-click override to Publisher or Buyer. Override visible on ALL LinkedIn routes regardless of confidence. Inference logic lives in the classifier prompt, not new code. |
| **Drafted action** | 1. `@milo/crm.createLead({ company, contact_name: name, linkedin_url: profile_url, role: title, source: 'research', metadata: { surface18_source: 'linkedin-drop' } })` — creates CRM lead with extracted data. 2. `@milo/blacklist.check(client, { company_name: company, contact_names: [name], linkedin_url: profile_url })` — screens entity. 3. If blacklist hit: surface in chat with warning. If clean: draft outreach message in the destination pill's voice. Chat response: "I pulled [Name], [Title] at [Company] from that screenshot. Clean on the blacklist. Here's an outreach draft:" |
| **Fallback** | If vision can't extract name + company (confidence < 0.5): "I see a screenshot but can't make out the details. Who is this and what's their role?" If vision identifies a non-LinkedIn screenshot: fall through to Route C (scam chat) or Route F (general). |
| **Effort** | **M** — new image upload path on client, new vision classifier prompt, new CRM/blacklist wiring. No new primitives. |

### Route C: Scam-Chat Screenshot (PNG/JPG)

| Column | Value |
|---|---|
| **Detection rule** | MIME `image/png` or `image/jpeg`. Same upload path as Route B. Classifier distinguishes from LinkedIn based on visual content (group chat UI vs profile page). |
| **Classification step** | `callClaudeVision` with classifier system prompt. Detection signals: multiple speaker bubbles, group chat UI (Teams/Telegram/WhatsApp chrome), scam-alert keywords, entity names being discussed, dollar amounts, accusation language. Extracts: `{type: 'scam_chat', entities: [{name, sentiment: 'negative'\|'positive'\|'neutral', evidence}], source_platform, confidence}`. |
| **Destination** | Vet pill for each extracted entity. Multiple entities trigger sequential blacklist checks. |
| **Drafted action** | For each named entity: 1. `@milo/blacklist.check(client, { company_name: entity.name })`. 2. If already blacklisted: "Already on the blacklist. Flagged on [date] for [reason]." 3. If NOT blacklisted + negative sentiment: "Not in the blacklist yet. Based on this screenshot: [evidence]. Add to blacklist?" — with action button affordance. 4. If positive sentiment: "Clean signal for [entity]. Want me to draft an outreach?" Chat response surfaces all entities found with per-entity disposition. |
| **Fallback** | If vision can't extract entity names: show OCR text dump and ask: "I see a group chat screenshot but can't pull specific names. Which entity are we deciding on?" |
| **Effort** | **M** — same image upload infrastructure as Route B (shared). Vision prompt is more complex (multi-entity extraction). Blacklist wiring exists. Action button affordance is new client-side work. |

### Route D: Audio File (MP3/WAV/M4A) — DEFERRED TO V2

| Column | Value |
|---|---|
| **Detection rule** | MIME `audio/mpeg`, `audio/wav`, `audio/x-m4a`, `audio/mp4`. |
| **Classification step** | N/A for v1. |
| **Destination** | N/A for v1. |
| **Drafted action** | N/A for v1. |
| **Fallback** | Milo responds: "I can't process audio files yet. If it's a call recording, use the call audit tool in /operator. If it's something else, try pasting a transcript and I'll work with that." |
| **Effort** | **L** for v2 — requires transcription infrastructure. No existing `@milo/*` package has transcription. ConvoQC pipeline is in a separate system (not extracted). Would need either a new Whisper integration or a Supabase Edge Function. Blocked until transcription primitive exists. |

### Route E: Spreadsheet (CSV/XLSX) — DEFERRED TO V2

| Column | Value |
|---|---|
| **Detection rule** | MIME `text/csv`, `application/vnd.openxmlformats-officedocument.spreadsheetml.sheet`. |
| **Classification step** | N/A for v1. |
| **Destination** | N/A for v1. |
| **Drafted action** | N/A for v1. |
| **Fallback** | Milo responds: "I see a spreadsheet. What do you want me to do with it? Give me context — billing data, publisher list, call logs — and I'll figure out where it goes." |
| **Effort** | **L** for v2 — spreadsheet parsing (SheetJS or similar), column detection, content classification. Too varied for v1's narrow scope. |

### Route F: Everything Else (TXT, ZIP, unknown PDF, etc.)

| Column | Value |
|---|---|
| **Detection rule** | Any MIME type not matched by Routes A-E. |
| **Classification step** | None. |
| **Destination** | Operator-driven. |
| **Drafted action** | None. |
| **Fallback** | Milo responds: "Not sure what to do with this [filename]. Tell me what it is — a contract, a screenshot, a data file — and I'll route it." Operator response triggers re-classification via the text classifier (existing auto-classify), with the file content attached as context. |
| **Effort** | **S** — just error copy and operator-driven fallback. |

---

## 3. Classifier Architecture

### 3a. Where the classifier lives

Backend: `/api/ask/route.ts`. The classifier runs AFTER rate limiting and BEFORE pill prompt execution. It replaces the current hard-coded `pill === 'vet' && contractFile?.data` gate with a general-purpose classification step.

```
Request arrives at /api/ask with fileUpload: { name, data, mime }
  ↓
Rate limit check (existing)
  ↓
File attached?
  ├── NO → existing text classifier (unchanged)
  └── YES → Surface 18 classifier
        ↓
      Detection rules (MIME + extension)
        ├── PDF/DOCX → extractText → text classifier prompt → Route A or fallback
        ├── Image (PNG/JPG) → callClaudeVision → vision classifier prompt → Route B or C
        ├── Audio → Route D fallback (v2 defer)
        ├── Spreadsheet → Route E fallback (v2 defer)
        └── Other → Route F fallback
        ↓
      Classification result: { destination, subtype, confidence, extracted_entities,
                               inferred_pill_switch, suggested_prompt }
        ↓
      Cross-pill check (LOCKED — Decision 112, Q-HIGH-1):
        ├── destination matches current pill → proceed, no confirmation needed
        └── destination differs from current pill → emit SSE confirm_switch event
              "I see a LinkedIn screenshot. Switch to Sales?" [Yes / Stay on Vet]
              Operator confirms or stays → proceed with chosen pill
        ↓
      Auto-submit threshold (LOCKED — Decision 112, Q-HIGH-3):
        ├── confidence ≥ ASK_CLASSIFIER_AUTO_SUBMIT_THRESHOLD (default 0.85)
        │     → auto-submit: classify → analysis runs → response renders. Zero typing.
        └── confidence < threshold
              → pre-fill input with suggested_prompt, operator confirms send
              "I see a screenshot but I'm not sure what it is.
               Want me to [vet the entity / draft outreach / something else]?"
        ↓
      Operator override? (if provided in follow-up message)
        ├── YES → re-route to operator-specified pill
        └── NO → proceed with classified destination
        ↓
      Execute destination pill with file context injected
```

### 3b. Classifier system prompts

**Document classifier prompt (Route A — text-based, after extraction):**

```
You are a document classifier for a pay-per-call performance marketing network.

Given the extracted text from a document, classify it.

Return ONLY valid JSON:
{
  "type": "contract" | "invoice" | "report" | "letter" | "resume" | "io" | "amendment" | "other",
  "confidence": 0.0 to 1.0,
  "counterparty_name": "extracted company name or null",
  "document_type": "MSA" | "IO" | "Amendment" | "Vendor Agreement" | "QC Guidelines" | "Other",
  "role_orientation": "publisher" | "buyer" | "unknown"
}

Signals for "contract": section headers like "TERMS AND CONDITIONS", "MASTER SERVICE AGREEMENT", "INSERTION ORDER", "WHEREAS", "IN WITNESS WHEREOF", signature blocks, party definitions, governing law clauses, indemnification sections.

Signals for "io": campaign-specific fields (vertical, payout, duration, states, HOO, cap, DID, ping URL).

If unsure, return type "other" with low confidence.
```

**Image classifier prompt (Routes B + C — vision-based):**

```
You are an image classifier for a pay-per-call performance marketing network.

Analyze this screenshot and classify it.

Return ONLY valid JSON:
{
  "type": "linkedin_profile" | "scam_chat" | "industry_chat" | "website" | "document_scan" | "other",
  "confidence": 0.0 to 1.0,
  "entities": [
    {
      "name": "extracted name",
      "company": "extracted company or null",
      "title": "extracted title or null",
      "role_guess": "buyer" | "publisher" | "unknown",
      "sentiment": "positive" | "negative" | "neutral",
      "evidence": "brief quote or signal from screenshot"
    }
  ],
  "profile_url": "linkedin.com URL if visible, else null",
  "source_platform": "linkedin" | "teams" | "telegram" | "whatsapp" | "slack" | "unknown"
}

For linkedin_profile: look for LinkedIn UI elements — blue nav bar, profile photo, "Experience" section, connection count, headline. Extract name, title, company, location, URL.

For scam_chat / industry_chat: look for multiple speaker bubbles, group chat name, message timestamps. Extract entity names being discussed. Determine sentiment per entity: "negative" if accusations, scam alerts, nonpayment claims. "positive" if vouches, recommendations.

If unsure what the image shows, return type "other" with low confidence.
```

### 3c. Classifier output shape (TypeScript type)

```typescript
interface ClassifierResult {
  destination: 'vet-contract' | 'vet-chat' | 'sales-linkedin' | 'publisher-linkedin' | 'buyer-linkedin' | 'vet-scam-chat' | 'defer-audio' | 'defer-spreadsheet' | 'ask-operator';
  subtype: string;
  confidence: number;
  extracted_entities: Array<{
    name: string;
    company: string | null;
    title: string | null;
    role_guess: 'buyer' | 'publisher' | 'unknown';
    sentiment: 'positive' | 'negative' | 'neutral';
    evidence: string | null;
  }>;
  file_meta: {
    mime: string;
    size: number;
    name: string;
  };
  // LOCKED — Decision 112, Q-HIGH-1: cross-pill confirmation
  inferred_pill_switch: {
    from: string;       // pill operator was on when file was dropped
    to: string;         // pill classifier wants to route to
    differs: boolean;   // true when from !== to → triggers confirmation
  };
  // LOCKED — Decision 112, Q-HIGH-3: low-confidence pre-fill
  suggested_prompt: string | null;  // pre-filled input text when confidence < threshold
                                    // null when confidence >= threshold (auto-submit)
}
```

### 3d. Classifier tier selection

The classifier calls use `@milo/ai-client` tiers:
- **Document classifier (Route A):** `tier: 'quick'` (Haiku) — text classification is fast and cheap. Full analysis engine runs separately at `tier: 'judgment'` (Opus).
- **Image classifier (Routes B + C):** `tier: 'standard'` (Sonnet) — vision classification needs enough capability to extract names and UI patterns. Haiku's vision is less reliable for OCR. Opus is overkill for classification.

### 3e. Classifier latency budget

Target: classifier decision in < 3 seconds total.
- Document extraction (pdf-parse/mammoth): ~500ms
- Document classification (Haiku, ~500 tokens): ~800ms
- Image classification (Sonnet vision, ~1000 tokens): ~2000ms

The operator sees a "Classifying..." banner during this window. The existing cinematic upload state from D181 handles the UX during extraction + classification.

---

## 4. Operator Experience Flow

### 4a. Drop → Classify → Confirm → Route → Deliver

*Updated with 3 LOCKED decisions from Decision 112.*

```
1. OPERATOR drops file on /ask chat surface (may be on any pill)
     ↓
2. Client: MIME detection + size check
   - Image (PNG/JPG/WEBP): readAsDataURL → base64, max 10MB
   - Document (PDF/DOCX): readAsArrayBuffer → base64, max 5MB (existing)
   - Audio: reject with "not yet" message
   - Spreadsheet: reject with "tell me what to do" message
   - Other: reject with "tell me what this is" message
     ↓
3. Client: sends to /api/ask with `fileUpload` field
   (LOCKED — Decision 112, Q-HIGH-1: replaces legacy `contractFile` field):
   { pill: currentPill, message: '[optional operator context]',
     fileUpload: { name, data: base64, mime } }
   Backward compat: if `contractFile` sent by stale client cache,
   treat as `fileUpload` with `mime: 'application/pdf'`.
     ↓
4. Backend: classifier runs
   - SSE event: { type: 'classifying', text: 'Classifying this...' }
     ↓
5. Cross-pill confirmation (LOCKED — Decision 112, Q-HIGH-1):
   - Classifier returns destination. Compare to `pill` in request (current pill).
   - SAME pill (destination matches current):
     → proceed without confirmation
   - DIFFERENT pill (destination differs from current):
     → SSE event: { type: 'confirm_switch', from: currentPill, to: destination }
     → Chat prompt: "I see a LinkedIn screenshot. Switch to Sales?"
       [Yes, switch to Sales] [Stay on Vet]
     → Wait for operator choice before proceeding.
     → ~1s friction on cross-pill routes. Prevents misroutes.
     ↓
6. Auto-submit vs confirm (LOCKED — Decision 112, Q-HIGH-3):
   Threshold: ASK_CLASSIFIER_AUTO_SUBMIT_THRESHOLD env var (default 0.85)
   - HIGH confidence (≥ threshold):
     AUTO-SUBMIT. Drop → classify → analysis runs → response renders.
     Zero typing required. Same UX as current Vet contract drops.
     SSE event: { type: 'classified', destination, entities, confidence }
     Chat banner: "I see a [contract from Granite Media / LinkedIn profile
       for Randy at Granite Media]. Running [analysis / outreach draft]..."
     [Override link visible: "Wrong? Route to ___"]
   - LOW confidence (< threshold):
     PRE-FILL + CONFIRM. Input field pre-filled with suggested prompt.
     SSE event: { type: 'classify_confirm', destination, entities,
       confidence, suggested_prompt }
     Chat prompt: "I see a screenshot but I'm not sure what it is.
       Want me to [vet the entity / draft outreach / something else]?"
     Operator picks action or types own intent + hits send.
     ↓
7. LinkedIn pill inference (LOCKED — Decision 112, Q-HIGH-2):
   For Route B only. Classifier extracts job title from LinkedIn screenshot.
   - buyer/advertiser/demand/brand/agency → Buyer pill
   - publisher/affiliate/media-buying/supply/traffic-broker → Publisher pill
   - ambiguous / no clear signal → Sales pill (default)
   Inference shown: "Looks like a Sales lead based on title"
   One-click override to Publisher or Buyer always visible.
     ↓
8. Backend: fires destination pipeline
   - Route A: contract analysis engine → Vet pill prompt with analysis context
   - Route B: CRM lead creation + blacklist check → destination pill prompt
     with entity context + title-inferred pill voice
   - Route C: per-entity blacklist check → Vet pill prompt with evidence context
     ↓
9. Client: renders response in destination pill's voice and format
   - Contract: expandable issue list with SeverityBadge (existing components)
   - LinkedIn: outreach draft with CRM lead creation confirmation
   - Scam chat: per-entity disposition with action affordances
```

### 4b. Override and confirmation mechanisms

Three interaction points exist, in order of appearance (LOCKED — Decision 112):

1. **Cross-pill confirmation (Q-HIGH-1):** When classifier destination differs from the pill the operator was on. Inline prompt: "I see a LinkedIn screenshot. Switch to Sales?" with [Yes, switch] / [Stay on {currentPill}] buttons. This fires BEFORE auto-submit logic. If operator stays, the file is processed in the current pill with file context injected.

2. **Low-confidence confirmation (Q-HIGH-3):** When confidence < `ASK_CLASSIFIER_AUTO_SUBMIT_THRESHOLD` (default 0.85). Input field pre-filled with suggested prompt (e.g., "Vet this entity" / "Draft outreach"). Operator edits or confirms. This is NOT a modal — it's the regular input field with pre-filled text. Operator hits send or types over it.

3. **Post-response override (unchanged):** After the classified banner on high-confidence responses, a small text link: "Wrong? Route to [Vet | Sales | Publisher | Buyer]" with clickable pill options. Clicking one re-routes without re-uploading. Backend re-runs destination pipeline with the new pill.

- **LinkedIn pill inference display (Q-HIGH-2):** On all LinkedIn routes, regardless of confidence level, the inference rationale is visible: "Looks like a Sales lead based on title." Override to Publisher or Buyer is always one click. This is not gated by confidence.

- **Override logging:** If the operator overrides at ANY of these three points, `operator_override: true` and `final_destination` are set on the chat_logs row. The original `classifier_decision` (including `inferred_pill_switch`) is preserved in `context_metadata` for training data.

### 4c. Mobile experience

File drops from phone:
- iOS/Android: tap the attachment icon (existing, needs extension) to open file picker
- Selected file → same base64 encoding path
- Classifier banner and override buttons must be touch-friendly (minimum 44px tap targets)
- Response rendering: same as desktop (existing components are already responsive from D181)

---

## 5. Instrumentation

### 5a. Per-classification data capture

Every Surface 18 file drop produces a chat_logs row with:

| Field | Storage location | Shape |
|---|---|---|
| File metadata | `attachments` JSONB (existing column) | `{ surface18: true, mime: 'image/png', size: 245000, name: 'randy-profile.png' }` |
| Classifier decision | `context_metadata` JSONB (existing column, extended) | `{ classifier: { destination: 'sales-linkedin', subtype: 'linkedin_profile', confidence: 0.92, extracted_entities: [...], model: 'claude-sonnet-4-6', latency_ms: 1840 } }` |
| Operator override | `operator_override` BOOLEAN (new column) | `true` / `false` |
| Final destination | `final_destination` TEXT (new column) | `'sales'` / `'vet'` / etc. |

**Schema migration (2 new columns on existing table):**
```sql
ALTER TABLE chat_logs ADD COLUMN IF NOT EXISTS operator_override BOOLEAN DEFAULT FALSE;
ALTER TABLE chat_logs ADD COLUMN IF NOT EXISTS final_destination TEXT;
```

### 5b. Training data generation

Over time, the classifier improves by analyzing:
- **Override rate:** `SELECT COUNT(*) FILTER (WHERE operator_override) / COUNT(*) FROM chat_logs WHERE (context_metadata->>'classifier') IS NOT NULL` — target < 25%
- **Confidence distribution:** histogram of classifier confidence scores to identify gray zones
- **Entity extraction accuracy:** compare extracted names to operator-entered names in CRM leads
- **Destination accuracy:** compare `context_metadata->'classifier'->>'destination'` to `final_destination` where overridden

### 5c. Thumbs rating extension

The existing thumbs UI (D192) extends for Surface 18 responses:

- Thumbs-down on a Surface 18 response triggers a secondary prompt: "Was the routing wrong, or just the response?"
- "Routing wrong" → `@milo/feedback.recordFeedback(client, { context_type: 'file_routing', signal: 'negative', detail: { classifier_decision, final_destination, operator_correction } })` — feeds back into classifier prompt tuning
- "Response wrong" → existing thumbs-down flow (admin_flag, no classifier feedback)

---

## 6. V1 Scope Decision

### Ships in v1

| Route | File type | Status | Reason |
|---|---|---|---|
| A | Contract PDF/DOCX | **SHIP** | 90% wired from D104. Classifier prompt is the only new piece. |
| B | LinkedIn screenshot | **SHIP** | Core daily workflow for TLP sales team. High-value, medium effort. |
| C | Scam-chat screenshot | **SHIP (behind feature flag)** | Complex vision extraction, multi-entity handling. Ship gated, expand if accuracy > 85%. |
| D | Audio (MP3/WAV/M4A) | **DEFER** | No transcription primitive exists. Would require new infrastructure. |
| E | Spreadsheet (CSV/XLSX) | **DEFER** | Too varied for narrow v1 scope. |
| F | Everything else | **SHIP** (as graceful fallback) | Operator-driven routing with helpful prompt. |

### v1 go/no-go criteria

**Ship v1 when:**
- Route A (Contract) correctly classifies 90%+ of test PDFs/DOCX as contracts (already proven in D104)
- Route B (LinkedIn) correctly extracts name + company from 85%+ of LinkedIn screenshots
- Route C (Scam chat) correctly extracts at least 1 entity name from 80%+ of scam chat screenshots
- Override rate across all routes < 25% on first 50 real file drops

**Hold v1 if:**
- Classifier accuracy < 85% on Route B (LinkedIn is the highest-traffic route)
- Override rate > 25% (operators don't trust the routing)
- Vision API costs exceed $0.50/classification average (Sonnet vision at scale)

---

## 7. Component Reuse Map

### Existing components reused as-is

| Component | Current location | Surface 18 use |
|---|---|---|
| Cinematic upload state | `ask/page.tsx` (D181) | "Classifying..." banner during classifier execution |
| ContractAnalysisCard | `ask/page.tsx` (D104) | Route A contract display |
| SeverityBadge | `src/components/contract/SeverityBadge.tsx` | Route A issue severity display |
| IssueCard expandable | `ask/page.tsx` (D104) | Route A per-issue display |
| Pill selector buttons | `ask/page.tsx` | Override pill selection UI |
| Thumbs rating | `ask/page.tsx` (D192) | Extended with routing feedback prompt |
| SSE event handler | `ask/page.tsx` | Extended with new event types |
| auto-classifier | `api/ask/route.ts` | Pattern extended for file classification |
| blacklist pre-check | `api/ask/route.ts` | Route B + C entity screening |
| CRM context injection | `api/ask/route.ts` (D102) | Route B entity data injection |

### Existing components reused with extension

| Component | Extension needed | Effort |
|---|---|---|
| File drop handler (`handleFileContent`) | Add image MIME detection, base64 encoding via `readAsDataURL`, 10MB size limit for images | S |
| Request body shape | Replace `contractFile` with `fileUpload: { name, data, mime }` (LOCKED — Decision 112, Q-HIGH-1). Backward compat shim for stale `contractFile` payloads. | S |
| SSE event types | Add `classifying`, `classified`, `classify_confirm`, `confirm_switch` events (LOCKED — Decision 112, Q-HIGH-1 + Q-HIGH-3) | S |
| `context_metadata` JSONB shape | Add `classifier` key with decision data | S |
| `attachments` JSONB shape | Add `surface18` flag and file metadata | S |

### New components (6 total, all client-side UI)

| Component | Purpose | Effort |
|---|---|---|
| ClassifiedBanner | Displays "I see a [type] for [entity]" with override link | S |
| ConfirmClassification | Low-confidence prompt with Yes/No + pill selector buttons | S |
| LinkedInLeadCard | Displays extracted LinkedIn data + CRM lead creation result + blacklist status | M |
| ScamChatEntityList | Per-entity disposition list with action affordances (Blacklist / Outreach / Ignore) | M |
| FileTypeRejectMessage | "I can't process [audio/spreadsheet] yet" styled message | S |
| RoutingFeedbackPrompt | "Was the routing wrong, or just the response?" secondary prompt on thumbs-down | S |

---

## 8. Implementation Effort Estimates

| Item | Effort | Notes |
|---|---|---|
| **Client: image upload path** | S | Add `readAsDataURL` for images, MIME detection, 10MB limit, `fileUpload` request field |
| **Backend: classifier dispatcher** | M | Replace hard-coded `pill === 'vet' && contractFile` with general routing. Detection rules → classifier call → SSE events → destination pipeline |
| **Backend: document classifier prompt** | S | Small Haiku prompt, parse JSON response, map to destination |
| **Backend: image classifier prompt** | M | Sonnet vision prompt, entity extraction, LinkedIn vs scam-chat disambiguation. More complex than document classifier. |
| **Backend: Route B wiring (LinkedIn)** | M | Vision → CRM lead creation → blacklist check → pill injection. Three primitive calls chained. |
| **Backend: Route C wiring (scam chat)** | M | Vision → multi-entity extraction → per-entity blacklist check → disposition list. More complex than Route B. |
| **Client: ClassifiedBanner + ConfirmClassification** | S | Two small components, SSE event handlers |
| **Client: LinkedInLeadCard** | M | Displays extracted data, CRM result, blacklist status. New layout. |
| **Client: ScamChatEntityList** | M | Per-entity cards with action buttons. New layout. |
| **Client: override mechanism** | S | Re-sends to /api/ask with overridden pill, preserves file data |
| **Schema migration** | S | 2 columns on chat_logs |
| **Instrumentation + feedback** | S | Extend existing thumbs with routing feedback |
| **Route D/E/F fallback messages** | S | Static copy, no logic |
| **Total** | **~8-10 dev-hours** | Majority is Routes B + C. Route A is nearly free. |

---

## 9. Open Questions for Mark

*3 HIGH questions resolved in Decision 112. 6 MEDIUM + 2 LOW remain OPEN for in-build resolution.*

### LOCKED — resolved by Mark (Decision 112)

~~4. **For LinkedIn screenshot routing to Sales vs Publisher — Milo decides automatically (based on title/company) OR always asks?**~~
**LOCKED (Q-HIGH-2, Decision 112):** Option B — Milo infers from extracted job title + company context. "Director of Marketing at insurance company" → Buyer. "ACA call publisher" / "PPC traffic broker" → Publisher. Ambiguous → Sales (default). Inference shown with one-click override always visible. Logic lives in classifier prompt, not new code. Post-v1: once accuracy >95%, can reduce override visibility.

~~6. **Should Route B (LinkedIn) auto-submit like Route A (contracts)?**~~
**LOCKED (Q-HIGH-3, Decision 112):** Hybrid — confidence-based. HIGH confidence (≥ 0.85): auto-submit, zero typing. LOW confidence (< 0.85): pre-fill input with suggested prompt, operator confirms. Threshold tunable via `ASK_CLASSIFIER_AUTO_SUBMIT_THRESHOLD` env var. Applies to ALL routes, not just LinkedIn.

~~8. **Should the `fileUpload` field replace `contractFile` or coexist?**~~
**LOCKED (Q-HIGH-1, Decision 112):** Option B — `fileUpload` replaces `contractFile`. Classifier runs first; if destination differs from current pill, surface inline confirmation before sending ("I see a LinkedIn screenshot. Switch to Sales?" [Yes / Stay on Vet]). If same pill, proceed without confirmation. Backward compat: stale `contractFile` payloads treated as `fileUpload` with `mime: 'application/pdf'`. Post-v1: once accuracy >95% via harness eval, can shift to silent auto-switch.

### OPEN — 6 MEDIUM (resolve during build)

1. **Should the classifier banner show its confidence score visibly or hide it?**
   Recommendation: HIDE for operators, LOG for admin. Operators don't need to see "0.87 confidence" — they need to see "I see a contract from Granite Media." The /review page can show confidence in the batch results view.

2. **Where does the operator override live — inline button in chat or dropdown?**
   Recommendation: INLINE in chat. A small "[Wrong? Route to Vet | Sales | Publisher | Buyer]" text link beneath the classified banner. Not a dropdown — dropdown feels like a form element, not a chat interaction.

3. **Should Surface 18 work in /operator surfaces too, or /ask only for v1?**
   Recommendation: /ask only for v1. /operator has its own file handling (contract-review page, entity detail panel). Unifying later is possible but adds scope now.

5. **Scam-chat screenshot routing — auto-blacklist if confidence > 0.95, or always require operator confirm?**
   Recommendation: ALWAYS CONFIRM for blacklist actions. Blacklisting is a high-consequence action (entity is blocked from doing business). Even at 0.99 confidence, a screenshot OCR could misread a name. "Add to blacklist?" with one-click confirm is fast enough.

7. **File size limit for images: 10MB or lower?**
   Phone screenshots are typically 1-5MB. 10MB allows high-res captures but increases base64 encoding time and API costs. Recommendation: 5MB to match contract limit, revisit if operators report size issues.

11. **Should the image classifier run at `standard` (Sonnet) or `quick` (Haiku)?**
    Sonnet vision is more reliable for OCR and UI pattern recognition. Haiku vision is faster and cheaper but may miss entity names in low-res screenshots. Recommendation: START with Sonnet, benchmark accuracy, downgrade to Haiku if accuracy holds above 85%.

### OPEN — 2 LOW (post-v1 polish)

9. **CRM lead source value for LinkedIn drops?**
   `LeadSource` enum is `'manual' | 'scrape' | 'referral' | 'inbound' | 'research'`. No `'linkedin-drop'` value. Options: (a) use `'research'` with metadata tag, (b) add `'linkedin-drop'` to enum. Recommendation: (a) use `'research'` with `metadata: { surface18_source: 'linkedin-drop' }`. Zero schema changes. Coder-1 can add the enum value later if filtering by source becomes important.

10. **Multi-file drops?**
    What if operator drops 3 files at once? Current handler takes only `e.dataTransfer?.files[0]`. Recommendation: v1 stays single-file only. Multi-file would need sequential classification + merged response. Defer to v2.

---

## 10. Data Flow Diagrams

### Route A: Contract PDF

```
File drop → MIME check → extractText(buffer, name)
  → callClaude(tier='quick', classifierPrompt, extractedText)
  → { type: 'contract', confidence: 0.93, counterparty: 'Granite Media' }
  → createAnalysisEngine('buyer').analyze(text)
  → SSE: { type: 'analysis', data: AnalysisResult }
  → Vet pill prompt with [CONTRACT ANALYSIS] context injected
  → SSE: { type: 'delta', text: '...' }
  → chat_logs row: pill='vet', analysis_id=X, attachments={surface18:true, ...}
```

### Route B: LinkedIn Screenshot

```
File drop → MIME check → readAsDataURL → base64
  → callClaudeVision(tier='standard', imageClassifierPrompt, base64)
  → { type: 'linkedin_profile', confidence: 0.91, entities: [{name: 'Randy', company: 'Granite Media', title: 'VP Sales'}] }
  → Parallel:
      (1) createLead({ company: 'Granite Media', contact_name: 'Randy', role: 'VP Sales', source: 'research' })
      (2) blacklist.check({ company_name: 'Granite Media', contact_names: ['Randy'] })
  → SSE: { type: 'classified', destination: 'buyer-linkedin', entities: [...] }
  → Buyer pill prompt with CRM + blacklist context injected
  → SSE: { type: 'delta', text: '...' }
  → chat_logs row: pill='buyer', context_metadata={classifier:{...}, crm:{lead_id:...}, blacklist:{...}}
```

### Route C: Scam-Chat Screenshot

```
File drop → MIME check → readAsDataURL → base64
  → callClaudeVision(tier='standard', imageClassifierPrompt, base64)
  → { type: 'scam_chat', confidence: 0.88, entities: [
      {name: 'Sky Marketing', sentiment: 'negative', evidence: '"Runnnnnnn" — Nestor'},
      {name: 'Kevin De Vincenzi', sentiment: 'negative', evidence: '"owes us $2k"'}
    ]}
  → For each entity: blacklist.check({ company_name: entity.name })
  → SSE: { type: 'classified', destination: 'vet-scam-chat', entities: [...] }
  → Vet pill prompt with per-entity blacklist results + evidence injected
  → SSE: { type: 'delta', text: '...' }
  → Client: render ScamChatEntityList with action buttons
  → chat_logs row: pill='vet', context_metadata={classifier:{...}, blacklist_results:[...]}
```

---

## 11. Risk Assessment

| Risk | Impact | Mitigation |
|---|---|---|
| Vision classifier misreads entity names from screenshots | Wrong CRM lead created, wrong blacklist check | Low-confidence confirmation prompt at < 0.85. Operator override always available. Thumbs-down feedback captures misroutes. |
| LinkedIn profile screenshot is actually a recruiter's view of someone else's profile | Entity extracted is the recruiter, not the target | Classifier prompt should instruct: "Extract the name of the person WHOSE PROFILE is being viewed, not the logged-in user." LinkedIn's profile page layout makes this distinguishable. |
| Scam-chat screenshots contain PII (phone numbers, emails, addresses) | PII stored in chat_logs | Already handled by existing /ask architecture — chat_logs stores `input_text.slice(0, 5000)`. Vision classifier extracts entity names only, not PII. Base64 image data should NOT be persisted to chat_logs (only the classifier result). |
| Base64 encoding of 5-10MB images bloats request payloads | Slow uploads, API timeouts | Client-side image compression before encoding? Or resize to max 2000px width. Monitor request sizes in production. |
| Sonnet vision costs at scale | $0.50-1.00 per classification | Track per-classification cost in context_metadata. Benchmark Haiku accuracy and downgrade if viable. |
| Feature flag for Route C may never get un-flagged | Scam-chat routing stays hidden indefinitely | Set explicit criteria: un-flag when 50+ scam-chat drops have < 25% override rate and > 80% entity extraction accuracy. |

---

## 12. What This Does NOT Cover (Explicit Exclusions)

- **No new primitives.** Surface 18 v1 wires ONLY existing `@milo/*` packages.
- **No new database tables.** Two new columns on existing `chat_logs` table.
- **No live chat widget.** Milo speaks through the existing /ask chat surface.
- **No multi-file drops.** v1 is single file only.
- **No audio transcription.** Deferred to v2 pending transcription primitive.
- **No spreadsheet parsing.** Deferred to v2.
- **No /operator integration.** v1 is /ask only.
- **No auto-blacklisting.** All blacklist actions require operator confirmation.
- **No contract redline link generation.** That's Part 2 of Vet wire-up (separate directive, D104 spec).
- **No classifier prompt self-tuning.** Classifier prompt is static in v1. Improvement is manual via override rate analysis on /review.

---

Report at: (to be pushed)
