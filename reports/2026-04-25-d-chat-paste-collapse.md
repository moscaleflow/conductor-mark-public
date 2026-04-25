# Report: /ask Chat History — Long Pasted Text Collapses to Expandable Block

**Agent:** Coder-2 | **Date:** 2026-04-25 | **Status:** SHIPPED (842d52c page.tsx, pushed to main)

---

## Summary

Closes the UX gap where paste-bubble collapsed input text but the submitted message still dumped the full wall in chat history. User message bubbles now collapse each pasted block to a 150-char preview with "Show more/less" toggle. Multiple blocks expand/collapse independently. Typed input renders normally above collapsed blocks.

---

## Changes

### `src/app/ask/page.tsx` (single file — Pattern 23)

**Message interface extended:**
- `pastedBlocks?: string[]` — individual pasted text contents stored per message
- `typedPrefix?: string` — typed input text (rendered normally, not collapsed)

**`ExpandableText` component** (defined inline, before Header):
- Props: `text`, `threshold` (default 500), `previewLength` (default 150)
- Below threshold: renders as `<span>` with `white-space: pre-wrap`
- Above threshold (collapsed): first 150 chars + "..." + "Pasted text (N chars)" label + "Show more" button
- Expanded: full text with `pre-wrap` + "Show less" button
- Toggle style: purple accent (`rgba(139,92,246,0.7)`), no border/background

**`doSubmit` extended:**
- New optional params: `pastedBlocks?: string[]`, `typedPrefix?: string`
- Stored in user `Message` when present

**`handleSubmit` extended:**
- Passes `pastedTexts.map(p => p.content)` as `pastedBlocks` to `doSubmit`
- Passes `input.trim()` as `typedPrefix` when pasted blocks exist

**Message rendering (3 branches for user messages):**
1. `msg.pastedBlocks` exists → typed prefix (if any) + each block via `ExpandableText`
2. No `pastedBlocks` but `content.length > 500` → entire content via `ExpandableText`
3. Short content → plain text (unchanged)

---

## Verification

### Playwright (5/5 passing)

| Test | Result |
|------|--------|
| Long paste submitted → collapsed in chat bubble | PASS |
| Click Show more → full text expands inline | PASS |
| Click Show less → collapses back | PASS |
| Multiple pasted blocks → each independently expandable | PASS |
| Short message → renders normally, no collapse | PASS |

### Screenshot

Stitched to `/Users/markymark/Desktop/d-chat-paste-collapse.png`:
- Collapsed state with "Show more"
- Expanded state with full text + "Show less"
- Re-collapsed state
- Multiple blocks independently expandable

### Build

`npm run build` — clean, no warnings.

---

## Co-commit Note

Commit includes Coder-1's concurrent vetResults array refactor (Message.vetResult → Message.vetResults). Paste-collapse code is additive and independent.

---

## Commit

`842d52c` — `feat: /ask chat history — long pasted text collapses to expandable block with Show more/less`
