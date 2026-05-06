# conductor-mark-public

Read-only public mirror of Milo project directive reports. Synced automatically from the private `conductor-mark` repo via GitHub Action.

## Purpose

Strategy-Claude reads these reports via `web_fetch` from `raw.githubusercontent.com` URLs, replacing the copy-paste relay between Coders and Strategy-Claude.

## What's included

All files from `conductor-mark/reports/*.md` except those on the exclusion list.

## Exclusion list

These files are excluded from the mirror for privacy/security reasons:

- `2026-04-21-d94-contacts-migration-live.md` — contains real partner email PII
- `2026-04-22-d147-env-audit-and-mediarite.md` — env var inventory with key lengths
- `2026-04-22-d142-item5-mediarite-sheet-id.md` — GCP service account details

## Redaction

Lines matching Resend API key prefixes (`re_` followed by 7+ alphanumeric chars) are redacted during sync.

## URL pattern

```
https://raw.githubusercontent.com/moscaleflow/conductor-mark-public/main/reports/<filename>.md
```

## Last synced

<!-- LAST_SYNCED -->
Synced from `ca9f0aebdc7d50d48344b65c61a6ce07ae23ded4` at 2026-05-06T21:43:59Z
<!-- /LAST_SYNCED -->
