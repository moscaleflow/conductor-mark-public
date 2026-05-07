# Onboarding Code Audit -- milo-for-ppc

**Coder-3 Research Lane | 2026-05-07**
**Repo:** `~/Documents/GitHub/milo-for-ppc/`

---

## 1. Pages and Routes -- Complete Map

### 1A. `/onboard` -- Team Member Self-Onboarding (Chat Flow)

**File:** `src/app/onboard/page.tsx` (1,077 lines)
**Auth:** Bypassed in middleware (line 50: `request.nextUrl.pathname === '/onboard'`)
**Purpose:** New team members onboard themselves via a conversational chat with Milo.

**Flow (6 steps):**
```
intro -> name -> email -> role -> focus -> setup -> done
```

1. **intro:** Milo types "Hey. I'm Milo. I run operations here at The Lead Penguin." then asks for name.
2. **name:** User types name. Milo responds with a randomized acknowledgment.
3. **email:** User types email. Stored for account creation.
4. **role:** User picks from pill buttons: Billing, Publisher QA, Outreach, Prospecting, DID Ops, Admin. Admin is only shown if email matches `ADMIN_EMAILS` (mark@scaleflow.co, tiffani@scaleflow.co).
5. **focus:** User picks focus areas from role-specific options (8 per role). Supports custom focus. "Let's go" button appears after 1+ selection.
6. **setup:** Cinematic progress screen. Connects to TrackDrive, loads call data, configures dashboard, sets permissions, prepares workspace. Shows role-specific "MILO TIP" cards. Calls `POST /api/onboard/signup` to create user account.

**Submits to:** `POST /api/onboard/signup`
**After success:** Auto-signs in with temp password, redirects to `/`.

**Design:** Dark theme (#0a0a0f bg), chat bubble UI, typing indicator with bounce animation, fade transition to setup screen, MILO wordmark.

### 1B. `/onboard/buyer/[token]` -- Buyer Cinematic Onboarding (External)

**File:** `src/app/onboard/buyer/[token]/page.tsx` (593 lines)
**Auth:** Not in middleware bypass list for `/onboard/buyer/` -- but tokens bypass auth via the verify-token API pattern.
**Purpose:** External buyer fills out onboarding form. Token carries pre-filled data.

**Flow (6 phases):**
```
intro -> confirm -> campaigns -> budget -> contact -> done
```

1. **intro:** TLP logo with float animation + pink glow. "Welcome, {contact}." Cinematic staggered reveals (StoryLine components with delays).
2. **confirm:** Company name + contact name fields. Pre-filled from token.
3. **campaigns:** Vertical selection grid (14 options: ACA, SSDI, Final Expense, Medicare, Auto Insurance, Home Services, Debt Relief, Legal, Solar, Roofing, HVAC, Workers Comp, MVA, Tax Relief). Custom "+ Other" input.
4. **budget:** Min/Max bid range inputs + budget notes textarea. Optional.
5. **contact:** Email (required), phone (optional), preferred contact method pills (Email, Phone, Text/SMS). Validation with field errors.
6. **done:** "You're in, {name}." + 5-step next-steps timeline (review, IO, account setup, QA, calls flow).

**Token verification:** On mount, calls `POST /api/onboard/verify-token`. Handles: verifying, valid, invalid, expired states.
**Submits to:** `POST /api/onboard/buyer`
**Design:** `#06080f` bg, Outfit font, accent color `#e87fbf` (pink), ambient glow orbs, float + breathe animations. TLP logo used throughout.

### 1C. `/onboard/publisher/[token]` -- Publisher Cinematic Onboarding (External)

**File:** `src/app/onboard/publisher/[token]/page.tsx` (669 lines)
**Auth:** Bypassed in middleware (line 51: `request.nextUrl.pathname.startsWith('/onboard/publisher/')`)
**Purpose:** External publisher fills out onboarding form. Identical cinematic style to buyer.

**Flow (8 phases):**
```
intro -> confirm -> verticals -> calltypes -> creatives -> contact -> done
     \-> form (skip cinematic shortcut)
```

1. **intro:** Same cinematic as buyer but says "Hey, {contact}." Has "Just give me the form" ghost button for skip.
2. **confirm:** Shows known data from token (company, contact, verticals). "That's right" / "Close enough" buttons.
3. **verticals:** Same 14 vertical grid as buyer.
4. **calltypes:** 4 call type pills (True Inbound, Warm Transfer, Blind Transfer, Live Transfer). **True Inbound triggers a snarky verification gate:** "Hold on. True inbound? As in... people actually dial in on their own?" with confirm/deny.
5. **creatives:** "Got creatives ready?" with two big option cards: "Yes -- I'll upload them" / "Not yet".
6. **contact:** Name, company, email, phone (all required for publisher). Optional file upload for creatives.
7. **done:** "You're in, {name}." + 5-step timeline (MSA/IO, TrackDrive setup, DIDs, test call, go live). Creative upload reminder if hasCreatives=true but no file.
8. **form (skip):** Compact all-in-one form with name/company/email/phone + verticals + call types in a single view.

**Token verification:** Same as buyer -- calls `POST /api/onboard/verify-token`.
**Submits to:** `POST /api/onboard/publisher` (POST for data, PUT for file upload)
**Design:** Identical to buyer cinematic.

### 1D. `/welcome/[slug]` -- Team Member Cinematic Welcome (One-Time Claim)

**File:** `src/app/welcome/[slug]/page.tsx` (208 lines, RSC) + `src/app/welcome/[slug]/WelcomeExperience.tsx` (420 lines, client)
**Auth:** Bypassed in middleware (line 41: `request.nextUrl.pathname.startsWith('/welcome/')`)
**Purpose:** One-time-use welcome links for 11 specific team members. Slug = team member identifier.

**Flow:**
```
RSC validates slug -> WelcomeExperience renders ->
welcome_typing -> welcome_pause -> recognition_typing -> recognition_pause -> form -> submitting -> booting -> /operator
```

**RSC (page.tsx):**
- Validates slug against `TEAM_ROSTER` whitelist (11 members: cam, fab, jackie, jen, kate, kent, lee, malvin, tiffani, vee, mark).
- Checks `user_profiles.has_claimed_welcome` in DB. If claimed, shows "This link has been used" screen.
- If profile missing, shows "Your welcome link isn't ready yet" screen.
- Passes `slug`, `displayName`, `welcomeLine`, `recognitionLine` to client component.

**Client (WelcomeExperience.tsx):**
- Types welcome_line char-by-char (40ms/char). E.g., "Good morning Cam. I'm Milo."
- 600ms pause.
- Types recognition_line. E.g., "I've been watching the pipeline -- that outreach hustle is going to feel different with a real board."
- 600ms pause.
- Fades in email + password form.
- Password coaching in Milo's voice: "Pick something strong." -> "We can do better." -> "Almost there." -> "Now we're talking." -> "That's a fortress. Proceed."
- Submits to `POST /api/welcome/[slug]/claim`.
- On success, renders `BootingUpSequence` (types 3 lines: "Pulling in your latest activity...", "Ranking what matters today...", "Loading your knowledge..."), then redirects to `/operator`.

**Claim API:** `src/app/api/welcome/[slug]/claim/route.ts` (282 lines)
- Validates slug against whitelist.
- Password min 8 chars, blocks common passwords (password, 12345678, qwerty, etc.).
- Updates existing auth.users row with new email + password (users pre-created by Mark/Agent 2b).
- Updates `user_profiles`: sets `has_claimed_welcome=true`, `has_completed_onboarding=true`, `setup_complete=true`, `has_completed_operator_tour=false`, focus_areas from roster, role, is_admin.
- Race protection: `UPDATE ... WHERE has_claimed_welcome = false` -- only first request wins.
- Plants Supabase session cookies on the response.

### 1E. `/verify/[token]` -- Entity Verification Form (External)

**File:** `src/app/verify/[token]/page.tsx` (1,123 lines)
**Auth:** Bypassed in middleware (line 52: `request.nextUrl.pathname.startsWith('/verify/')`)
**Purpose:** Comprehensive compliance + vetting form sent to prospective publishers/buyers. Covers: role, location, verticals, traffic sources, call center vetting (onshore/offshore/nearshore), compliance (TCPA, CAN-SPAM, transfer scripts, opt-in records), contact info + attestation.

**Flow:**
```
loading -> intro -> role -> verticals -> traffic -> [call_center_vetting] -> compliance -> contact -> submitting -> done
```

**Design:** Same cinematic style as buyer/publisher onboard but with purple accent (#8b5cf6). TLP logo. Context-dependent compliance blocks (inbound, transfers, SMS/email). Extensive offshore vetting questions.

**Token:** Uses `verification_tokens` table (not HMAC tokens). Fetches via `GET /api/verify?token=...`. Submits via `PATCH /api/verify`.

### 1F. `/v/[code]` -- Short Link Redirector

**File:** `src/app/v/[code]/route.ts` (33 lines)
**Purpose:** Looks up `verification_tokens` table by code, redirects to `/onboard/buyer/[token]` or `/onboard/publisher/[token]` based on entity_type.

### 1G. `/setup` -- Minimal Profile Setup (Fallback)

**File:** `src/app/setup/page.tsx` (245 lines)
**Purpose:** Bare-minimum profile setup page. Name + role selection (7 options: Operations, Sales, Finance, Management, Admin, Publisher Relations, Buyer Relations). Only shown when `user_profiles.setup_complete = false`.
**Design:** Dark theme (#0a0a0f), centered card, role pills.

### 1H. `/tiffani` -- Billing Dashboard (NOT Onboarding)

**File:** `src/app/tiffani/page.tsx` (7 lines) -> renders `TiffaniView` component (430 lines)
**Purpose:** Tiffani's dedicated billing/payment card view. NOT onboarding-related. Shows ready-to-send invoices.

---

## 2. API Routes -- Complete Surface

### 2A. `/api/onboard/signup` (POST)
**File:** `src/app/api/onboard/signup/route.ts` (183 lines)
**Purpose:** Creates new user account during chat onboarding.
**Actions:**
1. Creates auth user with auto-generated temp password (`Milo-{uuid}!`).
2. Auto-confirms email (`email_confirm: true`).
3. Upserts `user_profiles` with role, focus_areas, `has_completed_onboarding=true`.
4. Creates `team_members` row.
5. Saves onboarding conversation to `milo_conversations`.
6. Re-onboarding support: if user exists with `has_completed_onboarding=false`, resets password and upserts profile.
7. Returns `{ success, userId, tempPassword }`.
**Security:** Never trusts client-supplied role for admin. Admin only via direct DB.

### 2B. `/api/onboard/buyer` (POST)
**File:** `src/app/api/onboard/buyer/route.ts` (239 lines)
**Purpose:** Handles buyer self-serve onboarding form submission.
**Actions:**
1. Creates/finds CRM contact via `findOrCreateCrmContact`.
2. Creates/finds `prospect_pipeline` entry (entity_type='buyer', stage='onboarding') via `findOrCreatePipelineEntry`.
3. Stores submission metadata (phone, contact method, verticals, bid range) in `risk_flags` column.
4. Creates high-priority `action_items` for operator review.
5. Starts buyer onboarding checklist run via `startBuyerOnboarding()` (non-blocking).
6. Sends confirmation email via Resend (HTML email with TLP branding, 5-step timeline).
**Returns:** `{ success, entity_id, onboarding_run_id }`

### 2C. `/api/onboard/publisher` (POST + PUT)
**File:** `src/app/api/onboard/publisher/route.ts` (219 lines)
**POST:** Handles publisher self-serve onboarding form submission.
**Actions:**
1. Calls `handleInboundPublisherSubmission()` which runs blacklist screening + pipeline creation.
2. Stores phone + creative status in `risk_flags`.
3. Creates high-priority `action_items`.
4. Sends confirmation email via Resend.
5. Starts publisher onboarding run via `startPublisherOnboarding()`.
**PUT:** Creative file upload. Uploads to `onboarding-creatives` storage bucket. Creates action item for TCPA review.

### 2D. `/api/onboard/verify-token` (POST)
**File:** `src/app/api/onboard/verify-token/route.ts` (48 lines)
**Purpose:** Server-side HMAC token verification. Called by buyer/publisher pages on mount.
**Returns:** `{ valid, payload?, error?, legacy? }`

### 2E. `/api/onboard/doc-status` (GET)
**File:** `src/app/api/onboard/doc-status/route.ts` (58 lines)
**Purpose:** Fetches document signing status for an entity. Tries Milo Engine first, falls back to local `contract_documents` table.
**Query:** `?entity_name=X`
**Returns:** `{ entity_name, documents: [{type, status, signed, sent_at, signing_url}] }`

### 2F. `/api/onboard/generate-docs` (POST)
**File:** `src/app/api/onboard/generate-docs/route.ts` (295 lines)
**Purpose:** Proxies document generation to Milo Engine. For IOs, pulls commercial terms from `agreed_terms`.
**Guards:** IO generation requires confirmed agreed_terms with non-empty qualifiers.
**Body:** `{ prospect_id, document_types: ['msa'|'io'|'w9'], signer_name?, signer_email? }`

### 2G. `/api/onboarding/runs` (GET)
**File:** `src/app/api/onboarding/runs/route.ts` (184 lines)
**Purpose:** Lists onboarding runs with full state (steps, progress, stall info).
**Query:** `?entity_name=X&status=active&limit=50`
**Returns:** `{ runs: RunSummary[], total }`

### 2H. `/api/onboarding/runs/[runId]/advance` (POST)
**File:** `src/app/api/onboarding/runs/[runId]/advance/route.ts` (93 lines)
**Purpose:** Marks a step as complete in an onboarding run.
**Body:** `{ step_key, note? }`

### 2I. `/api/onboarding/runs/[runId]/skip` (POST)
**File:** `src/app/api/onboarding/runs/[runId]/skip/route.ts` (94 lines)
**Purpose:** Skips a step with a required reason.
**Body:** `{ step_key, reason }`

### 2J. `/api/onboarding/progress-batch` (POST)
**File:** `src/app/api/onboarding/progress-batch/route.ts` (54 lines)
**Purpose:** Batch-fetches onboarding progress for multiple entity IDs (max 50).
**Body:** `{ entity_ids: string[] }`

### 2K. `/api/publisher-onboarding` (POST)
**File:** `src/app/api/publisher-onboarding/route.ts` (211 lines)
**Purpose:** Two modes via `mode` field:
- `add_to_pipeline`: Creates pipeline record only (stage='qualifying'). No onboarding run.
- `trigger_onboarding`: Full onboarding with scam check + pipeline + onboarding run. Used by EntityDetailPanel "Start Onboarding" button.

### 2L. `/api/webhooks/document-signed` (POST)
**File:** `src/app/api/webhooks/document-signed/route.ts` (169 lines)
**Purpose:** Receives document signing events, triggers onboarding auto-advancement.
**Auth:** Bearer token or X-Webhook-Secret or X-Internal-Secret (all check WEBHOOK_SECRET env var).
**Body:** `{ documentType, action, entityName, entityId? }`

### 2M. `/api/cron/onboarding-stall-check` (GET)
**File:** `src/app/api/cron/onboarding-stall-check/route.ts` (290 lines)
**Purpose:** Daily cron (6 AM Mountain). Finds steps active 72+ hours. Creates deduplicated action_items.
**Auth:** `Bearer ${CRON_SECRET}`

### 2N. `/api/welcome/[slug]/claim` (POST)
**File:** `src/app/api/welcome/[slug]/claim/route.ts` (282 lines)
**Purpose:** One-shot claim for `/welcome/[slug]` flow. See section 1D above.

---

## 3. The Onboarding Engine

### 3A. `@milo/onboarding` Package (extracted to milo-engine)

**Location:** `~/Documents/GitHub/milo-engine/packages/onboarding/`
**Installed at:** `node_modules/@milo/onboarding/`
**Schema doc:** `node_modules/@milo/onboarding/SCHEMA.md` (835 lines)

**Architecture:** Generic step-based DAG state machine. Multi-tenant. Flow definitions stored in DB. Runs are immutable snapshots of the flow at start time.

**Core API:**
```typescript
createOnboardingClient(config) -> OnboardingClient
  config: { supabase, tenantId, baseUrl, webhookSecret }

OnboardingClient:
  defineFlow(opts) -> Flow
  listFlows(opts?) -> Flow[]
  startRun(opts) -> Run
  listRuns(opts?) -> Run[]
  getRunState(runId) -> RunState
  advanceStep(runId, stepKey, result?) -> Step
  skipStep(runId, stepKey, reason?) -> Step
  failStep(runId, stepKey, reason) -> Step
  findStalledSteps(opts?) -> StalledStep[]
```

**Database Tables (3):**

```
onboarding_flows (12 cols)
  id, tenant_id, name, description, steps (JSONB), is_active,
  version, metadata, archived_at, created_at, updated_at
  UNIQUE(tenant_id, name)

onboarding_runs (16 cols)
  id, tenant_id, flow_id, flow_snapshot (JSONB), counterparty_id,
  status ('active'|'completed'|'cancelled'|'expired'),
  started_by, entry_path, webhook_url, expires_at,
  completed_at, cancelled_at, cancel_reason, metadata,
  created_at, updated_at

onboarding_steps (15 cols)
  id, tenant_id, run_id, step_key, step_order,
  status ('pending'|'active'|'completed'|'failed'|'skipped'|'blocked'),
  started_at, completed_at, failed_at, skipped_at,
  result (JSONB), failure_reason, skip_reason,
  attempt_count, metadata, created_at
  UNIQUE(run_id, step_key)
```

### 3B. `onboarding-client.ts` -- TLP Wrapper

**File:** `src/lib/onboarding-client.ts` (247 lines)
**Purpose:** Wraps `@milo/onboarding` with TLP-specific flow definitions.

**TLP Publisher Flow (12 steps):**
```
add_to_mop -> msa_sent -> msa_signed -> w9_sent -> w9_signed
                                      -> creatives_submitted -> creatives_approved
                                         -> did_provisioned -> config_validated
                                            -> io_sent -> io_signed -> go_live
```

Step definitions with timeouts:
| Step | Label | Required | Timeout | Depends On |
|------|-------|----------|---------|------------|
| add_to_mop | Add to Milo | yes | none | [] |
| msa_sent | MSA sent | yes | 7d | [add_to_mop] |
| msa_signed | MSA signed | yes | 14d | [msa_sent] |
| w9_sent | W-9 sent | no | 7d | [msa_signed] |
| w9_signed | W-9 signed | no | 14d | [w9_sent] |
| creatives_submitted | Creatives submitted | yes | 7d | [msa_signed] |
| creatives_approved | Creatives approved | yes | none | [creatives_submitted] |
| did_provisioned | DID provisioned | yes | none | [creatives_approved] |
| config_validated | Config validated | yes | none | [did_provisioned] |
| io_sent | IO sent | yes | 7d | [config_validated] |
| io_signed | IO signed | yes | 14d | [io_sent] |
| go_live | Go live | yes | none | [io_signed] |

**TLP Buyer Flow (8 steps):**
```
io_sent -> io_signed -> td_buyer_created -> campaigns_assigned
                                          -> bid_price_confirmed
                                          -> crm_counterparty_synced
                                             -> qa_complete -> ready_to_launch
```

| Step | Label | Required | Timeout | Depends On |
|------|-------|----------|---------|------------|
| io_sent | IO sent | yes | 3d | [] |
| io_signed | IO signed | yes | 7d | [io_sent] |
| td_buyer_created | Buyer created in call platform | yes | 1d | [io_signed] |
| campaigns_assigned | Campaigns assigned | yes | 1d | [td_buyer_created] |
| bid_price_confirmed | Bid price confirmed | yes | 2d | [td_buyer_created] |
| crm_counterparty_synced | CRM synced | yes | 1d | [td_buyer_created] |
| qa_complete | QA complete | yes | 2d | [campaigns_assigned, bid_price_confirmed, crm_counterparty_synced] |
| ready_to_launch | Ready to launch | yes | none | [qa_complete] |

**Tenant ID:** `'tlp'` (hardcoded)
**Client singleton:** Cached, uses MILO_ENGINE_SUPABASE_URL + SERVICE_KEY.
**Flow caching:** `ensureTlpPublisherFlow()` and `ensureTlpBuyerFlow()` check for existing flow by name, create if missing.

### 3C. `onboarding-automation.ts` -- Document Event Handling

**File:** `src/lib/onboarding-automation.ts` (371 lines)
**Purpose:** Receives document events (signed/sent) and auto-advances matching onboarding steps.

**Document-to-step mapping:**
| documentType | action | step_key |
|--------------|--------|----------|
| msa | signed | msa_signed |
| msa | sent | msa_sent |
| io | signed | io_signed |
| io | sent | io_sent |
| w9 | signed | w9_signed |
| w9 | sent | w9_sent |
| w8ben | signed | w8ben_signed |
| w8ben | sent | w8ben_sent |

**Run matching:** Searches by counterparty_id first, falls back to entity_name in metadata.
**Pipeline auto-advance:** When all required steps complete (100%), automatically moves prospect_pipeline from 'onboarding' to 'activation' stage.
**Audit trail:** Logs all events to `ai_action_log`.

### 3D. `publisher-onboarding.ts` -- Publisher Convergence Point

**File:** `src/lib/publisher-onboarding.ts` (396 lines)
**Purpose:** Single convergence point for all publisher onboarding paths.

**Three entry paths:**
```
Path A: Pipeline entity exists -> vetted -> triggerPublisherOnboarding(pipeline_entity_id)
Path B: Trusted shortcut -> no pipeline record yet -> creates one at 'onboarding' stage
Path C: Inbound submission -> auto-creates pipeline record at 'onboarding' stage
```

**triggerPublisherOnboarding() sequence:**
1. Blacklist screening via `screenEntity()`. Blocks if matched.
2. Pipeline record: find existing or create via `findOrCreatePipelineEntry()`.
3. Start onboarding run via `startPublisherOnboarding()`.
4. Log trigger event to `ai_action_log`.
5. Create action item for tracking.

**handleInboundPublisherSubmission():**
- For self-serve publisher submissions.
- Blacklist screening first.
- Find/create contact + pipeline entry.
- Auto-advances pipeline from early stages (outreach, qualifying, drip) to 'onboarding'.

---

## 4. The Cinematic UI

### 4A. Buyer Cinematic (6 phases)

Located in `src/app/onboard/buyer/[token]/page.tsx`.

**What makes it "cinematic":**
- **StoryLine component:** Elements fade up from 30px below with a 1.1s cubic-bezier easing, staggered by configurable delays (e.g., 400ms, 1200ms, 2200ms).
- **InteractiveSection component:** Same fade but includes a subtle scale (0.98 -> 1.0) over 1.2s.
- **Ambient glow orbs:** Two fixed-position radial gradients (pink top-right, blue bottom-left) with low opacity.
- **TLP logo:** Float animation (translateY -8px over 4s) + pink drop-shadow glow.
- **Breathe animation:** Drop-shadow pulsing from 20px to 40px blur.
- **Outfit font:** Imported from Google Fonts.
- **Scroll behavior:** Each StoryLine auto-scrolls into view on reveal.
- **Pink accent:** `#e87fbf` throughout.
- **Dark background:** `#06080f`.

### 4B. Publisher Cinematic (8 phases)

Located in `src/app/onboard/publisher/[token]/page.tsx`.

Identical visual system to buyer. Additional unique UI:
- **"Just give me the form" button:** Skip cinematic entirely, render all fields in a compact form.
- **True Inbound verification gate:** Snarky interstitial: "Hold on. True inbound? As in... people actually dial in on their own?" with "I swear, true inbound" / "Ok fine... it's transfers" buttons. If confirmed, a green italic message: "Alright, alright. We believe you. We're going to want to see those creatives though..."
- **Creative options:** Two large option cards instead of pills.

### 4C. Welcome Cinematic (for team members)

Located in `src/app/welcome/[slug]/WelcomeExperience.tsx`.

- **TypedLine component:** Characters typed one at a time at 40ms/char with a blinking cursor.
- **BootingUpSequence:** Full-screen black overlay. Three lines typed sequentially at 30ms/char: "Pulling in your latest activity...", "Ranking what matters today...", "Loading your knowledge..." Previous lines dim to gray. Fades out after all lines complete. 8-second safety net timer.
- **No TLP branding:** This is the Milo internal experience, not customer-facing.

### 4D. Verify Cinematic (external verification)

Located in `src/app/verify/[token]/page.tsx`.

Same StoryLine/InteractiveSection system but with purple accent (#8b5cf6). 10 phases including conditional call center vetting and context-dependent compliance blocks.

---

## 5. Token/Auth System

### 5A. HMAC-SHA256 Signed Tokens

**File:** `src/lib/onboarding-token.ts` (241 lines)

**Format:** `base64url(payload).base64url(hmac-sha256-signature)`

**Signing secret:** `ONBOARDING_TOKEN_SECRET` env var, falls back to SHA256 hash of `"onboarding-token-salt:" + SUPABASE_SERVICE_ROLE_KEY`.

**Payload type:**
```typescript
interface OnboardingTokenPayload {
  lead_id?: string;
  company?: string;
  contact?: string;
  rep?: string;
  role?: string;
  email?: string;
  phone?: string;
  verticals?: string[];
  exp?: number; // epoch seconds
  [key: string]: unknown;
}
```

**Token lifecycle:**
- `signToken(payload, expiresInHours=72)` -- Creates signed token with 72-hour default expiry.
- `verifyToken(token)` -- Validates signature + checks expiry. Returns `{ valid, payload?, error?, legacy? }`.
- `resignLegacyToken(legacyToken)` -- Re-signs old unsigned (plain base64) tokens.

**Legacy support:** Handles unsigned tokens (plain base64 JSON) from MOP-era. Logs warning but accepts them with `legacy: true`.

### 5B. Token Usage in Pages

- `/onboard/buyer/[token]` -- Token in URL path. Verified on mount via `POST /api/onboard/verify-token`. Pre-fills: lead_id, company, contact, email, campaigns.
- `/onboard/publisher/[token]` -- Same pattern. Pre-fills: lead_id, company, contact, rep, verticals.
- `/verify/[token]` -- Uses `verification_tokens` table instead of HMAC tokens. Different system.
- `/v/[code]` -- Short-link redirector. Looks up `verification_tokens.token` and redirects to appropriate onboard page.

### 5C. Middleware Auth Bypass

**File:** `src/middleware.ts` (lines 37-53)

Paths that bypass authentication:
```
/api/*
/contract-review/*
/c/*
/auth/callback
/welcome/*           <-- team welcome
/ask
/pipeline-board
/contract-queue
/proof
/marketing/*
/samples/*
/test-fixtures/*
/login
/onboard             <-- chat onboarding
/onboard/publisher/* <-- publisher form
/verify/*            <-- verification form
```

**Notable:** `/onboard/buyer/*` is NOT explicitly bypassed. Buyer onboarding pages rely on `/api/*` bypass for their token verification calls, but the page itself would be caught by the middleware. However, the page does work because the middleware redirects unauthenticated users to `/onboard` only when hitting `/`, and other paths redirect to `/login` -- but the buyer page is a client component that renders without server-side auth check in the page component itself. The middleware bypass at `/api/` covers the fetch calls the page makes.

**Post-login redirect logic (lines 108-156):**
- If `setup_complete = false` -> redirect to `/setup`.
- If `has_completed_onboarding = false` -> redirect to `/onboard`.
- Root `/` -> redirect to `/operator`.

---

## 6. Webhook Integration

### 6A. Document-Signed Webhook

**File:** `src/app/api/webhooks/document-signed/route.ts`
**Consumes:** `src/lib/onboarding-automation.ts`

**Two consumption paths:**
1. External signing platform webhook -> `POST /api/webhooks/document-signed`
2. Internal trigger from contract processing pipeline (`src/app/api/contract/process/route.ts`)

**Auto-advancement flow:**
```
Document event received
  -> Validate auth (Bearer/X-Webhook-Secret/X-Internal-Secret)
  -> resolveStepKey(documentType, action) -> step_key
  -> findMatchingRuns(event) -> matching runs
  -> For each run:
     -> advanceStep(runId, stepKey)
     -> checkAndAdvancePipeline(runId)
        -> If 100% complete: advance prospect_pipeline from 'onboarding' to 'activation'
  -> Log to ai_action_log
```

### 6B. Stall Detection Cron

**File:** `src/app/api/cron/onboarding-stall-check/route.ts`
**Schedule:** Daily at 6 AM Mountain (13:00 UTC)

**Flow:**
```
findStalledOnboarding() (72+ hours active)
  -> For each stalled step:
     -> Resolve entity name from run metadata or prospect_pipeline
     -> Dedup check: existing open action_item with same source_id?
     -> Create action_item (priority: high if 7+ days, medium otherwise)
     -> Due date: 2 days from creation
```

---

## 7. OnboardingChecklist Widget

**File:** `src/components/operator/OnboardingChecklist.tsx` (774 lines)

**Props:**
```typescript
interface OnboardingChecklistProps {
  entityName?: string;  // filter runs by entity name
  runId?: string;       // direct run ID
  defaultCollapsed?: boolean;
}
```

**Rendered in:** EntityDetailPanel (for entities in onboarding stage), operator views.

**Fetches:** `GET /api/onboarding/runs?entity_name=X&status=active`

**Features:**
- Collapsible section with full-bar clickable header (not just chevron).
- Progress bar (blue normal, orange stalled, green complete).
- Stall warning banner: "A step has been active for N+ days without progress".
- Per-step display: status indicator (checkmark, circle, etc.), label, completion date, stall duration.
- Action buttons per step: "Mark Complete" (calls advance API), "Skip" (opens reason input, calls skip API).
- Step status indicators: completed (green checkmark), active (blue circle), pending (gray circle), skipped (orange strike), failed (red X), blocked (gray square).
- Optional tag on non-required steps.
- Multi-run support with run-level headers.
- Error state with retry button.
- Empty state: "No active onboarding runs found."
- Dark theme (#1c1c1e bg) matching operator UI.

---

## 8. Milo Tool Integration

### 8A. `setup_new_buyer` (Tool 9B)

**File:** `src/lib/tools/setup-new-buyer.ts` (355 lines)

End-to-end buyer setup tool. Calls `startBuyerOnboarding()` at step 10 after:
1. Scam check
2. IO check
3. Contact creation
4. Pipeline entry
5. Campaign resolution
6. TrackDrive buyer creation
7. td_buyer_id persistence
8. Campaign auto-detect
9. CRM sync

### 8B. `setup_new_publisher` (Tool 10A)

**File:** `src/lib/tools/setup-new-publisher.ts` (373 lines)

Uses `triggerPublisherOnboarding()` from publisher-onboarding.ts for steps 2-3 (contact + pipeline + scam check + onboarding run start). Then does TrackDrive traffic source creation, campaign_routes insertion, auto-detect, CRM sync.

---

## 9. TLP-Specific Content Found in Onboarding Code

| Location | TLP-Specific Content |
|----------|---------------------|
| `/onboard` page | "I run operations here at The Lead Penguin." (line 233) |
| Buyer/publisher pages | `<img src="/tlp-logo.png"` used throughout |
| Buyer/publisher pages | "The Lead Penguin" text in footer, emails, headings |
| Publisher page | True inbound snarky verification text |
| Confirmation emails | `support@theleadpenguin.com`, `https://tlp.justmilo.app/tlp-logo.png` |
| Confirmation emails | "The Lead Penguin" branding throughout HTML templates |
| Verticals list | ACA, SSDI, Final Expense, Medicare, Auto Insurance, etc. (PPC industry) |
| Role definitions | Billing, Publisher QA, Outreach, Prospecting, DID Ops, Admin |
| Focus area tips | "I auto-prepare invoices from call data", "ConvoQC flags", etc. |
| Welcome roster | 11 hardcoded team members with personalized lines |
| Onboarding client | `tenantId: 'tlp'` (line 35) |
| Buyer flow steps | td_buyer_created, campaigns_assigned (TrackDrive-specific) |
| Publisher flow steps | did_provisioned, config_validated (DID/TrackDrive-specific) |

---

## 10. Flow Diagram -- Complete Onboarding Architecture

```
                        ┌──────────────────────────────────────┐
                        │         ENTRY POINTS                  │
                        └──────────────────────────────────────┘
                                        │
           ┌────────────────────────────┼────────────────────────────┐
           │                            │                            │
    TEAM MEMBERS              EXTERNAL PARTNERS           OPERATOR-INITIATED
           │                            │                            │
    ┌──────┴──────┐            ┌────────┴────────┐        ┌──────────┴──────────┐
    │             │            │                 │        │                     │
  /welcome     /onboard    /onboard/buyer    /onboard/   /api/publisher-        │
  /[slug]      (chat)      /[token]          publisher/  onboarding             │
    │             │            │              /[token]    (trigger_onboarding)   │
    │             │            │                 │        │                     │
    │             │            ▼                 ▼        ▼                     │
    │             │    ┌───────────────────────────────────┐                    │
    │             │    │     HMAC Token Verification       │                    │
    │             │    │   POST /api/onboard/verify-token  │                    │
    │             │    └───────────────────────────────────┘                    │
    │             │            │                 │                              │
    ▼             ▼            ▼                 ▼                              │
┌─────────┐ ┌─────────┐ ┌──────────┐  ┌──────────────┐                        │
│ Claim   │ │ Signup  │ │ Buyer    │  │ Publisher    │                        │
│ Route   │ │ Route   │ │ Route    │  │ Route        │                        │
│ (POST)  │ │ (POST)  │ │ (POST)  │  │ (POST)       │                        │
└────┬────┘ └────┬────┘ └────┬─────┘  └──────┬───────┘                        │
     │           │           │               │                                │
     │           │           ├───────────────┤                                │
     │           │           ▼               ▼                                │
     │           │    ┌─────────────────────────────────┐                     │
     │           │    │   Blacklist Screening            │                     │
     │           │    │   screenEntity()                 │◄────────────────────┘
     │           │    └──────────────┬──────────────────┘
     │           │                   │
     │           │                   ▼
     │           │    ┌─────────────────────────────────┐
     │           │    │   CRM + Pipeline                │
     │           │    │   findOrCreatePipelineEntry()    │
     │           │    │   findOrCreateCrmContact()       │
     │           │    └──────────────┬──────────────────┘
     │           │                   │
     ▼           ▼                   ▼
┌───────────────────────────────────────────────────────┐
│          @milo/onboarding Engine                      │
│   startPublisherOnboarding() / startBuyerOnboarding() │
│                                                       │
│   onboarding_flows -> onboarding_runs -> onboarding_steps │
│   (DAG state machine with parallel step support)      │
└───────────────────┬───────────────────────────────────┘
                    │
         ┌──────────┼──────────┐
         │          │          │
         ▼          ▼          ▼
   ┌──────────┐ ┌────────┐ ┌──────────┐
   │ Webhook  │ │ Cron   │ │ Operator │
   │ Auto-    │ │ Stall  │ │ Checklist│
   │ Advance  │ │ Check  │ │ Widget   │
   └──────────┘ └────────┘ └──────────┘
```

---

## 11. Tables Touched by Onboarding Code

| Table | Read | Write | Purpose |
|-------|------|-------|---------|
| auth.users | yes | yes | User creation (signup), credential updates (welcome claim) |
| user_profiles | yes | yes | Profile setup, onboarding state flags, welcome claim |
| team_members | no | yes | Team member creation on signup |
| milo_conversations | no | yes | Onboarding conversation log |
| prospect_pipeline | yes | yes | Pipeline entry creation, stage advancement |
| contacts | yes | yes | Contact creation/lookup |
| action_items | yes | yes | Task creation for operator review, stall alerts |
| ai_action_log | no | yes | Audit trail for all onboarding events |
| agreed_terms | yes | yes | Commercial terms for IO generation |
| contract_documents | yes | no | Document signing status check |
| onboarding_flows | yes | yes | Flow definitions (via @milo/onboarding) |
| onboarding_runs | yes | yes | Run lifecycle (via @milo/onboarding) |
| onboarding_steps | yes | yes | Step state (via @milo/onboarding) |
| verification_tokens | yes | no | Token lookup for /v/ redirect and /verify/ page |
| blacklist_entries | yes | no | Screening via screenEntity() |
| crm_counterparties | yes | yes | CRM sync |
| campaigns | yes | no | Campaign resolution for buyer/publisher setup |
| campaign_routes | no | yes | Publisher campaign routing (setup tool) |

---

## 12. Key Observations for Company Admin + Team Onboarding Build

1. **No company/org concept exists.** The current system is single-tenant (TLP). `tenantId: 'tlp'` is hardcoded in `onboarding-client.ts`. Team members are a flat list in `TEAM_ROSTER`. There is no `companies` or `organizations` table.

2. **Team member onboarding has two paths that should NOT be duplicated:**
   - `/onboard` (chat flow) -- creates user from scratch, auto-signs in
   - `/welcome/[slug]` (cinematic) -- claims pre-created user, sets password

3. **The chat onboarding (`/onboard`) collects:** name, email, role (from 6 options), focus areas (from role-specific lists). This is the model for what a "team invite" flow would collect.

4. **`user_profiles` flags that gate navigation:**
   - `setup_complete` -- if false, redirect to `/setup`
   - `has_completed_onboarding` -- if false, redirect to `/onboard`
   - `has_completed_walkthrough` -- gates OperatorTour on first visit
   - `has_claimed_welcome` -- gates one-time welcome link usage

5. **External onboarding (buyer/publisher) is fully separate** from internal team onboarding. Different pages, different APIs, different auth model. No overlap except the `@milo/onboarding` engine.

6. **The `@milo/onboarding` engine is already multi-tenant** (via `tenant_id`). Adding company-scoped onboarding would mean new flows defined per company, not new engine code.

7. **ADMIN_EMAILS is hardcoded** in `/onboard/page.tsx` line 45: `['mark@scaleflow.co', 'tiffani@scaleflow.co']`. Admin role pill only shows for these emails. This must change for multi-company support.

8. **Focus area mapping** (`src/lib/focus-area-mapping.ts`) resolves custom focus strings to known block IDs. Used during signup to set initial dashboard layout.

9. **Middleware buyer bypass gap:** `/onboard/buyer/*` is not explicitly listed in middleware bypass (line 50-51 only lists `/onboard` and `/onboard/publisher/*`). The buyer page works because it's a client component and the API calls go through `/api/*` bypass, but this could cause issues with SSR or direct navigation.

10. **Confirmation emails** are sent via Resend for both buyer and publisher submissions. Templates are inline HTML strings in the route files, not shared components.

Report at: https://raw.githubusercontent.com/moscaleflow/conductor-mark-public/main/reports/C3-ONBOARDING-AUDIT-2026-05-07.md
