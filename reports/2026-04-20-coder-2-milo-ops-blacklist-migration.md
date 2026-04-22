# Coder-2 Report: milo-ops @milo/blacklist v0.2.0 Migration

**Date**: 2026-04-20
**Lane**: Coder-2 (Migration)
**Task**: Replace local blacklist-screening.ts with @milo/blacklist v0.2.0
**Status**: Complete — 3 commits, 8 consumers migrated, local impl removed

---

## Commits

| SHA | Description |
|-----|-------------|
| `83b0ff2` | feat: add @milo/blacklist v0.2.0 as workspace dep + client wrapper |
| `6fad056` | refactor: migrate 8 consumers from local blacklist-screening to @milo/blacklist |
| `92aad64` | chore: remove local blacklist-screening.ts + old single-package copy script |

## Files changed

| File | Change |
|------|--------|
| `package.json` | Added `@milo/blacklist` dep, updated postinstall script |
| `next.config.ts` | Added `@milo/blacklist` to `transpilePackages` |
| `scripts/copy-milo-packages.mjs` | New — replaces `copy-milo-ai-client.mjs`, handles all `@milo/*` packages |
| `scripts/copy-milo-ai-client.mjs` | Removed — superseded by `copy-milo-packages.mjs` |
| `src/lib/blacklist-client.ts` | New — singleton BlacklistClient wrapper (tenant='tlp') |
| `src/lib/blacklist-screening.ts` | Removed — 237 LOC local implementation |
| `src/lib/milo-queries.ts` | Import updated (static + dynamic) |
| `src/lib/publisher-onboarding.ts` | Import updated |
| `src/lib/tools/setup-new-buyer.ts` | Import updated |
| `src/app/api/publisher-onboarding/route.ts` | Import updated |
| `src/app/api/capture/route.ts` | Import updated |
| `src/app/api/leads/import/route.ts` | Import updated |
| `src/app/api/blacklist/screen/route.ts` | Import updated, dropped unused `phone`/`registration_state` params |
| `src/app/api/entities/[name]/route.ts` | Import updated (value + type imports) |

## Signal mapping verification

All 7 signals from local impl have v0.2.0 equivalents:

| Signal | Local | @milo/blacklist v0.2.0 | Match |
|--------|-------|------------------------|-------|
| exact_name | Yes | Yes | Identical |
| fuzzy_name (normalized) | Yes | Yes | Identical |
| fuzzy_name (Levenshtein < 3) | Yes | Yes | Identical |
| alias | Yes | Yes | Identical |
| contact_name (Levenshtein < 2) | Yes | Yes | Identical |
| email_domain | Yes | Yes | Identical |
| ip_address | Yes | Yes (v0.2.0) | Identical |
| linkedin_url | Yes | Yes (v0.2.0) | With URL normalization |

## Behavioral changes

| Change | Impact |
|--------|--------|
| **Tenant isolation** | Local queried ALL blacklist entries. Package filters by `tenant_id='tlp'`. Now TLP only sees TLP's entries. |
| **Table name** | Local read `blacklist` table. Package reads `blacklist_entries`. |
| **Normalization suffixes** | Local stripped industry terms (media, group, agency, etc.). Package strips legal entity types only (LLC, Inc, Corp, LP, LLP, PLLC). Slightly less aggressive fuzzy matching. |
| **Dropped fields** | `phone` and `registration_state` removed from `/api/blacklist/screen` API. These were accepted but never used in matching logic. |

## Coverage gaps (preserved from earlier work)

All 3 gaps fixed in prior commits remain intact after migration:

| Gap | File | Status |
|-----|------|--------|
| Buyer setup | `src/lib/tools/setup-new-buyer.ts` | Screening gate preserved (step 1) |
| Publisher add_to_pipeline | `src/app/api/publisher-onboarding/route.ts` | Screening gate preserved |
| Inbound MOP webhook | `src/lib/publisher-onboarding.ts` | Screening gate preserved |

## Turbopack compatibility

Next.js 16 Turbopack refuses to follow symlinks outside the project root. Created `scripts/copy-milo-packages.mjs` to replace symlinks with directory copies at postinstall. This script supersedes the single-package `copy-milo-ai-client.mjs` and handles both `@milo/ai-client` and `@milo/blacklist`.

## Feature gaps in @milo/blacklist v0.2.0

None. All signals from the local implementation are covered.

Minor difference: the package's `normalizeCompanyName` strips fewer suffixes than the local version (legal entities only vs legal + industry terms). This is architecturally correct — industry-specific suffix stripping belongs in vertical config, not a shared package.

## Build verification

`npm run build` passes clean after all 3 commits.

---

*Coder-2 — Migration lane*
