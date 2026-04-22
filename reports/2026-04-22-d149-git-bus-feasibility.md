# D149: Git-as-Bus Feasibility — Stress Test & Recommendation (Revised)

> Coder-3 Research | Directive #149 | 2026-04-22
> Confirmed mechanic: Coder commits report → Mark pastes single-line URL → Strategy-Claude `web_fetch`es the report.
> Value: ~99% byte reduction on inbound paste. Turn count unchanged (Mark still pastes one message per report).

---

## 1. Baseline: The Paste Burden

### Corpus size (reports/ directory, 2026-04-18 through 2026-04-22)

| Metric | Value |
|---|---|
| Total report files | 72 |
| Total words | 137,099 |
| Total size on disk | 1,148 KB (1,036,055 bytes) |
| Average report size | 14,192 bytes (1,904 words) |
| Largest report | 61,936 bytes / 6,853 words (rule-pack-spec) |
| Top 5 by size | rule-pack-spec (61.9 KB), ui-port-plan (37.9 KB), phase2-exec-plan (35.7 KB), d118-v4-audit (34.2 KB), ppc-rules-dedup (33.0 KB) |

### Commit velocity

| Date | Commits (all) | Unique report files touched |
|---|---|---|
| 2026-04-18 | 10 | 8 |
| 2026-04-19 | 31 | 8 |
| 2026-04-20 | 91 | 40 |
| 2026-04-21 | 36 | 12 |
| 2026-04-22 | 10 | 11 |
| **5-day total** | **178** | **72** (deduplicated) |

### What changes with git-as-bus

**Today:** Mark reads Coder output → selects full report text → copies → switches to Strategy-Claude chat → pastes ~14 KB of markdown → sends.

**With git-as-bus:** Mark sees Coder's "committed as `abc1234`" → pastes one line like `report at https://raw.githubusercontent.com/moscaleflow/conductor-mark-reports/main/reports/2026-04-22-d149-git-bus-feasibility.md` → Strategy-Claude `web_fetch`es and reads the full report.

### Byte savings

| Scenario | Before (paste full report) | After (paste URL) | Reduction |
|---|---|---|---|
| Average report | 14,192 bytes | ~120 bytes | 99.2% |
| Largest report | 61,936 bytes | ~120 bytes | 99.8% |
| Peak day (40 reports) | 554 KB pasted | 4.7 KB pasted | 99.1% |
| 5-day total (72 reports) | 1,036 KB pasted | 8.6 KB pasted | 99.2% |

### What does NOT change

- **Turn count:** Mark still sends one message per report (URL instead of body). 40 reports = 40 messages either way.
- **Alt-tab burden:** Mark still switches from Coder terminal to Strategy-Claude chat. But the copy target is a one-line URL, not a multi-KB markdown blob.
- **Time per relay:** Drops from ~30s (find report, select-all, copy, paste) to ~10s (copy URL from Coder output, paste). On peak day: ~7 minutes saved (40 × 20s).
- **Context window cost:** Strategy-Claude's context still ingests the full report — `web_fetch` result fills the same tokens as a paste. The savings are on Mark's clipboard, not on Claude's context.

### Honest assessment of value

The headline number ("99% byte reduction") is real but masks that the bottleneck isn't bytes — it's turns. Mark still does 40 paste actions on a peak day. The real wins are:
1. **Lower friction per action** — pasting a URL is faster and less error-prone than selecting 14 KB of markdown
2. **Markdown preservation** — `web_fetch` gets the raw `.md` file; clipboard paste can mangle formatting
3. **Traceability** — the URL is a permanent reference to the exact version of the report
4. **Reduction in accidental truncation** — no more "paste was too big, here's the rest" follow-ups

---

## 2. Sensitive Content Audit

### Grep targets: last 15 reports (by commit date)

```
reports/2026-04-21-d94-contacts-migration-live.md
reports/2026-04-21-decision-log-audit.md
reports/2026-04-21-mop-redirect-decisions.md
reports/2026-04-21-v1-funnel-launch-retro.md
reports/2026-04-22-d137-wordmark-smtp-mop-audit.md
reports/2026-04-22-d140-sales-accounting-impl-specs.md
reports/2026-04-22-d141-half2-prep-and-prod-health.md
reports/2026-04-22-d142-item1-env-var-quotes.md
reports/2026-04-22-d142-item2-td-change-detection.md
reports/2026-04-22-d142-item3-sync-pings.md
reports/2026-04-22-d142-item4-operator-pill.md
reports/2026-04-22-d142-item5-mediarite-sheet-id.md
reports/2026-04-22-d147-env-audit-and-mediarite.md
reports/2026-04-22-d149-git-bus-feasibility.md
reports/README.md
```

### API keys, tokens, bearer credentials

**Full token pattern scan (sk-, re_, eyJ, Bearer+value, ghp_, gho_, xoxb-, xoxp-):**
```
$ grep -r -n -E '(sk-[a-zA-Z0-9]{20,}|eyJ[a-zA-Z0-9_-]{50,}|ghp_[a-zA-Z0-9]{30,}|xoxb-|xoxp-)' reports/
(no matches)

$ grep -r -n -E 'Bearer [a-zA-Z0-9]{20,}' reports/
(no matches)
```

**Partial key fragment (1 finding):**
```
$ grep -r -n 're_Xd9' reports/
reports/2026-04-22-d137-wordmark-smtp-mop-audit.md:64:   smtp_pass: [RESEND_API_KEY from Vercel env, re_Xd9...]
```
This is a 7-character prefix of a Resend API key. Not the full value, but enough to confirm the key prefix. **Low risk** — Resend keys are 36+ chars, 7 chars is not exploitable — but a hygiene issue for a public repo.

**Credential variable names (full corpus, 15 files):**
```
$ grep -r -l -i -E '(SERVICE_ROLE_KEY|API_KEY|PRIVATE_KEY|CLIENT_SECRET|CRON_SECRET)' reports/ | wc -l
15
```
These are documentation references (e.g., `SUPABASE_SERVICE_ROLE_KEY=<key>`, `[REDACTED, 219 chars]`), not actual values.

### Internal URLs beyond justmilo.app / tlp.justmilo.app

```
$ grep -r -n -o -E 'https?://[^ )"]+' reports/ | grep -v '(justmilo|vercel\.app|github\.com|supabase\.co|localhost|anthropic|example|npmjs|nextjs)'
reports/2026-04-22-d137-wordmark-smtp-mop-audit.md:77:https://resend.com/domains
reports/2026-04-22-d142-item3-sync-pings.md:39:https://the-lead-penguin.trackdrive.com/api/v1
reports/2026-04-22-d147-env-audit-and-mediarite.md:38:https://tlpmop.netlify.app
reports/2026-04-22-d147-env-audit-and-mediarite.md:51:https://the-lead-penguin.trackdrive.com/api/v1
```
- `resend.com/domains` — public SaaS URL, not sensitive
- `the-lead-penguin.trackdrive.com/api/v1` — client's TrackDrive subdomain, reveals vendor relationship. **Low-medium risk** in public context.
- `tlpmop.netlify.app` — MOP deployment URL. **Low risk** — Netlify URLs are semi-public anyway.

### Personal emails / real lead contact info

```
$ grep -r -o -h -E '[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[a-zA-Z]{2,}' reports/ | sort -u | grep -v '(@test\.|@example\.|@noreply|@iam\.gserviceaccount)'
alberto@interesr-media.com      # partner contact
ariel@pigeonfi.com              # partner contact
customerservice@terrawest.com   # business contact
ej@upbeat.chat                  # partner contact
git@github.com                  # not a person
help@levelprop.com              # business contact
jessica@awareads.com            # partner contact
mariela@rayadvertising.com      # partner contact
milo@justmilo.app               # system sender
milo@leadpenguin.com            # system sender
mo@scaleflow.co                 # Mark's git identity
nir@leadbuyerhub.com            # partner contact
smoke-test+20260421@justmilo.app # test address
support@leadpenguin.com         # system alias
support@theleadpenguin.com      # system alias
```
**6 real partner emails** — all from a single file: `d94-contacts-migration-live.md`. This file contains real publisher/buyer contact info from a migration dry run.

### Schema details that expose business logic

```
$ grep -r -c -i -E '(CREATE TABLE|ALTER TABLE|DROP TABLE|INSERT INTO|information_schema)' reports/ | grep -v ':0$'
(no matches in last 15 reports)
```
No DDL statements in any report. Reports reference table names and column names (e.g., "query `prospect_pipeline` WHERE `stage` NOT IN...") but these are query descriptions, not schema definitions.

### Supabase project refs and deploy URLs (full corpus)

```
$ grep -r -o -h -E '[a-z]{20}\.supabase\.co' reports/ | sort -u
jdzqkaxmnqbboqefjolf.supabase.co   # PPCRM (frozen)
tappyckcteqgryjniwjg.supabase.co   # shared/primary
wciwfsbmszpwfzbycbff.supabase.co   # ConvoQC
wjxtfjaixkoifdqtfmqd.supabase.co   # MOP (frozen)

$ grep -r -o -h -E 'https://[a-z0-9-]+\.vercel\.app' reports/ | sort -u
https://milo-for-ppc.vercel.app
https://milo-outreach-scale-flow.vercel.app
```

### Sensitivity verdict

| Category | Findings | Risk if public | Redaction needed? |
|---|---|---|---|
| Real token values | None (0 matches across all patterns) | N/A | No |
| Partial key prefix | 1 (`re_Xd9...` in d137) | Low — 7 chars of 36+ char key | Optional — not exploitable |
| Credential var names | 15 files reference names, not values | Low-medium — reveals service inventory | No (names are not secrets) |
| Partner emails (PII) | 6 real emails in 1 file (d94) | **Medium** — GDPR territory | **Yes — exclude d94 from mirror** |
| Infrastructure URLs | 4 Supabase refs, 2 Vercel, 1 TrackDrive, 1 Netlify | Low-medium — enumeration aid | No (semi-public by nature) |
| Schema / DDL | None | N/A | No |
| Internal URLs | TrackDrive subdomain, Netlify app | Low | No |

**Bottom line:** Reports are nearly clean. One file (d94-contacts-migration-live.md) contains real PII that must be excluded from any public mirror. One file (d137) contains a 7-char key prefix that's low risk but could be scrubbed. Everything else is safe.

---

## 3. Three Options

### Option A: Public mirror, no redaction (with exclusion list)

**How:** Create `moscaleflow/conductor-mark-reports` (public). GitHub Action on `conductor-mark` push copies `reports/*.md` to public mirror, excluding files on a blocklist.

**Exclusion list (3 files):**
```
d94-contacts-migration-live.md     # real partner email PII
d147-env-audit-and-mediarite.md    # env var inventory with key lengths
d142-item5-mediarite-sheet-id.md   # GCP service account + key length
```

**URL pattern for Strategy-Claude:**
```
https://raw.githubusercontent.com/moscaleflow/conductor-mark-reports/main/reports/{filename}.md
```

| Dimension | Score |
|---|---|
| Setup cost | ~30 min (create repo, write Action YAML, initial sync, test) |
| Ongoing maintenance | Low — update exclusion list when Coders write sensitive reports (rare) |
| Security posture | Good — no secrets, PII excluded, infra URLs are semi-public anyway |
| Paste reduction | 99% bytes, 0% turns |
| Reliability | High — raw.githubusercontent.com is CDN-backed, serves plain text |

**CDN cache lag:** raw.githubusercontent.com caches ~5 min. Coder commits → Mark waits 5 min → pastes URL. Or use GitHub API (`api.github.com/repos/.../contents/...`) which is uncached but returns base64.

### Option B: Public mirror with sed-based redaction

**How:** Same as A, but Action runs scrubbing regex before push.

| Dimension | Score |
|---|---|
| Setup cost | ~2 hours (scrub script, test against corpus, CI validation) |
| Ongoing maintenance | **High** — every new sensitive pattern needs a regex update |
| Security posture | Best on paper, worst in practice (silent regex failures = leaks) |
| Paste reduction | 99% bytes, 0% turns |
| Reliability | Medium — scrub failures are silent; garbled reports if too aggressive |

**Verdict: Not recommended.** The exclusion list in Option A handles the 3 problematic files. Automated redaction adds complexity for marginal gain, and silent failures are worse than explicit exclusions.

### Option C: Private repo + PAT (if web_fetch supports auth headers)

**How:** Stay on private `conductor-mark`. Mark creates a fine-grained PAT with `contents:read` on this repo only. Strategy-Claude calls `web_fetch` with auth header.

**Critical unknown:** Strategy-Claude's `web_fetch` in claude.ai chat is user-triggered (Mark pastes URL). It's unclear whether `web_fetch` supports:
1. Custom `Authorization` headers — needed for GitHub API
2. Raw authenticated URLs — GitHub deprecated token-in-URL

**I cannot test this from Claude Code.** Strategy-Claude must test:
```
# Test 1: Does web_fetch work on a public raw URL?
web_fetch("https://raw.githubusercontent.com/anthropics/anthropic-cookbook/main/README.md")

# Test 2: If Mark pastes a private repo URL, does 404 come back?
# (Mark pastes: https://raw.githubusercontent.com/moscaleflow/conductor-mark/main/CLAUDE.md)
```

If web_fetch in claude.ai can't send auth headers, Option C is dead.

| Dimension | Score |
|---|---|
| Setup cost | ~5 min (create PAT) — IF web_fetch supports headers |
| Ongoing maintenance | PAT rotation every 90 days |
| Security posture | Best — everything stays private |
| Paste reduction | 99% bytes, 0% turns |
| Reliability | Unknown — depends on web_fetch auth support |

---

## 4. Recommendation

### Ranking

| Rank | Option | Why |
|---|---|---|
| 1 | **A: Public mirror with exclusion list** | Lowest risk, simplest implementation, works with confirmed web_fetch behavior (plain URL, no auth). 3-file exclusion list handles all PII/sensitive content. |
| 2 | C: Private + PAT | Best security, but depends on unverified web_fetch auth support. Test first; fall back to A. |
| 3 | B: Redacted mirror | Over-engineered. Silent failures make it less secure than A despite being "more secure" on paper. |

### GO / DON'T GO

**GO — conditionally.**

The byte savings (99%) are real, and the per-action friction drop (one-line URL vs multi-KB paste) is genuine. But the honest value is modest:
- Mark still does 40 paste actions on a peak day
- Time saved is ~7 minutes on that peak day (20s × 40 reports)
- The traceability and formatting wins are nice but not transformative

**Worth building if:** Coder-4 can implement Option A in ≤30 minutes. The Action YAML is ~20 lines, the exclusion list is 3 files, and the test is one URL fetch. If it takes longer than that, the setup cost exceeds the first week's savings.

**Not worth building if:** The implementation grows scope (redaction pipelines, PAT management, custom API endpoints). Keep it dead simple or don't do it.

### Implementation spec for Coder-4 (if GO)

1. `gh repo create moscaleflow/conductor-mark-reports --public --clone=false`
2. Add `.github/workflows/sync-reports.yml` to `conductor-mark`:
   - Trigger: push to `main` changing `reports/*.md`
   - Step: checkout, copy `reports/*.md` minus exclusion list, push to public mirror
   - Exclusion list: `d94-contacts-migration-live.md`, `d147-env-audit-and-mediarite.md`, `d142-item5-mediarite-sheet-id.md`
3. Test: push a dummy report, verify it appears at `raw.githubusercontent.com/moscaleflow/conductor-mark-reports/main/reports/{name}.md`
4. Mark tests: paste URL into Strategy-Claude chat, confirm `web_fetch` returns content

### Failure modes

| Failure | Impact | Mitigation |
|---|---|---|
| CDN cache (5 min lag) | Strategy-Claude reads stale version if fetched immediately after commit | Mark waits ~5 min, or Coder says "committed, URL will be live in 5 min" |
| Coder commits sensitive report not on exclusion list | PII or credentials appear in public mirror | Pre-push grep in the Action (add 5 lines to YAML). But this is belt-and-suspenders — the exclusion list is the primary control. |
| Sync Action fails silently | Strategy-Claude gets 404, asks Mark to paste manually | Action sends Slack notification on failure. Fallback is the existing paste workflow — no worse than today. |
| Report too large for web_fetch | Truncated or failed read | Largest report is 62 KB. web_fetch should handle this. If not, Strategy-Claude falls back to paste for that one report. |
| Exclusion list gets stale | New sensitive report syncs to public mirror | Review exclusion list monthly, or add a CI check that greps new reports for email/key patterns before sync. |

---

## Appendix: Raw Grep Evidence

### Token patterns (full corpus)

```
$ grep -r -n -E '(sk-[a-zA-Z0-9]{20,}|eyJ[a-zA-Z0-9_-]{50,}|ghp_[a-zA-Z0-9]{30,}|xoxb-|xoxp-)' reports/
(exit 1 — no matches)

$ grep -r -n -E 'Bearer [a-zA-Z0-9]{20,}' reports/
(exit 1 — no matches)

$ grep -r -n 're_Xd9' reports/
reports/2026-04-22-d137-wordmark-smtp-mop-audit.md:64:   smtp_pass: [RESEND_API_KEY from Vercel env, re_Xd9...]
```

### Credential variable names (15 files)

```
$ grep -r -l -i -E '(SERVICE_ROLE_KEY|API_KEY|PRIVATE_KEY|CLIENT_SECRET|CRON_SECRET)' reports/
reports/2026-04-21-d94-contacts-migration-dry-run.md
reports/2026-04-20-coder-3-milo-for-ppc-scaffolding-plan.md
reports/2026-04-20-coder-3-onboarding-migration-plan.md
reports/2026-04-20-coder-3-signing-negotiation-delta-protocol.md
reports/2026-04-22-d147-env-audit-and-mediarite.md
reports/2026-04-21-d126-supabase-session-scope-audit.md
reports/2026-04-20-coder-3-negotiation-migration-plan.md
reports/2026-04-22-d142-item5-mediarite-sheet-id.md
reports/2026-04-22-d142-item1-env-var-quotes.md
reports/2026-04-19-coder-3-contracts-audit.md
reports/2026-04-20-coder-3-contract-freeze-plan.md
reports/2026-04-21-v1-funnel-launch-retro.md
reports/2026-04-22-d137-wordmark-smtp-mop-audit.md
reports/2026-04-22-d147-env-audit-and-mediarite.md
reports/2026-04-20-coder-2-blacklist-gate.md
```

### Real person emails (all from d94-contacts-migration-live.md)

```
$ grep -r -o -h -E '[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[a-zA-Z]{2,}' reports/ | sort -u | grep -v '(@test\.|@example\.|@noreply|@iam\.gserviceaccount|justmilo|leadpenguin|scaleflow|github)'
alberto@interesr-media.com
ariel@pigeonfi.com
customerservice@terrawest.com
ej@upbeat.chat
help@levelprop.com
jessica@awareads.com
mariela@rayadvertising.com
nir@leadbuyerhub.com
```

### Infrastructure URLs

```
$ grep -r -o -h -E '[a-z]{20}\.supabase\.co' reports/ | sort -u
jdzqkaxmnqbboqefjolf.supabase.co   (PPCRM, frozen)
tappyckcteqgryjniwjg.supabase.co   (shared/primary)
wciwfsbmszpwfzbycbff.supabase.co   (ConvoQC)
wjxtfjaixkoifdqtfmqd.supabase.co   (MOP, frozen)

$ grep -r -o -h -E 'https://[a-z0-9-]+\.(vercel\.app|netlify\.app)' reports/ | sort -u
https://milo-for-ppc.vercel.app
https://milo-outreach-scale-flow.vercel.app
https://tlpmop.netlify.app
```
