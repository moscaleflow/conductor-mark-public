# D180 — Schema Migration + Push + Post-Deploy Smoke

**Date:** 2026-04-22  
**Agent:** Coder-1  
**Scope:** Ship hidden_pills schema, fix deploy blocker, push both repos, verify production

---

## Summary

1. Schema doc amended + migration file created
2. Migration applied via `supabase db push` (also picked up Coder-3's pill_config migration)
3. Fixed deploy blocker: missing ContractClauseList.tsx + /api/operator/clauses not committed
4. Pushed both repos, Vercel deploy succeeded
5. Post-deploy smoke test passed (Playwright + screenshots)

---

## Commits

### milo-for-ppc (8 total pushed, 2 new this directive)

| Commit | Description |
|---|---|
| `987b705` | schema: amend user_profiles base schema + add 20260422400001_hidden_pills.sql |
| `66e4638` | fix: add missing ContractClauseList.tsx + /api/operator/clauses route |

### conductor-mark (2 total pushed, 1 existing + 1 new)

| Commit | Description |
|---|---|
| `c8a0324` | reports: D177 + D173 |
| (this report) | D180 report |

---

## Schema migration

Two migrations applied via `supabase db push --linked`:

1. `20260422300001_pill_config.sql` — Coder-3 D176 (was pending)
2. `20260422400001_hidden_pills.sql` — D173b jiggle mode

Column verified live:
```
curl .../user_profiles?select=hidden_pills&limit=1
→ [{"hidden_pills":[]}]
```

Base schema file `20260327100001_user_profiles.sql` amended with all columns added since initial creation (custom_pills, focus_areas, hidden_pills, is_test_user, has_completed_operator_tour, has_claimed_welcome, pill_config).

---

## Deploy blocker fix

**Root cause:** PillDrawer.tsx imports `ContractClauseList` (added by linter during D177 session), but the component file and its backing API route were never git-added. Local build passed because the files existed on disk; Vercel build failed because they weren't in git.

**Fix:** Committed both files as `66e4638`, pushed, Vercel redeploy succeeded.

---

## Post-deploy verification

### API tests (Playwright, 3/3 pass)

| Test | Result |
|---|---|
| accounting disputes tab → action_pill_id=disputes | PASS (1 linked item) |
| contracts_review pending_signature → action_pill_id=esign | PASS (0 items, vacuous) |
| aging overdue → action_pill_id=disputes | PASS (3 linked items) |

### UI smoke (Playwright, 1/1 pass)

| Check | Result |
|---|---|
| /operator loads | PASS |
| PillBar renders with pills + badges | PASS |
| `+ Add pill` opens picker overlay | PASS |
| Picker shows YOUR PILLS / OTHER ROLES / Create custom | PASS |
| Mic icon visible in SearchBar | PASS |

### Screenshots (on Desktop)

- `d180-smoke-operator.png` — operator page with pill bar, badges, mic icon, `+ Add pill`
- `d180-smoke-picker.png` — picker overlay with 5 active pills (green dots), OTHER ROLES collapsed, Create custom pill link

No visual issues found. No Pattern 16 screenshots required.

---

## Remaining items for visual review (Coder-4 D165)

- Jiggle mode (long-press) — cannot test via headless Playwright (requires 500ms hold interaction)
- Hide pill persistence — requires jiggle mode entry
- Cross-pill drawer switch animation — requires drawer open + tap on linked item
