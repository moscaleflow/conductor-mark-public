# Coder-2 Report: milo-ops @milo/ai-client Migration Audit

**Date**: 2026-04-20
**Lane**: Coder-2 (Migration)
**Task**: Migrate remaining direct @anthropic-ai/sdk imports to @milo/ai-client
**Status**: Complete — no actionable work found

---

## Files audited

| File | Reference | Verdict |
|------|-----------|---------|
| `src/lib/models.ts` | `export { TIER as MODELS } from '@milo/ai-client'` | Already migrated — single source of truth for TIER constants |
| `src/lib/anthropic.ts` | `@milo/ai-client` | Already migrated |
| `src/lib/milo-agent.ts` | `@ai-sdk/anthropic` (Vercel AI SDK) | Leave alone — `streamText` streaming primitive |
| `src/lib/milo-tools.ts` | `@ai-sdk/anthropic` (Vercel AI SDK) | Leave alone — tool-call streaming |
| `src/lib/knowledge-engine.ts` | `@ai-sdk/anthropic` (Vercel AI SDK) | Leave alone — `generateText` streaming |
| `next.config.ts` | `serverExternalPackages: ['@anthropic-ai/sdk']` | Infrastructure — required for transitive dep bundling |
| `package.json` | `"@anthropic-ai/sdk": "^0.78.0"` | Transitive dep of `@milo/ai-client`, needed for `serverExternalPackages` |
| `CLAUDE.md` | Mentions `@anthropic-ai/sdk` as "do not import directly" | Documentation — already correct |
| `docs/trackdrive-mcp/server.js` | `require('@anthropic-ai/sdk')` | Standalone MCP server, separate project — not in scope |

## Files migrated

None. All source files in `src/` already route through `@milo/ai-client`.

## Files explicitly left alone

- **`@ai-sdk/anthropic` (3 files)**: Vercel AI SDK provider for `streamText`/`generateText` — different primitive from `@milo/ai-client`. Per directive: don't touch.
- **`package.json` `@anthropic-ai/sdk` dep**: Transitive dependency of `@milo/ai-client`. Removing risks `serverExternalPackages` breakage.
- **`docs/trackdrive-mcp/`**: Standalone MCP server with its own `package.json`. Not part of milo-ops app.

## Final grep confirmation

```
$ grep -r "from ['\"]@anthropic-ai/sdk" src/
(no results)

$ grep -r "require.*@anthropic-ai/sdk" src/
(no results)
```

Zero direct SDK imports in `src/`. All AI calls route through `@milo/ai-client` or Vercel AI SDK (`@ai-sdk/anthropic`).

## TIER constants verification

`src/lib/models.ts` re-exports `TIER` and `TierName` from `@milo/ai-client`. No hardcoded `claude-*` model strings found anywhere in `src/`.

---

*Coder-2 — Migration lane*
