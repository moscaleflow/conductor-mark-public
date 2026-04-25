# Vet Voice Audit + Plumbing Fixes

**Date:** 2026-04-24
**Agent:** Coder-1 (3 sub-agents via Pattern 23 fan-out)
**Status:** COMMITTED
**Source:** Coder-3 audit findings

## What shipped

Four files updated to apply voice audit findings: Vet prompt refined with signature closing, bookend intro, web_search suppression, and off-topic deflection. Analysis prompt gets Milo voice instruction + MEDIUM issue visibility. Two API routes migrated from raw Anthropic SDK to @milo/ai-client.

### Sub-agent fan-out (Pattern 23)

| Agent | Scope | Files | Changes |
|-------|-------|-------|---------|
| A — Vet voice refinements | vet.ts prompt edits | 1 file | +11/-2 |
| B — Analysis prompt voice + MEDIUM filter | route.ts | 1 file | +12/-1 |
| C — @milo/ai-client wireup | refine-redline + explain-further routes | 2 files | +25/-54 |

### Changes detail

**Sub-agent A — vet.ts:**
1. Signature closing: "would you sign this? Say so like you mean it" replaces "You're the closing voice"
2. Bookend intro: 1-line Milo intro before cards (e.g., "I ran CrossBlade's MSA through the engine...")
3. Web search suppression: "Do not web search in contract vet mode" — engine already analyzed the document
4. Off-topic deflection: "Not my department. I do vetting. Give me something in that lane."

**Sub-agent B — route.ts:**
1. Milo voice instruction appended to `analysisSystemPrompt`: "Write problem_explanation and plain_summary in a direct, operator-facing voice"
2. MEDIUM issues added as "ALSO FLAGGED" section in `formatAnalysisContext()` — titles only, after KEY ISSUES block

**Sub-agent C — refine-redline + explain-further:**
1. Both routes: `import Anthropic from '@anthropic-ai/sdk'` → `import { callClaude } from '@milo/ai-client'`
2. Removed API key environment checks (handled by @milo/ai-client)
3. Removed raw `sdk.messages.create()` → `callClaude(message, { tier: 'structured', maxTokens: 1024, system, cacheSystemPrompt: true })`
4. Removed TextBlock filtering boilerplate (callClaude returns string directly)

### Tool-reuse audit

| Primitive | Reused? | Notes |
|-----------|---------|-------|
| `@milo/ai-client` callClaude | Yes | Now used by refine-redline + explain-further (tier: structured) |
| Existing vet.ts prompt structure | Yes | Edits within existing sections |
| Existing route.ts formatAnalysisContext | Yes | Extended with MEDIUM filter block |
| **New primitives created** | **Zero** | |

### Raw Anthropic SDK elimination

| File | Before | After |
|------|--------|-------|
| refine-redline/route.ts | `new Anthropic()` + `sdk.messages.create()` | `callClaude()` from @milo/ai-client |
| explain-further/route.ts | `new Anthropic()` + `sdk.messages.create()` | `callClaude()` from @milo/ai-client |

Evidence: `grep -r 'new Anthropic\|@anthropic-ai/sdk' src/app/api/ask/refine-redline/ src/app/api/ask/explain-further/` returns zero matches.

## Files changed

| File | Action | Lines |
|------|--------|-------|
| `src/lib/ask/prompts/vet.ts` | Modified | +11/-2 |
| `src/app/api/ask/route.ts` | Modified | +12/-1 |
| `src/app/api/ask/refine-redline/route.ts` | Modified | +13/-26 |
| `src/app/api/ask/explain-further/route.ts` | Modified | +12/-28 |

## Commit

- `4373d9f` feat: Vet voice audit — signature line, MEDIUM filter, @milo/ai-client wireup

## Build

`npm run build` — clean, no errors, no warnings.

## Audit findings addressed

| Audit ID | Finding | Resolution |
|----------|---------|------------|
| HIGH-01 | Signature closing line weak | Replaced with "would you sign this? Say so like you mean it" |
| HIGH-02 | Missing bookend intro | Added 1-line intro instruction to CONTRACT VET MODE |
| HIGH-03 | Off-topic deflection missing | Added OFF-TOPIC HANDLING section |
| HIGH-04 | Web search in contract vet mode | Added "Do not web search in contract vet mode" instruction |
| HIGH-05 | Analysis voice bland | Added Milo voice instruction to analysisSystemPrompt |
| MED-01 | MEDIUM issues invisible in verdict | Added ALSO FLAGGED section in formatAnalysisContext |
| PLUMB-01 | refine-redline uses raw SDK | Replaced with @milo/ai-client callClaude |
| PLUMB-02 | explain-further uses raw SDK | Replaced with @milo/ai-client callClaude |
