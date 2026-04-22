# D126 — Supabase Session Cookie Domain Scope Audit

**Directive**: D126  
**Lane**: Coder-4 Ops  
**Date**: 2026-04-21  
**Scope**: Read-only — no code changes  

---

## Current cookie configuration

### Browser client (`src/lib/supabase-browser.ts`)

```typescript
client = createBrowserClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!
);
```

No explicit cookie options. `createBrowserClient` from `@supabase/ssr` uses internal defaults — no `cookieOptions`, no `domain`, no `path`, no `sameSite` overrides.

### Middleware client (`src/middleware.ts`)

```typescript
const supabase = createServerClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
  {
    cookies: {
      getAll() {
        return request.cookies.getAll()
      },
      setAll(cookiesToSet) {
        cookiesToSet.forEach(({ name, value }) => {
          request.cookies.set(name, value)
        })
        supabaseResponse = NextResponse.next({ request })
        cookiesToSet.forEach(({ name, value, options }) => {
          supabaseResponse.cookies.set(name, value, options)
        })
      },
    },
  }
)
```

Custom cookie handler forwards `options` from Supabase's internal cookie config. No explicit domain override applied.

### API route auth helper (`src/lib/auth-server.ts`)

```typescript
function createClientFromRequest(req: NextRequest) {
  return createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      cookies: {
        getAll() {
          return req.cookies.getAll();
        },
        setAll() {
          // no-op — API route handlers don't need to set cookies here
        },
      },
    },
  );
}
```

Read-only. `setAll` is a no-op.

### Admin client (`src/lib/supabase-admin.ts`)

```typescript
createClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.SUPABASE_SERVICE_ROLE_KEY!,
  { auth: { autoRefreshToken: false, persistSession: false } },
);
```

No cookies. Session persistence explicitly disabled. Not relevant to this audit.

### next.config.ts

No cookie-related headers or Supabase cookie config.

---

## Current scope: default (no explicit domain)

**No file in the codebase sets an explicit cookie `domain`.** The browser and middleware clients both rely on `@supabase/ssr` defaults.

When `@supabase/ssr`'s `createBrowserClient` sets cookies via `document.cookie`, it does **not** include a `Domain=` attribute unless explicitly configured via `cookieOptions`. Per RFC 6265 §4.1.2.3, when a `Set-Cookie` header (or `document.cookie` assignment) omits the `Domain` attribute, the browser scopes the cookie to the **exact origin hostname only** — no subdomain sharing.

This means:
- Cookies set on `tlp.justmilo.app` are scoped to `tlp.justmilo.app`
- They are **not** accessible from `justmilo.app` (marketing site)
- They are **not** accessible from any other `*.justmilo.app` subdomain
- The middleware's `setAll` forwards Supabase's cookie options to `response.cookies.set()`, which produces a `Set-Cookie` response header — same RFC rules apply

---

## Assessment

| Scope | Implication |
|---|---|
| **Current: `tlp.justmilo.app` (implicit, no explicit domain)** | Product auth is isolated. Marketing site (`justmilo.app`) has no auth session. Logging out on `tlp` has no effect on `justmilo.app`. Clean boundary. |
| **If `.justmilo.app` (shared)** | Session would persist across marketing + product. Logout on `tlp` kills session everywhere. Marketing hero/funnel would inherit operator auth cookies — unnecessary and potentially confusing. |
| **If `tlp.justmilo.app` (explicit)** | Same as current implicit behavior, but codified. Protects against future library changes. |

---

## Recommendation

**Current config is correct for the target architecture.**

- Operator auth → `tlp.justmilo.app`-scoped (already the case, implicitly)
- Marketing (`justmilo.app`) → no persistent auth needed (email capture is transactional per D71 voice-in-copy approach, no session required)

### Should we make it explicit?

Optional. Adding an explicit `domain` to the cookie config would make the boundary intentional rather than accidental:

```typescript
// src/lib/supabase-browser.ts — change
client = createBrowserClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!
);

// to
client = createBrowserClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
  {
    cookieOptions: {
      domain: 'tlp.justmilo.app',
      path: '/',
      sameSite: 'lax',
      secure: true,
    },
  }
);
```

And in the middleware (`src/middleware.ts`), the `setAll` handler would need to merge explicit options:

```typescript
setAll(cookiesToSet) {
  cookiesToSet.forEach(({ name, value }) => {
    request.cookies.set(name, value)
  })
  supabaseResponse = NextResponse.next({ request })
  cookiesToSet.forEach(({ name, value, options }) => {
    supabaseResponse.cookies.set(name, value, {
      ...options,
      domain: 'tlp.justmilo.app',
      path: '/',
      sameSite: 'lax',
      secure: true,
    })
  })
},
```

### Risk if made explicit

- **Session invalidation**: Setting an explicit `domain: 'tlp.justmilo.app'` will NOT invalidate existing sessions. The browser already associates the cookies with `tlp.justmilo.app` (implicit exact-host). Setting the explicit `domain` attribute creates a domain-scoped cookie that's accessible to `tlp.justmilo.app` and any sub-subdomains (e.g., `x.tlp.justmilo.app`) — which don't exist. No existing sessions break.
- **Hardcoded domain**: The `domain: 'tlp.justmilo.app'` string would break on localhost or Vercel preview deploys. If making this explicit, use an environment variable: `domain: process.env.COOKIE_DOMAIN || undefined`.
- **Complexity for no gain**: Since the current implicit behavior already achieves the correct scope, the explicit config adds complexity without changing behavior.

---

## Verdict

**No change needed.** The current default configuration already scopes auth cookies to `tlp.justmilo.app` by RFC 6265 behavior. The marketing site has no auth session. The boundary is clean.

Making it explicit would be a defensive hardening measure but introduces a hardcoded domain problem (breaks localhost/preview). Recommend leaving as-is unless the architecture adds new subdomains that should share auth (e.g., `admin.justmilo.app`).

---

## Mark approval flag

**No approval needed** — no code change proposed. Current config is already correct.

If Mark later decides to add explicit cookie domain scoping, that would be an auth behavior change requiring approval before shipping.
