# D121 — Onboarding Re-fire + Routing Audit

**Directive**: D121  
**Lane**: Coder-4 Ops  
**Date**: 2026-04-21  
**Scope**: Read-only investigation — no code changes  

---

### Onboarding re-fire

**Symptom**: Returning users who have already completed onboarding get redirected back to `/onboard`.

**Root cause**: Middleware guard at `src/middleware.ts:122-124`:

```typescript
if (profile && profile.has_completed_onboarding === false) {
  return redirectWithCookies('/onboard')
}
```

This fires on every authenticated request where `has_completed_onboarding` is `false`. If the profile row in Supabase never gets its `has_completed_onboarding` column flipped to `true` at the end of the onboarding flow, the user is permanently trapped.

**Secondary factor**: The onboarding page (`src/app/onboard/page.tsx:48`) enforces a TEAM_NAMES whitelist:

```typescript
const TEAM_NAMES = ['mark', 'tiffani', 'fab', 'malvin', 'jen', 'kate', 'vee', 'cam', 'lee', 'jackie', 'kent'];
```

A user whose name is not in this list could complete the onboarding UI but fail the name validation gate, leaving `has_completed_onboarding` as `false` and creating a redirect loop.

**Proposed fix**: Two changes needed:
1. `src/app/onboard/page.tsx` — On successful onboarding completion, ensure `has_completed_onboarding` is set to `true` in the profiles table. Verify this write actually succeeds before navigating away.
2. `src/app/onboard/page.tsx` — Remove or expand the TEAM_NAMES whitelist (it blocks any user not in the hardcoded list from completing onboarding cleanly).

**Files**: `src/middleware.ts:122-124`, `src/app/onboard/page.tsx:48,263-267`

---

### /dashboard-v2 vs /operator

**Symptom**: All users land on `/dashboard-v2` after login, regardless of role. Operators should land on `/operator`.

**Root cause**: The middleware has correct role-based routing at `src/middleware.ts:151-164`:

```typescript
if (request.nextUrl.pathname === '/') {
  const landingPath = effectiveIsAdmin ? '/dashboard-v2' : '/operator'
  return redirectWithCookies(landingPath)
}
```

But this logic is **never reached** because two upstream files hardcode the post-login destination:

1. **Auth callback** (`src/app/auth/callback/page.tsx`) — Lines 39, 56, 68 all do:
   ```typescript
   window.location.href = '/dashboard-v2';
   ```
   This is a client-side redirect that bypasses middleware entirely.

2. **Login page** (`src/app/login/page.tsx`) — Line 70:
   ```typescript
   router.push('/dashboard-v2');
   ```
   Same bypass — client-side navigation straight to `/dashboard-v2`.

The middleware would work correctly if these pages redirected to `/` instead, letting middleware handle the role-based routing.

**Proposed fix**:
1. `src/app/auth/callback/page.tsx` — Change all three `window.location.href = '/dashboard-v2'` to `window.location.href = '/'`.
2. `src/app/login/page.tsx` — Change `router.push('/dashboard-v2')` to `router.push('/')`.

This lets the existing middleware logic handle role-based routing. No middleware changes needed.

**Recommended default post-login surface**: `/operator` for non-admin users, `/dashboard-v2` for admins — exactly what the middleware already implements. The middleware logic is correct; the problem is purely that it's being bypassed.

**Files**: `src/app/auth/callback/page.tsx:39,56,68`, `src/app/login/page.tsx:70`, `src/middleware.ts:151-164`

---

### Display name bug

**Symptom**: Users see "Operator" as their display name in the top bar instead of their actual name.

**Root cause**: Fallback chain in `src/components/Topbar.tsx:965`:

```typescript
const userName = profile?.name ?? serverName ?? (authLoading ? '' : 'Operator');
```

When `profile.name` is `null` (never set or not returned from the profiles query) **and** the `/api/walkthrough` endpoint doesn't return a `userName` field, the final fallback is the literal string `'Operator'`.

This happens when:
- The user skipped onboarding (name never written to profile)
- The profile was created via magic link auth without a name field
- The `/api/walkthrough` endpoint returns no `userName`

**Proposed fix**:
1. `src/components/Topbar.tsx` — Replace the `'Operator'` literal fallback with the user's email prefix (e.g., `user?.email?.split('@')[0] ?? 'Operator'`). This gives a reasonable display name even when profile.name is missing.
2. Longer term: ensure the onboarding flow always writes a name to the profile before setting `has_completed_onboarding = true`.

**Files**: `src/components/Topbar.tsx:965`

---

### Logout control

**Symptom**: There is no way for a user to log out of the application.

**Root cause**: The `signOut` function exists in `src/components/AuthProvider.tsx:111-116`:

```typescript
const signOut = useCallback(async () => {
  await supabase.auth.signOut();
  setUser(null);
  setProfile(null);
  router.push('/login');
}, [supabase, router]);
```

But it is **never exposed to any UI component**. No logout button, link, or menu item exists anywhere in the rendered application. The function is defined in the auth context but no component consumes it.

**Proposed fix**:
1. `src/components/Topbar.tsx` — Add a logout button/icon to the top bar. Import `useAuth` from AuthProvider and call `signOut()` on click.
2. Verify the AuthProvider's context value object actually includes `signOut` in its provided value (check the `<AuthContext.Provider value={...}>` props).

**Files**: `src/components/AuthProvider.tsx:111-116`, `src/components/Topbar.tsx`

---

### Recommended fix ordering

1. **Logout control** — Zero-risk addition, unblocks manual testing of the other fixes. No existing behavior changes.
2. **/dashboard-v2 vs /operator routing** — Two-line change (redirect to `/` instead of `/dashboard-v2`). Unlocks the correct middleware logic that already exists. High impact.
3. **Display name bug** — One-line fallback improvement in Topbar. Low risk.
4. **Onboarding re-fire** — Requires understanding the full onboarding write path and the TEAM_NAMES whitelist intent. Most investigation needed before fixing.

---

### Surprise findings

**Auth callback bypasses middleware entirely via client-side `window.location.href`** — the middleware's role-based routing logic is correct and complete but has never been reachable for post-login flows. Every user has been hardcoded to `/dashboard-v2` since the auth callback was written.
