---
directive: "Signing migration delta audit — unfreeze window"
lane: research
coder: Coder-3
started: 2026-04-20 ~07:00 MDT
completed: 2026-04-20 ~07:15 MDT
supplements: reports/2026-04-20-coder-3-signing-migration-audit.md (commit a79fcf0)
---

# Signing Documents Migration Delta Addendum — 2026-04-20

## 1. Situation

The signing migration audit (commit a79fcf0) planned for **109 rows** based on the MOP freeze manifest (2026-04-19 ~14:50 MDT).

However, between:
- **Decision 45** (2026-04-20 ~14:47 UTC): MOP global unfreeze (`MOP_WRITES_FROZEN=false`)
- **Decision 52** (2026-04-20): Contract-writes freeze code deployed but **NOT ACTIVATED** (`CONTRACT_WRITES_FROZEN=false`)

...all signing write endpoints have been live. The unfreeze window is **still open**.

## 2. What Could Have Written

| Endpoint | Could Fire? | Why | Impact |
|----------|-------------|-----|--------|
| `/api/sign/[token]/submit` | **YES** | Public token access, no auth gate | Status changes: pending/viewed → signed |
| `/api/sign/[token]/decline` | **YES** | Public token access | Status changes: pending/viewed → declined |
| `/api/sign/[token]/capture-view` | **YES** | Public, fires on page load | Updates: viewed_at, status → 'viewed' |
| `/api/bot/documents/generate` | **NO** | Requires x-api-key from PPCRM, which is frozen | No new rows |
| `/api/bot/documents/[id]/void` | **NO** | Requires x-api-key from PPCRM | No status changes |
| `/api/bot/webhooks/refire` | **NO** | Requires x-api-key | No audit_trail updates |

### Key Constraint: PPCRM Is Frozen

PPCRM's freeze prevents the onboardbot from calling MOP's bot API. This means:
- **No new signing_documents rows** can be created (only PPCRM triggers document generation)
- **Only status transitions on existing rows** are possible (human signers accessing their token URLs)

## 3. Possible Delta Scenarios

### Scenario A: No Human Signers Accessed Tokens (Most Likely)

The 109 rows were in various states at freeze time. If no one visited a signing URL between 2026-04-20 14:47 UTC and now:
- Row count: still 109
- No status changes
- Migration proceeds as planned

### Scenario B: One or More Signers Viewed/Signed

If a counterparty had an active signing token and happened to access it:
- **View:** `viewed_at` set, `status` changes from 'pending' → 'viewed', audit_trail appended
- **Sign:** `signed_at` set, `status` → 'signed', `signature_data` populated, `audit_trail` appended, webhook fired to PPCRM (blocked — PPCRM frozen)

**Impact on migration:** The 109-row baseline snapshot (from `full-backup.sql`) is now stale for those rows. Migration would copy old state while production has new state.

### Scenario C: A Signer Declined

- `status` → 'declined', `declined_at` set, `decline_reason` populated, webhook fired (blocked)
- Same staleness issue as Scenario B

## 4. Risk Assessment

| Factor | Assessment |
|--------|-----------|
| Window duration | **Still open** (hours to days depending on when freeze activates) |
| Probability of writes | **LOW-MEDIUM** — depends on active token count at freeze time |
| Maximum possible affected rows | Limited to rows with `status IN ('pending', 'viewed')` at freeze |
| Can new rows be created? | **NO** — PPCRM frozen, can't call generate endpoint |
| Data loss risk | LOW — production has NEWER data, not lost data |

## 5. Detection Strategy

When CONTRACT_WRITES_FROZEN is activated (or before migration fires), run this aggregate-only query on MOP production:

```sql
-- Delta detection (no PII, aggregate only)
SELECT
  COUNT(*) AS total_rows,
  COUNT(*) FILTER (WHERE created_at > '2026-04-19T20:50:00Z') AS new_rows_since_freeze,
  COUNT(*) FILTER (WHERE signed_at > '2026-04-20T14:47:59Z') AS signed_during_window,
  COUNT(*) FILTER (WHERE viewed_at > '2026-04-20T14:47:59Z') AS viewed_during_window,
  COUNT(*) FILTER (WHERE status = 'declined' AND updated_at > '2026-04-20T14:47:59Z') AS declined_during_window
FROM signing_documents;
```

**Expected results:**
- `total_rows`: 109 (no new rows possible — PPCRM frozen)
- `new_rows_since_freeze`: 0
- `signed_during_window`: 0-3 (low probability)
- `viewed_during_window`: 0-5 (slightly higher — page loads are casual)
- `declined_during_window`: 0-1 (very rare)

## 6. Impact on 109-Row Migration Plan

### If Delta = 0 (No Writes)

Migration proceeds exactly as specified in commit a79fcf0. No changes needed.

### If Delta > 0 (Some Status Transitions)

**The migration still works correctly** because:

1. **The migration copies from the frozen backup** (`full-backup.sql` from 2026-04-19), not from live. This means:
   - Migrated data reflects the freeze-time snapshot
   - Production data is AHEAD of the migrated copy

2. **After migration + freeze activation:** Production writes stop. The delta rows have NEWER state in MOP than in shared Supabase.

3. **Reconciliation step needed:** After the initial migration, run a delta sync:
   ```sql
   -- For each row where MOP production state > backup state:
   UPDATE signing_documents_shared
   SET status = mop_live.status,
       signed_at = mop_live.signed_at,
       viewed_at = mop_live.viewed_at,
       signature_data = mop_live.signature_data,
       audit_trail = mop_live.audit_trail,
       -- ... other mutable fields
   WHERE id = mop_live.id
     AND (signing_documents_shared.updated_at < mop_live.updated_at);
   ```

4. **Alternative (simpler):** Delay migration until AFTER `CONTRACT_WRITES_FROZEN=true` is activated. Then the backup IS the final state and no delta exists.

## 7. Recommended Approach

```
┌─────────────────────────────────────────┐
│ Step 1: Activate CONTRACT_WRITES_FROZEN │
│   → Set true in Netlify, redeploy       │
│   → Signing endpoints now return 503    │
└────────────────┬────────────────────────┘
                 │
┌────────────────▼────────────────────────┐
│ Step 2: Run delta detection query       │
│   → Count: signed/viewed/declined       │
│     since 2026-04-20 14:47 UTC          │
└────────────────┬────────────────────────┘
                 │
         ┌───────┴───────┐
         │               │
    Delta = 0       Delta > 0
         │               │
┌────────▼──────┐  ┌─────▼──────────────────┐
│ Migrate from  │  │ Take FRESH backup of    │
│ existing      │  │ signing_documents NOW   │
│ backup (109)  │  │ (post-freeze, includes  │
│               │  │ delta). Migrate from    │
│               │  │ THIS snapshot instead   │
└───────────────┘  └─────────────────────────┘
```

**Why this works:** Once the contract-writes freeze is active, no more changes can occur. A fresh snapshot at that moment is the authoritative final state. Whether it's 109 rows or 110 rows, the migration SQL from commit a79fcf0 handles it identically (same column mapping, same transforms).

## 8. Row Count Update

| Scenario | Rows | Column Mapping Change? | Migration SQL Change? |
|----------|------|----------------------|---------------------|
| No delta | 109 | No | No |
| 1-3 status transitions | 109 (same rows, different state) | No | No |
| (Impossible) New rows | N/A — PPCRM frozen, can't generate | — | — |

**The signing migration audit (commit a79fcf0) remains valid regardless of delta.** The column mapping is the same whether a row has `status='pending'` or `status='signed'`. The only difference is which values populate the mapped columns.

## 9. Open Questions for Mark

1. **Activate freeze NOW?** The longer CONTRACT_WRITES_FROZEN stays inactive, the wider the potential delta window. If no active signing sessions need to complete, activate immediately.

2. **Fresh backup vs reconciliation:** If delta exists, do you prefer:
   - (a) Take a fresh `signing_documents` backup post-freeze and migrate from that (cleaner)
   - (b) Migrate from existing backup + run delta sync (more complex, same result)
   
   Recommendation: (a) — take a fresh pg_dump of just signing_documents after freeze activation.

3. **Active signing tokens:** Before activating freeze, should we warn any counterparties with pending tokens? If someone is mid-signature and gets a 503, that's a bad UX. Consider a grace period or communication.

---

## Summary

| Finding | Status |
|---------|--------|
| Unfreeze window | **STILL OPEN** — CONTRACT_WRITES_FROZEN not yet activated |
| New rows possible? | NO — PPCRM frozen, can't call generate |
| Status transitions possible? | YES — human signers can view/sign/decline |
| Migration plan (a79fcf0) validity | **UNCHANGED** — column mapping works for any row state |
| Recommended action | Activate freeze → detect delta → fresh backup if delta > 0 → migrate |
