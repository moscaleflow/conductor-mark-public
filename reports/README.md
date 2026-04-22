# Reports

**Purpose:** Completion record for every directive. Each report matches a directive 1:1.

## Filename convention

Matches the directive filename. If directive is `2026-04-20-coder-1-extract-ai-client.md`, report is `reports/2026-04-20-coder-1-extract-ai-client.md`.

## Report structure

```markdown
# Report: [directive title]

**Coder:** Coder-N
**Directive:** link to directives/complete/[filename].md
**Status:** ✅ complete | ⚠️ partial | 🚫 blocked
**Started:** timestamp
**Completed:** timestamp

## What landed
[bullet-level summary — what actually shipped]

## Evidence
- Commit SHAs (list each)
- Test outputs (or link to CI run)
- Preview URLs if applicable
- Curl outputs, screenshots, etc.

## Issues encountered
[anything that went sideways, even if resolved]

## Open items
[anything not covered by this directive that surfaced during execution — candidate for next directive]

## Drift-check compliance
- [ ] All commits passed pre-commit hooks
- [ ] No --no-verify used
- [ ] Required docs updated where triggered
```

## Reviewing reports

Mark (or Strategy-Claude) reviews each report before the next directive in the same dependency chain fires. Specific patterns to watch for in reports:

- "Pre-existing issue" framing — this hides blockers as footnotes. Read everything labeled that way as a real issue until proven otherwise.
- Test results with asterisks or "expected" caveats — drill into the caveat before approving.
- Missing sections — a report without an "Issues encountered" section is usually hiding issues, not proving their absence.
- Curl outputs that show success codes but empty data — successful API call is not the same as successful business outcome.

## Reports are permanent

Never delete reports. Even failed/aborted directive reports stay in `reports/` — they're the audit trail. If a report needs correction, append a "## Correction" section with new date. Don't rewrite history.
