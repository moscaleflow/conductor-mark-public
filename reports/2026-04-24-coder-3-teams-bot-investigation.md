# D71 — PPCRM Teams Bot Code Investigation

**Date:** 2026-04-24  
**Agent:** Coder-3  
**Scope:** Research-only. Determine if existing Teams bot is port-ready for PPC Fight Club use case.

---

## Exec Summary

**Verdict: YELLOW — mostly clean, 1-3 days to port.**

The PPCRM Teams bot is a well-architected Supabase Edge Function (Deno) that receives webhook POSTs from Azure Bot Service, routes messages through a VACoordinator intent classifier, and replies back via Bot Connector REST API. It supports text, Adaptive Cards, PDF upload/analysis, typing indicators, proactive DMs, and conversation reference capture. The Bot Framework transport layer (~250 LOC across `getBotToken` + `sendReply` + `sendTyping` + `downloadTeamsAttachment`) is completely generic and reusable as-is. The coupling is in two areas: (1) the VACoordinator routing layer delegates to 6 PPCRM-specific sub-bots (opsbot, billingbot, salesbot, onboardbot, did-manager, briefing), and (2) it writes to PPCRM-specific tables (bot_logs, partner_channels, va_profiles, documents). Both are swappable — the transport doesn't care what backend it talks to.

---

## 1. Code Location + File Inventory

All bot code lives in `~/PPC3.18/PPCRM/supabase/functions/` (Deno Edge Functions).

### Core bot files

| File | LOC | Purpose |
|---|---|---|
| `teams-bot/index.ts` | 1,083 | Main webhook handler — receives Teams activities, routes messages, sends replies |
| `send-message/index.ts` | 585 | Outbound messaging relay — proactive channel posts, DMs, Power Automate, legacy webhooks |
| `_shared/teams-dm.ts` | 121 | Proactive 1:1 DM helper via Bot Framework |
| `_shared/personas.ts` | 83 | Team persona definitions (Milo, Rex, Lexi, Nora, Dana) + formatting |
| `_shared/adaptive-cards.ts` | 1,955 | Adaptive Card templates for Teams UI (document generated, help menu, etc.) |
| `_shared/crm-reader.ts` | 78 | CRM database reader (separate Supabase project for @milo/crm) |
| `_shared/freeze-guard.ts` | 16 | Write-freeze guard (PPCRM_WRITES_FROZEN env) |
| **Subtotal (bot transport + shared)** | **3,921** | |

### Supporting routing layer (NOT the bot itself)

| File | LOC | Purpose |
|---|---|---|
| `vacoordinator/index.ts` | 1,914 | Central intent classifier + router to sub-bots |
| `opsbot/index.ts` | ~varies | Rex: reports, alerts, call metrics |
| `billingbot/index.ts` | ~varies | Nora: invoices, aging, reconciliation |
| `salesbot/index.ts` | ~varies | Lexi: vetting, pipeline, pricing |
| `onboardbot/index.ts` | ~varies | Dana: checklists, documents, onboarding |

### Teams App manifest

| File | LOC | Purpose |
|---|---|---|
| `teams-app/manifest.json` | 49 | Teams app definition — bot ID, scopes, static tab, valid domains |
| `teams-app/build-package.sh` | 24 | Zips manifest + icons into MiloApp.zip for sideloading |
| `teams-app/color.png` | — | Bot icon (color) |
| `teams-app/outline.png` | — | Bot icon (outline) |

### Dashboard client

| File | LOC | Purpose |
|---|---|---|
| `dashboard/workspaces/saleschat.js` | 1,224 | Browser-side chat UI (vanilla React, `window.PPCRM`) — NOT part of the bot server |

---

## 2. Architecture Walkthrough

### Inbound message flow

```
Teams user @mentions Milo or sends message in 1:1 chat
    │
    ▼
Azure Bot Service receives activity
    │
    ▼
POST → Supabase Edge Function: /functions/v1/teams-bot
    │  (JWT verification disabled in config.toml)
    │
    ▼
teams-bot/index.ts: Deno.serve(async (req) => { ... })
    │
    ├── activity.type === 'conversationUpdate' → sends greeting
    │
    ├── activity.type === 'message'
    │   ├── stripMention() → clean text
    │   ├── Check Adaptive Card button clicks (activity.value.command)
    │   ├── Greeting interception (/^hi|hello|hey/i → friendly reply)
    │   ├── Help menu interception → send interactive Adaptive Card
    │   ├── Capture AAD Object ID → upsert va_profiles (fire-and-forget)
    │   ├── Capture conversation_reference → upsert partner_channels
    │   ├── sendTyping() (fire-and-forget)
    │   ├── PDF attachment routing:
    │   │   ├── "file revision" + draft docs → onboardbot/file-revision
    │   │   ├── "analyze" keyword → onboardbot/analyze-contract
    │   │   └── No context → default to analysis
    │   ├── Send acknowledgment ("On it." / "Working on it...")
    │   ├── Progress timer (8s → "Taking a bit longer...")
    │   └── routeMessage() → VACoordinator → sub-bot → format reply → sendReply()
    │
    └── activity.type === 'invoke' → Adaptive Card Universal Action handling
```

### Outbound message flow

The bot **DOES reply** back to Teams. Three methods:

1. **sendReply()** — direct Bot Connector API POST (text or Adaptive Card, with conditional threading)
2. **sendProactiveChannelMessage()** (in send-message) — proactive channel posts using stored conversation references
3. **sendProactiveDM()** (in teams-dm.ts + send-message) — create 1:1 conversation by AAD Object ID, then post

### Authentication

- **Inbound:** Azure Bot Service handles Teams-to-webhook auth. The Edge Function trusts the POST (JWT verification disabled).
- **Bot → Teams:** Client credentials OAuth2 flow against `login.microsoftonline.com/{tenant_id}/oauth2/v2.0/token` with scope `https://api.botframework.com/.default`. Token cached ~1 hour.
- **Bot → Sub-bots:** `EDGE_FUNCTION_JWT` (custom secret, not the auto-injected service role key).
- **send-message API key:** `MILO_MESSAGING_KEY` in `x-api-key` header for external callers.

---

## 3. Azure Infrastructure Status

### Registered bot

| Item | Value | Status |
|---|---|---|
| Bot App ID | `4f48d0f3-35c6-4da9-ad31-ebe2b7e37b06` | In manifest.json — likely **still registered** in Azure (registration doesn't expire with Supabase freeze) |
| Azure AD Tenant ID | Via `AZURE_BOT_TENANT_ID` env var | **UNKNOWN** — value in frozen Supabase secrets |
| Bot App Secret | Via `AZURE_BOT_APP_SECRET` env var | **UNKNOWN** — may have expired (Azure app secrets expire 1-2 years) |
| Messaging endpoint | `https://jdzqkaxmnqbboqefjolf.supabase.co/functions/v1/teams-bot` | **FROZEN** — responds with 503 freeze guard for POST requests |
| Teams manifest version | 1.0.6 (manifest version 1.17) | Sideloaded — not in Teams App Store |

### Channel infrastructure

| Channel | Method | Status |
|---|---|---|
| Alerts | Bot Framework proactive (conversation ID `19:1496b...@thread.tacv2`) | **FROZEN** (write-blocked) |
| Operations | Power Automate workflow URL (hardcoded) | **UNKNOWN** — Power Automate flow may still be active |
| Billing, Sales, Huddle, Onboarding | Legacy Office 365 connectors (env var URLs) | **LIKELY DEAD** — Microsoft retired O365 connectors in 2025 |

### Teams App details

- **Developer:** The Lead Penguin
- **Scopes:** personal, team, groupChat
- **Static tab:** "Command Center" → ppcrm-dashboard.netlify.app (the PPCRM vanilla React dashboard)
- **supportsFiles:** false (files received via Teams attachment mechanism regardless)
- **Valid domains:** jdzqkaxmnqbboqefjolf.supabase.co, ppcrm-dashboard.netlify.app

---

## 4. Port-Readiness Assessment

### GREEN — Transport layer (reuse as-is)

These functions are completely generic — no PPCRM-specific logic:

| Function | LOC | What it does |
|---|---|---|
| `getBotToken()` | ~40 | OAuth2 client credentials → Bot Framework token (cached) |
| `sendReply()` | ~48 | POST activity back to Teams conversation (text or Adaptive Card, with threading) |
| `sendTyping()` | ~16 | Typing indicator |
| `downloadTeamsAttachment()` | ~30 | Fetch file from Teams via bot token |
| `stripMention()` | ~7 | Strip `<at>Milo</at>` from text |
| `sendProactiveDM()` (teams-dm.ts) | ~65 | Create 1:1 conversation + send message by AAD Object ID |
| `sendProactiveChannelMessage()` (send-message) | ~88 | Post to channel by conversation ID |
| **Total GREEN** | **~294** | |

### YELLOW — Needs modification (coupling points)

| Area | What's coupled | Effort |
|---|---|---|
| **VACoordinator routing** | `routeMessage()` calls `/functions/v1/vacoordinator/command` which routes to 6 PPCRM sub-bots. Fight Club would replace this with a single call to Milo `/ask` Vet pill. | Replace 1 function (~80 LOC → ~30 LOC) |
| **Persona system** | 5 TLP-specific personas (Milo, Rex, Lexi, Nora, Dana). Fight Club would need different personas or simplified single-persona. | Swap `personas.ts` content (~83 LOC) |
| **Adaptive Cards** | 1,955 LOC of TLP-specific card templates (document generated, EOD report, partner vetting, help menu). Fight Club needs different card templates for vet responses. | Replace relevant cards (~200 LOC new, drop ~1,700 LOC unused) |
| **Conversation reference capture** | Writes to `partner_channels` with partner detection via `crm_counterparties` lookup. Fight Club may want conversation capture but against different tables. | ~20 LOC to retarget |
| **AAD Object ID capture** | Writes to `va_profiles.aad_object_id` and `.bot_framework_id`. Fight Club would write to a different user table or skip. | ~15 LOC to retarget or remove |
| **PDF upload routing** | 3 scenarios route to `onboardbot/file-revision` and `onboardbot/analyze-contract`. Fight Club doesn't need this. | Delete ~250 LOC |
| **Help menu interception** | Queries `crm_counterparties` for recent partners dropdown. Fight Club would have different help content. | ~30 LOC to rewrite |
| **Greeting patterns** | Includes Tagalog greetings (`kumusta`, `magandang umaga`). Minor — keep or trim. | ~2 LOC |
| **freeze-guard.ts** | PPCRM freeze wrapper. Not needed for new deployment. | Remove 1-line import |

### RED — Nothing. No deeply tangled code.

---

## 5. Minimal Viable Rebuild for Fight Club

### Target use case

PPC Fight Club chat → Teams message → Milo Vet pill → structured vet response → reply to Teams

### Architecture

```
Fight Club Teams chat
    │
    ▼
Azure Bot Service (reuse existing bot registration OR create new)
    │
    ▼
Next.js API route: /api/teams-bot (on milo-for-ppc, NOT Supabase Edge Function)
    │
    ├── Extract message text (reuse stripMention)
    ├── Call /api/milo/stream with vet prompt  ← replaces VACoordinator
    ├── Format response as Adaptive Card or markdown
    └── sendReply() back to Teams
```

### What to reuse from existing bot

| Component | Action | LOC |
|---|---|---|
| `getBotToken()` | Copy to Next.js `lib/teams-bot.ts` | 40 |
| `sendReply()` | Copy | 48 |
| `sendTyping()` | Copy | 16 |
| `stripMention()` | Copy | 7 |
| `manifest.json` structure | Fork, update bot ID + domain | 49 |
| **Total reuse** | | **~160** |

### What to build new

| Component | What | ~LOC |
|---|---|---|
| `/api/teams-bot/route.ts` | Webhook handler — parse activity, route to Milo, reply | ~80 |
| `lib/teams-bot.ts` | Ported transport functions (getBotToken, sendReply, sendTyping, stripMention) | ~120 (extracted from existing) |
| Vet response formatter | Format Milo vet result as Teams markdown or Adaptive Card | ~40 |
| Azure Bot registration | New registration in Azure Portal (or reuse existing 4f48d0f3 if credentials still valid) | Config only |
| Teams app manifest | Fork existing, update botId + validDomains | Config only |
| **Total new code** | | **~240** |

### Complexity rating: LOW-MEDIUM

- Bot Framework REST API is well-documented, no SDK needed (existing code uses raw fetch)
- No npm dependencies beyond what's already in milo-for-ppc
- The hardest part is Azure setup (bot registration, app secret, Teams manifest sideloading) — not code

### Build estimate

- **Code:** 4-6 hours (port transport, write handler, format responses)
- **Azure setup:** 1-2 hours (register bot or recover credentials, update messaging endpoint, test)
- **Teams manifest + sideload:** 30 minutes
- **Total:** ~1 day if Azure credentials are recoverable, ~1.5 days if new registration needed

---

## 6. Open Questions for Mark

1. **Reuse existing Azure bot registration?** Bot ID `4f48d0f3-35c6-4da9-ad31-ebe2b7e37b06` may still be active in Azure AD. If the app secret hasn't expired, we can just update the messaging endpoint URL. Do you have Azure Portal access to check?

2. **Same Teams tenant?** Is PPC Fight Club in the same Azure AD tenant as TLP? If so, same bot registration works. If different tenant, need a new multi-tenant registration.

3. **Response format?** Should vet results post as:
   - Plain markdown (simpler, faster to build)
   - Adaptive Card (richer UI, action buttons — the existing bot has extensive card infrastructure)
   - Both (card for channel, markdown fallback for 1:1)

4. **Scope of monitoring?** The directive says "monitors a PPC Fight Club chat" — is this:
   - Responds only when @mentioned (current bot behavior)
   - Passively reads all messages and auto-identifies vet requests
   - Both (mentions + keyword detection)

5. **Where to deploy?** Options:
   - **milo-for-ppc Next.js API route** (recommended — same deployment, same env vars, same auth)
   - **New Supabase Edge Function** (separate deployment, Deno runtime)
   - **Standalone service** (overkill)

6. **Environment variables needed:** 3 Azure secrets (`AZURE_BOT_APP_ID`, `AZURE_BOT_APP_SECRET`, `AZURE_BOT_TENANT_ID`). Where should these live?

---

## Summary Table

| Aspect | Rating | Notes |
|---|---|---|
| Transport layer | GREEN | 294 LOC, fully generic, copy-paste |
| VACoordinator routing | YELLOW | Replace with Milo /ask call |
| Persona system | YELLOW | Swap content or simplify |
| Adaptive Cards | YELLOW | Build new vet-specific cards, drop TLP templates |
| Azure infrastructure | UNKNOWN | Need Mark to check Azure Portal |
| Overall port | YELLOW | ~1 day build, ~240 LOC new |

---

Report at: https://raw.githubusercontent.com/moscaleflow/conductor-mark-public/main/reports/2026-04-24-coder-3-teams-bot-investigation.md
