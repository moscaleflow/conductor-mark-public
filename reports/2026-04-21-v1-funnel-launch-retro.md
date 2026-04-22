# v1 Funnel Launch Retrospective — 2026-04-21

> Covers the 2026-04-20 through 2026-04-21 session. 20 decisions logged (D53–D72), ~30 directives executed across 4 Coder lanes.

---

## 1. What shipped

| # | Deliverable | Commit / Decision | Notes |
|---|-------------|-------------------|-------|
| 1 | Phase 1 scaffold deployed to Vercel | 8c031b2, D65 | milo-for-ppc.vercel.app live, 90 routes, 0 TS errors |
| 2 | 10-step cinematic Milo-led conversational funnel | 38f4b19, D71 | Vertical gate → role pills → context framing → sample download → drag-drop → analyzing state → results → redline CTA → email capture → thank-you |
| 3 | Host-based middleware routing | D65 | justmilo.app = marketing hero, tlp.justmilo.app = product surface, single deploy |
| 4 | PPC vertical rule pack (32 rules) | 8c031b2, D65 | 7 pub financial, 9 TCPA/compliance, 6 operational, 7 buyer financial, 3 buyer data/liability. 3 PatternPlugins, 4 scoring profiles. |
| 5 | @milo/feedback v0.1.0 primitive | D70 | 8th primitive. 17 tests. Polymorphic feedback collection. |
| 6 | /proof page (demoted from /real-contracts) | 38f4b19, D71 | Anonymized contract showcase, auth chrome stripped, secondary proof asset |
| 7 | 2 real Milo sample PPC contracts | 34a2a3b, D63 | Publisher (11 rules) + buyer (10 rules) PDFs with manifest.json |
| 8 | VISION-MILO-V1-FUNNEL.md north star | 38f4b19, D71 | §3 rewritten for cinematic funnel, §11 voice guide added |
| 9 | Brand positioning locked | d363507, D69 | PPC-only public face, horizontal platform underneath |
| 10 | Milo first-person voice locked | 38f4b19, D71 | "I" / "me" / "Milo" — never "we" / "our team" / "our AI" |
| 11 | 26 open decisions batch-resolved | 28ce204, D64 | System prompt, AI tiers, and 24 others — Phase 1 unblocked |
| 12 | Shared Supabase auth redirect URLs | 9a2a55e, D68 | 3 redirect URLs added for Milo-for-PPC |
| 13 | Demo tenant TTL cron | D65 | Hourly, safe dry-run mode. tenant='demo-ppc', 24h TTL. |
| 14 | Entity mapping table stress-tested and killed | 8157fa9/51f462c, D66→D67 | SCHEMA.md amended then reverted. Option C locked. |
| 15 | Session handoff bundle (11 files) | c2cc27f, D68 | MASTER-PROMPT + PROJECT-GLOSSARY + 9 reference docs for claude.ai upload |
| 16 | 8 new drift guard patterns (9–16) | 28c263b/941a0f2/b4af4ce/8299603 | Patterns 9–16 covering decision collisions through visual verification |

---

## 2. What broke / near-broke

### B1. MOP extract proxy chain broken — Opus 4.7 temperature deprecation

- **Symptom:** All MOP contract extractions returning 500 errors. "MOP extract chain broken" in logs.
- **Root cause:** Anthropic deprecated `temperature` parameter on Opus 4.7. @milo/ai-client's `callClaude()` passed temperature unconditionally. Opus 4.7 returned 400 "temperature is deprecated for this model." Error string didn't match 'api-key'/'auth'/'401' patterns, fell through to generic 500.
- **Detection method:** Coder-4 investigation under D69. Error was masked for hours as generic 500 before raw upstream response was inspected.
- **Time to detect:** Hours (masked by generic error handling).
- **Drift guard:** Pattern 15 written. Rule: AI primitives must maintain model-quirk lookup table; log full upstream error detail, not just status code.
- **Decision:** D72.
- **Status at session end:** Fix in progress (Coder-4 D69).

### B2. Sample PDFs served login.html — middleware regex bug

- **Symptom:** Downloading /samples/ppc/*.pdf returned login.html instead of the PDF file. User sees "login page downloaded as file."
- **Root cause:** Next.js middleware public-route bypass regex excluded .svg/.png/.jpg but NOT .json or .pdf. Any non-image static asset behind a "public" route hit the auth middleware and got redirected.
- **Detection method:** Manual incognito test by Mark.
- **Time to detect:** Caught during QA pass, before external users hit it.
- **Drift guard:** Pattern 14 written. Rule: bypass must cover ALL static extensions; prefer path-prefix bypass over extension regex.

### B3. ANTHROPIC_API_KEY stale in Vercel production

- **Symptom:** API calls failing in production after Phase 1 deploy.
- **Root cause:** Key in Vercel env vars was stale/expired. Refreshed via D68.
- **Detection method:** Deploy-time error.
- **Time to detect:** Immediate on first production API call attempt.
- **Status:** Resolved.

### B4. Purple MILO logo on hero page

- **Symptom:** Hero displayed purple-branded MILO logo instead of white.
- **Root cause:** Logo asset or CSS style inherited from operator dashboard theme.
- **Detection method:** Incognito visual review by Mark.
- **Time to detect:** Caught in first visual pass.
- **Status:** Reverted to white.

### B5. Operator app chrome rendered on /proof (formerly /real-contracts)

- **Symptom:** /real-contracts page showed full operator sidebar + nav chrome instead of bare public page.
- **Root cause:** Layout assumed all routes under the app were operator routes. /real-contracts wasn't marked as a bare route.
- **Detection method:** Visual review.
- **Time to detect:** Caught during QA.
- **Fix:** Added to BARE_ROUTES list, stripped auth chrome.

### B6. Entity mapping table overengineered — SCHEMA.md amended then reverted

- **Symptom:** D55 and D66 designed `crm_entity_id_mappings` table. SCHEMA.md v0.3.0 committed (51f462c). Then D58 stress-test found zero Phase 2 consumers actually need ID resolution.
- **Root cause:** Infrastructure designed for "future need" without verifying immediate consumers. All Phase 2 UIs use denormalized display names already present in migrated data.
- **Detection method:** D58 stress-test (Coder-3) — explicit audit of every Phase 2 UI to check whether it JOINs through entity IDs. None did.
- **Time to detect:** ~4 directives of design work before stress-test killed it.
- **Drift guard:** Pattern 12 written. Rule: verify consumers exist before building. "Enables future features X, Y, Z" with zero current consumers = speculative. Defer.
- **Decision:** D66 (DEFERRED), D67 (Option C locked).

### B7. Build OOM / Vercel deploy instability

- **Symptom:** Vercel builds intermittently OOM during Phase 1 scaffold deploy.
- **Root cause:** 90 Next.js routes + 7 vendored @milo/* packages in single build. Memory pressure at edge of Vercel build limits.
- **Detection method:** Vercel build logs.
- **Time to detect:** During deploy attempts.
- **Status:** Resolved by build config tuning. Noted for monitoring if route count grows.

### B8. D62 visual verification accepted via text — no screenshot

- **Symptom:** Coder reported "the page renders correctly" for synthetic sample contracts without providing a screenshot file.
- **Root cause:** Directive didn't include explicit screenshot requirement. Strategy-Claude accepted text description as verification.
- **Detection method:** Mark flagged the gap during session review.
- **Time to detect:** Post-hoc during process review.
- **Drift guard:** Pattern 16 written. Rule: visual verification directives must demand stitched screenshots saved to ~/Desktop/. Accept nothing less.

---

## 3. What we learned

### Theme 1: The product-is-demo model works — but only if the funnel IS the product

The 10-step cinematic funnel is not a demo of Milo; it IS Milo for first-contact users. The decision to kill Storylane/Navattic (D69) and make the actual product the demo experience was validated by the funnel design. A dashboard-first experience would have required login before value delivery. The funnel delivers value (contract analysis) before asking for anything.

**Implication:** Every future vertical launch should lead with an interactive funnel, not a dashboard. The dashboard is for retained users.

### Theme 2: Middleware-as-routing-layer is powerful but brittle at the edges

Host-based routing via middleware (justmilo.app vs tlp.justmilo.app, same deploy) avoided a multi-deploy architecture. But middleware regex for public bypass broke on static assets (B2). The middleware layer is the single point of failure for every route decision.

**Implication:** Middleware changes need the same test discipline as database migrations. Curl every public path after changes. Path-prefix bypass over extension regex.

### Theme 3: AI model provider changes are silent breaking changes for primitives

Opus 4.7 deprecated `temperature` with no compile-time signal (B1). The error was a 400 that looked like a 500 because the error-handling branch didn't match the new error string. This will happen again with every model version change.

**Implication:** @milo/ai-client needs a model-quirk lookup table and full upstream error logging. Integration tests must cover latest-model x all-params x all-tiers. Unit tests are not enough.

### Theme 4: Stress-testing before building saves more time than it costs

The entity mapping table (B6) consumed ~4 directives of design work across D55, D57, D66 before D58's stress-test killed it. If the stress-test had run first, those 4 directives would have been available for other work. The same pattern played out positively with the v1 funnel — D60's 4-Coder stress-test surfaced the competitive positioning and dogfooding evidence that shaped D69's brand lock.

**Implication:** Default to stress-test before build for anything touching schema or architecture. The cost of a stress-test directive is 1 Coder for ~1 directive. The cost of building then reverting is 4+ directives across multiple Coders.

### Theme 5: Escalation hygiene determines Coder velocity

Coder-4 escalated the same Cloudflare DNS blocker three times (D48, D59, D67) with "Mark, please add the A record manually." Each escalation burned Mark's time on dev-team work and left Coder-4 idle waiting. Pattern 13 codified the rule: exhaust self-serve paths before escalating.

**Implication:** Every infra blocker reported as "Mark needs to do X" should be rejected by Strategy-Claude and redirected to self-serve. Mark's time is product and strategy, not dashboard clicks.

---

## 4. What to do differently next vertical launch

### Keep

- **4-Coder stress-test before vision lock.** D60 pattern: all 4 Coders argue against the plan from different angles (architecture, market, dogfooding, competitive). Surfaced issues that shaped the final design.
- **Brand positioning lock before any build directive.** D69 locked PPC-only positioning before Phase 1 scaffold started. Every build decision downstream was aligned.
- **Real sample data over synthetic.** D63 replaced D62's synthetic contracts with Mark's real Milo-authored PDFs. Authenticity > convenience.
- **Host-based routing in single deploy.** Avoided multi-project coordination overhead. Worth the middleware testing cost.

### Change

- **Run stress-test on every schema change BEFORE committing the amendment.** B6 committed SCHEMA.md v0.3.0 then reverted. Should have stress-tested consumer need first.
- **Include explicit screenshot requirements in EVERY UI directive from day 1.** B8 happened because the protocol didn't exist yet. Pattern 16 now exists — enforce it from directive #1 next time.
- **Middleware changes get curl verification in the same commit.** B2 would have been caught if the directive included "curl every static asset path, confirm 200 + correct Content-Type."

### Kill

- **Kill "Mark needs to do X manually" as a valid Coder report.** Pattern 13 codified this, but it should be a hard gate in the directive template itself. If a Coder's report says "escalate to Mark for infra," Strategy-Claude rejects it before it reaches Mark. No exceptions for DNS, env vars, or dashboard settings.

### Add

- **Model compatibility smoke test on every @milo/ai-client change.** Before any AI primitive ships, run a live call against every supported model with every parameter combination. B1 would have been caught by a 2-minute integration test.
- **Pre-launch static asset checklist.** Before any marketing or public page goes live: enumerate every static file served from public paths, curl each one, verify 200 + correct Content-Type. Template this so it's copy-paste for every launch.
- **Date-check protocol at session start.** Added to MASTER-PROMPT in D69. Prevents the date drift that caused all handoff docs to say 04-20 when the session had crossed into 04-21 PT.

---

## 5. Open questions for Mark

**Q1.** MOP extract chain fix (D69) is in flight. Once Coder-4 ships the Opus 4.7 temperature patch, who retests the full MOP extract flow? Should this be a Coder-1 verification directive or Mark's incognito retest?

**Q2.** DNS apex for justmilo.app — Coder-4 is attempting self-serve (D70). If Cloudflare token scope blocks it, do we expand the existing token or create a new zone-scoped token? Pattern 13 says exhaust self-serve, but there may be a security reason Mark scoped the token narrowly.

**Q3.** Demo tenant auto-provisioning (Coder-2 D46) — is the 24h TTL + hourly cron sweep the right cadence? For a PPC demo, users may want to return within a day to show their boss. Should TTL be 48h or 72h?

**Q4. LOCKED — Decision 82.** @milo/feedback consumer integration (Coder-1 D62) — the hero funnel collects feedback at step 10. Where does this feedback surface? Is there a Phase 2 feedback dashboard, or does it just write to the table for now?

**Q5.** The session produced 20 decisions (D53–D72). Several reference "Phase 2" as next. Is Phase 2 start gated on all in-flight work completing (D46, D62, D69, D70), or can Phase 2 directives begin while those are still running?

**Q6.** MOP signing redirect (spec written in D74): Mark needs to decide on the 6 open questions in the spec before redirect implementation can start. Is this Phase 2 work or should it run in parallel with current in-flight?
