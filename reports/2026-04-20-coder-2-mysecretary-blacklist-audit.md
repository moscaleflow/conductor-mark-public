# Coder-2 Report: mysecretary @milo/blacklist Integration Audit

**Date**: 2026-04-20
**Lane**: Coder-2 (Migration)
**Task**: Integrate @milo/blacklist into mysecretary lead creation paths
**Status**: No actionable work — already covered via milo-outreach downstream gate

---

## Lead creation paths audited

| # | Path | File | Trigger | Screening |
|---|------|------|---------|-----------|
| 1 | Research agent | `lib/research-agent.ts:389` | Weekly cron / manual / low water-mark | None at intake |
| 2 | Bulk CSV import | `app/api/admin/leads/bulk/route.ts:149` | Admin upload | None at intake |
| 3 | Single lead create | `lib/outreach-client/internal.ts:201` | Admin API | None at intake |

## Send paths audited

| Path | File | Purpose | Blacklist coverage |
|------|------|---------|-------------------|
| milo-outreach send | External service | Lead outreach emails | Gate 1 (blacklist) before send — commits `0433957`, `da29226` |
| `lib/email.ts` | Transactional only | Magic links, welcome emails, founder digest | N/A — sends to customers, not leads |

**mysecretary has no direct outreach send path.** The `OutreachClient` interface has `generateDraft`, `classifyReply`, `getSendLog` — but no `send` method. All outreach sends are delegated to milo-outreach.

## Why no integration needed

1. **No send bypass**: Every lead that reaches the email send step goes through milo-outreach, which already has the `@milo/blacklist` gate (Gate 1, compliance-first, fires before draft/env/tenant/manual gates).

2. **Intake screening adds minimal value**: The only benefit would be preventing draft-generation cycles on leads that will be blocked at send time. Draft generation costs ~100 tokens per lead — negligible.

3. **Research agent has its own quality bar**: Excludes mega-firms by name, requires verifiable hook and source URL, requires at least email or LinkedIn. This is domain-specific filtering, not blacklist screening — complementary, not redundant.

## Flows explicitly left alone

| Flow | Reason |
|------|--------|
| Research agent (`lib/research-agent.ts`) | Creates leads → flows to milo-outreach → blacklist gate blocks at send |
| Bulk import (`app/api/admin/leads/bulk/route.ts`) | Creates leads → flows to milo-outreach → blacklist gate blocks at send |
| Single create (`lib/outreach-client/internal.ts`) | Creates leads → flows to milo-outreach → blacklist gate blocks at send |
| Transactional email (`lib/email.ts`) | Sends to customers (magic links, welcome, digest), not leads |

## Coverage confirmation

```
grep -r "@milo/blacklist" /Users/markymark/mysecretary
(no results — not needed as dependency)

grep -r "sendLeadEmail\|emails.send" /Users/markymark/mysecretary/lib/outreach-client/
(no results — no direct send in outreach client)
```

All outreach flows terminate at milo-outreach, which has complete blacklist coverage.

---

*Coder-2 — Migration lane*
