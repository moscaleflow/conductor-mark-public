# Coder-2 Report: MOP @milo/ai-client Audit

**Date**: 2026-04-19
**Lane**: Coder-2 (Migration)
**Task**: Task B — MOP @milo/ai-client migration
**Status**: Complete (nothing to do)

---

## Stale info source

The directive language ("Replace 9 files in MOP with direct @anthropic-ai/sdk imports") was based on an earlier audit state of MOP. Between the audit and this session, MOP's `src/lib/anthropic.ts` was already migrated to `@milo/ai-client` (callClaude, callClaudeVision, withFastInsight). The "9 files" count no longer reflects the codebase.

## Actual current state of MOP AI usage

### Already migrated to @milo/ai-client

| File | Functions used | Tier |
|------|---------------|------|
| `src/lib/anthropic.ts` | `callClaude`, `callClaudeVision`, `withFastInsight` | structured, judgment |
| `src/lib/models.ts` | `TIER` (re-exported as `MODELS`) | all tiers |

### Using @ai-sdk/anthropic (Vercel AI SDK provider) — NOT a migration target

| File | AI SDK function | Purpose | Model tier |
|------|----------------|---------|------------|
| `src/lib/milo-agent.ts` | `streamText` | Multi-step agent with 48 tools, streaming responses | `MODELS.judgment` |
| `src/lib/milo-tools.ts` | `anthropic.tools.webSearch_20250305()` | Native Anthropic web search tool | N/A (built-in) |
| `src/lib/knowledge-engine.ts` | `generateText` | Entity knowledge evaluation, anomaly detection | `MODELS.structured` |

`@ai-sdk/anthropic` is the Vercel AI SDK Anthropic provider. It enables streaming, tool calling, and multi-step agent loops via `streamText`/`generateText`. This is architecturally different from `@milo/ai-client`'s `callClaude()` (one-shot text completion). Replacing it would lose streaming and tool calling — a regression, not a migration.

### Out of scope

| File | SDK | Notes |
|------|-----|-------|
| `docs/trackdrive-mcp/server.js` | `@anthropic-ai/sdk` (direct, CommonJS) | Standalone docs utility, not production |

### package.json dependency

`@anthropic-ai/sdk` remains in `package.json` as:
- Transitive dependency via `@milo/ai-client`
- Required for `next.config.ts` `serverExternalPackages` resolution
- Removing risks build breakage — keep as-is

## Model policy compliance

All 3 Vercel AI SDK call-sites already reference `MODELS.judgment` or `MODELS.structured` from `src/lib/models.ts`, which re-exports `@milo/ai-client`'s `TIER` constants. Model policy is enforced centrally.

## Recommendation

Update MILO-ENGINE-PRIMITIVES.md v7 to reflect:
- MOP `@milo/ai-client` migration: **done** (not deferred)
- MOP Vercel AI SDK usage (`@ai-sdk/anthropic`): **correct architecture**, not a migration target
- Ping Coder-4 (Ops) to pick up the PRIMITIVES doc update in next ops cycle

---

*Coder-2 — Migration lane*
