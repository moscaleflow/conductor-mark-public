# V4 Shell Refinement: Decommission /dashboard-v2

> Coder-3 Spec | D152 (updated D156) | For Coder-1 execution
> Source: D85 amendment — "/dashboard-v2 decommission + 'Full dashboard →' link removal"
> Cross-reference: D153 reference audit (af97218) — Coder-1 found 50 references, 7 hard dependencies
> Build order: 1 of 6 (ship first — clears tech debt before building new features)

---

## User-observable behavior

**Before:** Two landing pages exist:
- `/dashboard-v2` — admin landing (2,237-line page with 20+ dashboard blocks)
- `/operator` — operator landing (V4 shell: briefing + pills + drawer)

Admins land on `/dashboard-v2` via middleware redirect (line 153). The operator page already renders the admin executive view (Top5Frame) when `isAdmin === true`. Both pages call `/api/dashboard-v2?view=executive` for admin data.

**After:**
- `/dashboard-v2` route is removed
- All users (admin + operator) land on `/operator`
- Middleware redirect sends `/` → `/operator` for everyone
- Old `/dashboard-v2` URLs redirect to `/operator` (301)
- All "Back to dashboard" links point to `/operator`
- The `/api/dashboard-v2` route survives (renamed comment, kept as-is) because `/operator` page.tsx line 172 fetches it for admin executive data

---

## Inventory of references to update

### Direct route references (must change)

| File | Line | Current | Change to |
|---|---|---|---|
| `src/middleware.ts` | 151-153 | `effectiveIsAdmin ? '/dashboard-v2' : '/operator'` | `'/operator'` for all users |
| `src/middleware.ts` | 157-165 | Redirect admin+impersonating from `/dashboard-v2` to `/operator` | Remove this block (no longer needed) |
| `src/components/BackLink.tsx` | 10 | `href = '/dashboard-v2'` | `href = '/operator'` |
| `src/components/SupportPanel.tsx` | 15 | `'dashboard': '/dashboard-v2'` | `'dashboard': '/operator'` |
| `src/components/ImpersonationBanner.tsx` | 33 | `window.location.href = '/dashboard-v2'` | `window.location.href = '/operator'` |
| `src/components/Topbar.tsx` | 947 | `window.location.href = '/dashboard-v2'` | `window.location.href = '/operator'` |
| `src/app/contract-review/[id]/page.tsx` | 618 | `router.push('/dashboard-v2')` | `router.push('/operator')` |
| `src/app/contract-review/[id]/page.tsx` | 668 | `router.push('/dashboard-v2')` | `router.push('/operator')` |
| `src/app/contract-review/[id]/page.tsx` | 1024 | `router.push('/dashboard-v2')` | `router.push('/operator')` |
| `src/app/setup/page.tsx` | 72 | `router.push('/dashboard-v2')` | `router.push('/operator')` |
| `src/lib/team-roster.ts` | 40 | Comment: "Admins land on /dashboard-v2" | Update comment |
| `src/components/admin/AdminSkillBar.tsx` | 45-46 | `pathname === '/dashboard-v2'` check | Remove `/dashboard-v2` branch, keep `/operator` |

### Comment-only references (update text, no behavior change)

| File | Lines | Action |
|---|---|---|
| `src/app/operator/page.tsx` | 27, 126, 360 | Update comments that reference `/dashboard-v2` |
| `src/components/AdminExecutiveCards.tsx` | 7, 12 | Update comment |
| `src/components/OperatorTour.tsx` | 8 | Update comment |
| `src/app/api/briefing/route.ts` | 1172 | Update comment |
| `src/app/api/operator/pill/route.ts` | 759, 767 | Update comments |
| `src/app/observatory/page.tsx` | 28 | Update comment |
| `src/app/api/walkthrough/route.ts` | 7 | Update comment |
| `src/app/api/profile/complete-operator-tour/route.ts` | 40 | Update comment |
| `src/lib/impersonation-state.ts` | 8 | Update comment |
| `src/lib/greetings.ts` | 4 | Update comment |
| `src/lib/executive-ranker.ts` | 12 | Update comment |
| `src/lib/operator-pills.ts` | 9, 60, 145 | Update comments. Note: line 9 imports `RoleName` from `dashboard-v2-types` — this import stays (type file is not being deleted in v4.0) |
| `src/components/admin/AddSkillModal.tsx` | 29 | Update comment |
| `src/components/Topbar.tsx` | 812, 923 | Update comments |

### Cross-component imports from dashboard-v2 (keep, do not delete)

Per D153 audit (af97218) — these import reusable components/types that live inside the dashboard-v2 folder. They depend on the folder existing, not the route being accessible.

| File | Import | Why keep |
|---|---|---|
| `src/app/pipeline-board/page.tsx` | `EntityDetailPanel` from `dashboard-v2/` | Used by pipeline-board, not by dashboard-v2 page |
| `src/app/pipeline/page.tsx` | `KanbanBlock`, `EntityDetailPanel` from `dashboard-v2/` | Used by pipeline page |
| `src/components/ConversationModal.tsx` | `EntityDetailPanel`, `ContractAnalysisDisplay` from `dashboard-v2/` | Used by conversation modal for inline entity/contract views |
| `src/components/OnboardingTour.tsx` | `BlockId` type from `dashboard-v2-types` | Tour step targeting |
| `src/app/api/walkthrough/route.ts` | `ROLE_BLOCKS`, `USER_BLOCKS`, `BLOCK_TITLES`, `BlockId`, `RoleName` from `dashboard-v2-types` | Walkthrough API logic |
| `src/lib/operator-pills.ts` | `RoleName` from `dashboard-v2-types` | Type definition still needed |

**Relocation plan (v4.1):** Move `EntityDetailPanel`, `KanbanBlock`, `ContractAnalysisDisplay` to `src/components/shared/`. Move `dashboard-v2-types.ts` to `src/lib/types.ts` or split into domain-specific type files. Not in v4.0 scope — the folder stays, only the page route is deleted.

### Test file references (update)

D153 found test assertions that hardcode `/dashboard-v2`:

| File | Lines | Action |
|---|---|---|
| `tests/role-qa.spec.ts` | 316, 345 | Update admin route assertion to `/operator` |
| `scripts/e2e/auth-flow-2026-04-21.ts` | 92, 99, 102, 164 | Update auth flow assertions to `/operator` |
| `tests/screenshots/d146-api-*.json` | various | Update hardcoded test result URLs |

### Files to delete

| Path | LOC | Notes |
|---|---|---|
| `src/app/dashboard-v2/page.tsx` | 2,237 | The page itself. Admin view now lives on /operator. |

### Files to NOT delete (v4.0)

| Path | Reason |
|---|---|
| `src/app/api/dashboard-v2/route.ts` (2,066 LOC) | `/operator` page.tsx line 172 fetches this for admin executive data. Rename to `/api/operator/executive` in v4.1. |
| `src/components/dashboard-v2/*.tsx` (32 files, 12,909 LOC) | `pipeline-board` and `pipeline` pages import EntityDetailPanel and KanbanBlock. Extract shared components in v4.1. |
| `src/lib/dashboard-v2-types.ts` (412 LOC) | `RoleName` type imported by `operator-pills.ts`. Rename in v4.1. |

### Redirect for old URLs

Add a redirect rule so bookmarked `/dashboard-v2` URLs don't 404:

**Option A (middleware):** Add a block in `middleware.ts`:
```typescript
if (request.nextUrl.pathname === '/dashboard-v2') {
  return NextResponse.redirect(new URL('/operator', request.url), 301);
}
```

**Option B (next.config.js redirect):**
```javascript
redirects: async () => [
  { source: '/dashboard-v2', destination: '/operator', permanent: true },
]
```

**Recommended: Option B** — it's declarative, doesn't add middleware complexity, and works at the edge.

---

## Acceptance criteria

- `/` redirects to `/operator` for ALL users (admin and non-admin)
- `/dashboard-v2` returns 301 redirect to `/operator`
- Admin users see Top5Frame executive view on `/operator` (already works — no change needed)
- Non-admin users see briefing + pills + drawer on `/operator` (already works — no change needed)
- "Back" links from contract-review, setup, reconciliation, pipeline-board, pipeline, invoices, disputes all navigate to `/operator`
- "Stop impersonating" in ImpersonationBanner and Topbar navigate to `/operator`
- `npm run build` passes clean
- Test assertions in `role-qa.spec.ts` and `auth-flow-2026-04-21.ts` pass with `/operator` routes
- No 404s on any page that previously linked to `/dashboard-v2`

---

## File-level scope

| File | Change type | ~LOC |
|---|---|---|
| `src/middleware.ts` | Edit: simplify redirect logic | -10 |
| `src/components/BackLink.tsx` | Edit: change default href | 1 |
| `src/components/SupportPanel.tsx` | Edit: change path | 1 |
| `src/components/ImpersonationBanner.tsx` | Edit: change href | 1 |
| `src/components/Topbar.tsx` | Edit: change href + update comments | 3 |
| `src/app/contract-review/[id]/page.tsx` | Edit: 3 router.push changes | 3 |
| `src/app/setup/page.tsx` | Edit: change router.push | 1 |
| `src/components/admin/AdminSkillBar.tsx` | Edit: remove dashboard-v2 pathname check | -2 |
| `src/app/dashboard-v2/page.tsx` | **Delete** | -2,237 |
| `next.config.js` or `next.config.mjs` | Add redirect rule | 3 |
| `tests/role-qa.spec.ts` | Edit: update admin route assertions | 2 |
| `scripts/e2e/auth-flow-2026-04-21.ts` | Edit: update 4 auth flow assertions | 4 |
| ~14 files | Comment updates only | ~14 |

**Net: -2,214 LOC. 14 files edited, 1 file deleted, ~14 comment-only updates.**
**D153 cross-check: all 7 hard dependencies and 3 soft dependencies from af97218 are covered above.**

---

## Data dependencies

None. No schema changes. No API changes. The `/api/dashboard-v2` route stays.

---

## Visual requirements

None — this is a route deletion + redirect. No new UI.

---

## Blast radius

**High surface area, low risk.** Many files reference `/dashboard-v2` but every change is a simple string replacement. The operator page already handles admin users (Top5Frame executive view) so removing the dashboard-v2 page doesn't remove any functionality — it removes a duplicate path to the same data.

**Risk 1:** Admin users who have bookmarked `/dashboard-v2` hit a redirect. 301 redirect handles this transparently.

**Risk 2:** External integrations or monitoring that hit `/dashboard-v2` directly. **Mitigation:** Check Vercel analytics for `/dashboard-v2` traffic before deletion. The redirect handles this, but it's worth knowing the traffic volume.

**Risk 3:** The `/api/dashboard-v2` route name is confusing after the page is gone. **Mitigation:** This is tech debt, not a risk. Rename to `/api/operator/executive` in v4.1 (requires updating operator/page.tsx line 172).

---

## Build order rationale

**Ranked 1 of 6 — ship first** because:
- Pure cleanup — removes 2,237 LOC of dead code
- Unblocks all other refinements by establishing `/operator` as the sole landing page
- Every reference change is a trivial string replacement (low risk of bugs)
- The redirect ensures zero broken bookmarks
- Must ship before any V4 shell refinements so that Coder-1 isn't building features for a page that's about to be deleted
- `npm run build` will confirm nothing breaks

---

## Open questions

None. All decisions are derivable from D85 + D118 + current code state.
