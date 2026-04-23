# V4.1: Rename /api/dashboard-v2 → /api/operator/executive

> Coder-3 stub | D164b | Prerequisite: V4.0 spec 06 ships first

## What

Rename the API route from `/api/dashboard-v2` to `/api/operator/executive`. The route stays functionally identical — only the path changes.

## Why

After spec 06 deletes the `/dashboard-v2` page route, the API route name is misleading. It serves admin executive data consumed by `/operator/page.tsx` (line 172). The name should reflect its actual consumer.

## Scope

| File | Change |
|---|---|
| `src/app/api/dashboard-v2/route.ts` (2,066 LOC) | Move to `src/app/api/operator/executive/route.ts` |
| `src/app/operator/page.tsx` | Line 172: update fetch URL |
| Comments in ~5 files | Update references to the old API path |

## Rough LOC

~5 LOC changed (1 fetch URL + comments). 0 logic changes. The 2,066-line route file moves to a new path.

## Dependencies

- Spec 06 must ship first
- If any external system hits `/api/dashboard-v2` directly, add a redirect in `next.config` (same pattern as the page redirect)
