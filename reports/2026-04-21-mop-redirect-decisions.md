# MOP Redirect — Mark Decision Sheet

> **How to use:** Read each Q, pick an option (or write your own), add any notes. Strategy-Claude will convert your picks into implementation directives.

> **Source spec:** [specs/MOP-SIGNING-REDIRECT-SPEC.md](../specs/MOP-SIGNING-REDIRECT-SPEC.md) (e39476c)

---

### Q1. MOP Netlify deploy access

> "The `_redirects` deploy requires pushing to MOP's git repo and Netlify auto-deploying. Does Coder-4 have push access to MOP, or does Mark need to execute the deploy? MOP is frozen — is a `_redirects`-only commit acceptable under the freeze policy, or does Mark want to formally unfreeze for this single change?"

**Context:** MOP is declared "frozen forever" (SESSION-HANDOFF). D52 deployed a selective contract-writes freeze via `CONTRACT_WRITES_FROZEN` env var. The redirect is a static `_redirects` file — no app code changes, no route handler modifications. Drift guard Pattern 7 requires freeze/unfreeze to be git-committed (no deploy-time-only state). The `_redirects` file is git-committed by definition.

**Options:**
- **Option A: Allow `_redirects`-only commit under existing freeze policy.** The freeze targets app code and write paths. A static redirect file doesn't modify any route handler, API endpoint, or business logic. Netlify processes `_redirects` at the edge before the app starts. This is infrastructure, not app code. — *Pro:* No policy change needed, Coder-4 can execute. *Con:* Sets a precedent that "frozen" repos accept infrastructure commits.
- **Option B: Formally unfreeze MOP for this single change, then re-freeze.** Log a decision: "MOP unfrozen for redirect commit [SHA], re-frozen immediately after." — *Pro:* Clean audit trail, no ambiguity about freeze policy. *Con:* Ceremony for a static file.
- **Option C: Mark executes the deploy personally.** Coder-4 writes the `_redirects` file content, Mark pushes it. — *Pro:* Mark retains direct control over frozen repo changes. *Con:* Burns Mark's time on dev-team work (Pattern 13).

**Recommendation:** Option A. The `_redirects` file is edge infrastructure, not app code. The freeze (D52) targets contract write paths — `_redirects` doesn't touch any of them. Pattern 7 is satisfied because the file is git-committed. No unfreeze ceremony needed.

**Risk if deferred:** Redirect implementation cannot start. All 68 pending signing recipients stay pointed at frozen MOP.

**Reversibility:** Fully reversible. Delete `_redirects`, push, Netlify redeploys without redirects in ~2 minutes.

---

### Q2. Short code redirect scope — **LOCKED — Decision 78**

> "MOP's `/s/[code]` endpoint also resolves invoice short codes (not just signing). Should the redirect cover `/s/*` (all short codes, including invoices) or only signing short codes? Invoices are not migrated — redirecting invoice short codes to tlp.justmilo.app would 404."

**Context:** MOP's `/s/[code]` route (spec §1) is a generic short code resolver. It looks up the code in `signing_documents.short_code` first, then falls back to `invoices.short_code`. Only signing data was migrated (D54: 109 docs). Invoice data was not migrated — `/invoice/[token]` is explicitly out of scope in spec §1. A blanket `/s/*` redirect would send invoice short codes to tlp.justmilo.app where they'd 404.

**Options:**
- **Option A: Exclude `/s/[code]` from static redirects entirely.** Leave `/s/*` on MOP. Signing short codes still resolve on MOP (read path is unfrozen). Invoice short codes also still resolve. — *Pro:* Zero risk of 404s. *Con:* Signing short code users stay on MOP permanently; no path to MOP decommission for `/s/*`.
- **Option B: Redirect `/s/*` with a Netlify function that checks type before redirecting.** Deploy a Netlify serverless function that queries the DB to determine if a short code is signing or invoice, then redirects signing codes to tlp.justmilo.app and serves invoices locally. — *Pro:* Signing codes migrate cleanly, invoices stay working. *Con:* Requires unfreezing MOP more than a static file; adds a serverless function to a "frozen" repo; ongoing maintenance.
- **Option C: Redirect `/s/*` blanket, accept invoice 404s.** — *Pro:* Simple. *Con:* Any invoice short code links in the wild will break.

**Recommendation:** Option A (per spec §6 recommendation). Leave `/s/[code]` on MOP. The signing short codes still resolve because MOP's read paths are unfrozen. This avoids invoice breakage and avoids adding serverless functions to a frozen repo. When MOP decommission approaches (6-12 months), revisit whether `/s/*` needs migration.

**Risk if deferred:** None immediate. `/s/*` works on MOP today. Decision only matters for eventual MOP decommission timeline.

**Reversibility:** Fully reversible. `/s/*` can be added to `_redirects` later if invoices are migrated or deprecated.

---

### Q3. Counterparty notification for active negotiation — **LOCKED — Decision 83**

> "Should Mark send a courtesy email to the counterparty on the active negotiation before cutover? Not technically required (redirect is transparent), but reduces confusion if they notice the URL change in their browser bar."

**Context:** D56 migrated 1 active negotiation in `counterparty_review` state with 5 links. The redirect (301) is transparent — the counterparty's existing link will auto-redirect to tlp.justmilo.app. Their browser bar will show the new domain. The signing page content and token are identical. The only visible change is the domain name.

**Options:**
- **Option A: Send courtesy email before cutover.** Brief note: "We've upgraded our platform. Your review link will redirect automatically." — *Pro:* Professional, reduces support inquiries if counterparty notices domain change. *Con:* Draws attention to a change the counterparty might never notice; requires Mark to draft and send.
- **Option B: No notification.** The redirect is transparent by design. 301 is the standard mechanism for permanent moves. — *Pro:* Zero effort, no unnecessary attention to the migration. *Con:* If the counterparty is security-conscious, they might flag the domain change.
- **Option C: Notification only if the counterparty clicks within 48h of cutover.** Set up a monitoring trigger — if Netlify logs show a `/review/*` redirect hit, Mark sends a follow-up email. — *Pro:* Only notifies if needed. *Con:* Monitoring overhead, delayed notification.

**Recommendation:** Silent redirect, no proactive notification. Notify only if the counterparty reports a broken link. The 1 active negotiation is in counterparty_review state — sending "our URLs changed" is an objection-generating interruption during legal review. Working redirect = they never need to know.

**Risk if deferred:** Low. Redirect works regardless of notification. Worst case: counterparty contacts Mark asking about the domain change, Mark explains in 30 seconds.

**Reversibility:** N/A — a sent email can't be unsent, but it's low-stakes.

---

### Q4. MOP decommission timeline — **LOCKED — Decision 84**

> "How long should MOP stay live as a redirect target before full decommission? Recommend 6 months minimum — long enough for all 68 pending recipients to either sign or let their tokens expire. After that, MOP Netlify site can be deleted."

**Context:** MOP serves as the redirect origin — Netlify edge processes `_redirects` and sends 301s to tlp.justmilo.app. Once MOP is deleted, the `tlpmop.netlify.app` domain stops resolving entirely. 68 pending signing documents have URLs in the wild. Migrated tokens have `expires_at: null` (D54), so they never expire on the database side. The question is how long real-world links survive in email inboxes, bookmarks, and browser history.

**Options:**
- **Option A: 6 months (October 2026).** — *Pro:* Covers most realistic email link lifetimes. *Con:* 6 months of Netlify hosting cost (minimal — free tier likely covers a static redirect).
- **Option B: 12 months (April 2027).** — *Pro:* Covers even long-dormant counterparties who might dig up old emails. *Con:* Longer maintenance window. MOP repo stays in "don't touch" state longer.
- **Option C: Indefinite — keep MOP redirect alive until Mark explicitly decommissions.** — *Pro:* Zero risk of breaking any link ever. *Con:* MOP never fully dies; perpetual (tiny) hosting cost.
- **Option D: 3 months (July 2026) with monitoring.** Check Netlify redirect hit count at month 2. If zero hits in the last 30 days, decommission early. — *Pro:* Data-driven, potentially faster cleanup. *Con:* Requires someone to check analytics.

**Recommendation:** Decommission trigger formula: **(60 days of zero redirect hits on Netlify) OR (6 months elapsed) — whichever is later.**

Per D54, signing tokens were migrated with `expires_at: null` (never expire). Token expiration cannot serve as a decommission signal.

Live-token query (run during D87, read-only against shared Supabase tappyckcteqgryjniwjg):
```
live_tokens:        0  (expires_at > NOW() — none, all are NULL)
expired_tokens:     0  (expires_at <= NOW() — none, all are NULL)
latest_expiration:  null
```
Supplementary: 68 pending + 13 viewed = 81 active tokens, ALL with `expires_at: null`.

Since all MOP-era tokens never expire at the DB level, the calendar floor of 6 months becomes the effective minimum. The 60-day-zero-hits gate allows earlier decommission only if traffic proves dead before the 6-month mark.

**Risk if deferred:** None. MOP stays live regardless. This decision only matters when someone wants to delete the Netlify site.

**Reversibility:** Fully reversible — MOP can be re-deployed from git at any time if decommissioned too early.

---

### Q5. Invoice and offer routes — **LOCKED — Decision 79**

> "`/invoice/[token]` and `/offer/[token]` are out of scope for this redirect (not migrated to primitives). Should they stay live on MOP indefinitely, or should MOP serve a static 'this service has moved' page for all non-redirected routes?"

**Context:** Spec §1 lists `/invoice/[token]` and `/offer/[token]` as out of scope — invoicing and campaign offers were not extracted to @milo/* primitives. MOP's read paths for these routes are unfrozen (D52 only froze contract write paths). These routes currently work on MOP. The question is what happens to them when/if MOP is decommissioned.

**Options:**
- **Option A: Leave them live on MOP until decommission, then let them 404.** — *Pro:* Zero work now. Invoicing and offers are legacy MOP features with no active users (per spec: "no pending actions"). *Con:* If any invoice links exist in the wild, they'll break on decommission.
- **Option B: Add a catch-all `_redirects` rule for non-redirected routes → static "service moved" page.** After the signing/review redirects, add `/* /moved.html 200` as a catch-all. Deploy a simple `moved.html` with Mark's contact info. — *Pro:* Clean UX for anyone hitting any MOP URL after signing redirects go live. *Con:* Requires building and deploying a static page. Minor effort.
- **Option C: Migrate invoice/offer data to primitives before decommission.** — *Pro:* Complete migration, no orphaned features. *Con:* Significant work for features with "no pending actions." Violates Pattern 12 (solving unnamed problem with named infrastructure).

**Recommendation:** Option A for now. Spec says "no pending actions" on invoices and offers. If Q4 monitoring (Option D) shows zero hits on `/invoice/*` and `/offer/*` in 3 months, they're confirmed dead. If hits appear, revisit with Option B as a catch-all.

**Risk if deferred:** None. Invoice/offer routes work on MOP today. Only matters at decommission time.

**Reversibility:** Fully reversible. Can add catch-all redirect at any point before decommission.

---

### Q6. PPCRM webhook URLs — **LOCKED — Decision 80**

> "MOP's signing submit route fires webhooks to `PPCRM_WEBHOOK_URL` (event `document.signed`, `document.declined`). PPCRM is frozen. After redirect, the new tlp.justmilo.app submit handler needs to fire the same webhooks — or not, if PPCRM is dead. Does the new handler need webhook parity with MOP, or can webhooks be dropped since PPCRM is frozen?"

**Context:** MOP's `/api/sign/[token]/submit` fires webhooks to PPCRM on `document.signed` and `document.declined` events. PPCRM (jdzqkaxmnqbboqefjolf) is frozen forever — its webhook receiver is a Next.js API route that updates PPCRM's local state. Since PPCRM is frozen and no one reads its state, these webhooks write to a database no one queries.

**Settled by prior decisions:** PPCRM is "frozen forever" (SESSION-HANDOFF, MASTER-PROMPT). Waves 2-3 consumer migrations are "permablocked." No PPCRM consumer reads the webhook-updated state.

**Options:**
- **Option A: Drop PPCRM webhooks entirely.** The new tlp.justmilo.app submit handler does not fire webhooks to PPCRM. — *Pro:* Clean break. No dependency on frozen infra. Removes a dead integration. *Con:* If anyone or anything reads PPCRM signing state (unlikely given "frozen forever"), they'd see stale data.
- **Option B: Maintain webhook parity in new handler.** Fire the same `document.signed`/`document.declined` events to PPCRM. — *Pro:* Zero behavioral change from MOP. *Con:* Perpetuates a dependency on frozen infrastructure. PPCRM's webhook receiver may not even be running. The HMAC format changed (spec §2 incompatibilities table) — new handler would need to use the OLD HMAC format to match PPCRM's verifier, creating backwards-compat code.
- **Option C: Fire webhooks to a new generic endpoint (future-proofing).** Replace PPCRM-specific webhook with a generic event bus. — *Pro:* Future-ready. *Con:* Speculative infrastructure with no current consumer (Pattern 12).

**Recommendation: Option A — propose locking without Mark review.** PPCRM is frozen forever. No consumer reads its webhook-updated state. Maintaining webhook parity (Option B) requires backwards-compat HMAC code for a dead receiver. Building a generic event bus (Option C) is Pattern 12 (no current consumer). Drop the webhooks. If a future need arises, @milo/contract-signing can add an event hook interface then.

**Risk if deferred:** Low but creates confusion. If the new submit handler is built without deciding this, the implementer will ask "should I wire up PPCRM webhooks?" mid-directive, burning a round-trip.

**Reversibility:** Fully reversible. Webhooks can be added to the new handler at any time if a consumer appears.

---

## Summary

| Q | Topic | Status | Decision # |
|---|-------|--------|------------|
| Q1 | MOP deploy access / freeze policy | MARK-DECISION (open) | — |
| Q2 | Short code redirect scope | **LOCKED** — Option A | D78 |
| Q3 | Counterparty notification | **LOCKED** — silent redirect | D83 |
| Q4 | Decommission timeline | **LOCKED** — trigger formula | D84 |
| Q5 | Invoice/offer routes | **LOCKED** — Option A | D79 |
| Q6 | PPCRM webhooks | **LOCKED** — Option A | D80 |

**Locked (D87):** Q2 (D78), Q5 (D79), Q6 (D80). Settled by prior decisions.
**Locked (D100):** Q3 (D83), Q4 (D84). Mark verbal direction.

**Needs Mark:** Q1 only (freeze policy / deploy access — TrackDrive proximity requires explicit yes).
