# D196 — /ask Markdown Rendering + Tightened Spacing

**Date:** 2026-04-24  
**Agent:** Coder-1  
**Scope:** react-markdown install, response renderer replacement, spacing overrides

---

## Summary

Replaced plain-text `whiteSpace: pre-wrap` response rendering with ReactMarkdown + remark-gfm. `**bold**` now renders as bold weight, `---` renders as thin divider lines, numbered/bullet lists render as proper HTML. Custom component map tightens all vertical spacing. Vet 🚩/✅ markers preserved.

---

## Commits

### milo-for-ppc

| Commit | Description |
|---|---|
| `25caf4b` | feat: D196 /ask markdown rendering — react-markdown + remark-gfm, tightened spacing |

---

## Packages

| Package | Version | Status |
|---|---|---|
| react-markdown | ^9.1.0 | **Newly installed** |
| remark-gfm | ^4.0.1 | **Newly installed** |

---

## Changes

### src/app/ask/page.tsx

**Imports added:**
- `ReactMarkdown` from `react-markdown`
- `remarkGfm` from `remark-gfm`
- `ComponentPropsWithoutRef` from `react`

**Response rendering:**
- Removed `whiteSpace: 'pre-wrap'` from response div
- Added `className="ask-response"` for test targeting
- Assistant messages with content render via `<ReactMarkdown remarkPlugins={[remarkGfm]}>`
- User messages and streaming placeholder (`...`) still render as plain text

**Custom component map (spacing overrides):**

| Element | Style |
|---|---|
| `p` | margin: 0.5em 0 |
| `ul`, `ol` | margin: 0.5em 0, padding-inline-start: 1.25em |
| `li` | margin: 0.15em 0 |
| `hr` | margin: 0.75em 0, 1px solid rgba(255,255,255,0.08) |
| `h1/h2/h3` | margin: 0.75em 0 0.4em, font-weight: 600 |
| `strong` | font-weight: 600 |
| `code` (inline) | background rgba(255,255,255,0.06), padding 1px 4px, radius 3px |

---

## Visual Verification

Stitched screenshot: `/Users/markymark/Desktop/d196-ask-markdown-render.png` (1280x3260)

| Panel | Content | Key rendering evidence |
|---|---|---|
| d196-01 | Vet: TestCo LLC | **GREEN FLAGS** ✅ renders bold + emoji, **CONFIDENCE RATING** bold, `---` as thin hr, **Suggested blacklist note:** bold |
| d196-02 | Buyer: dispute handling | **Buyer name / vertical**, **Billing type**, **Claimed reason** all bold. Nested list indented properly |
| d196-03 | Sales: ghosted buyer | **what vertical is this buyer in?** renders bold inline. Tight paragraph spacing |
| d196-04 | Publisher: ACA publishers | Bold section headers, bullet lists render as HTML lists |

### Before vs After

| Element | Before (plain text) | After (ReactMarkdown) |
|---|---|---|
| `**bold**` | Literal asterisks visible | Rendered as bold weight |
| `---` | Literal dashes visible | Thin 1px divider line |
| `1. 2. 3.` | Plain text with numbers | Proper `<ol>` list |
| `- item` | Plain text with dashes | Proper `<ul>` list with bullets |
| Paragraph spacing | Large gaps (pre-wrap) | Tighter 0.5em margins |

---

## Playwright Coverage

5/5 passing (final run)

| Test | Assertion |
|---|---|
| Vet — markdown renders | No literal `**`, RED/GREEN FLAGS sections present |
| Buyer — markdown renders clean | No literal `**` |
| Sales — no markdown artifacts | No literal `**` |
| Publisher — markdown renders clean | No literal `**` |
| Verify screenshots | 4/4 exist |

---

## Decision

Logged as **Decision 97** in DECISION-LOG.md.
