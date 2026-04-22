# V4 Shell Refinement: Curated Pill Set + Picker

> Coder-3 Spec | D152 | For Coder-1 execution
> Source: D85 lock — "curated + picker (no free text)" for pill customization
> Build order: 3 of 6

---

## User-observable behavior

**Before:** The `+ Add pill` button opens a Milo conversation: "Make a new custom pill for me. Ask me what label and what query it should run." This is free-text — the operator describes what they want, Milo creates a custom pill via the `createCustomPill` tool. There's no catalog of available pills to browse.

**After:** Clicking `+ Add pill` opens a picker overlay (not a Milo conversation). The picker shows:

**Picker layout:**
1. **Header:** "Add a pill" with a close button
2. **Already active:** Pills currently in the bar, shown as disabled/checked items (can't add duplicates)
3. **Available:** All pills from the operator's role set that are not currently active (including any hidden via jiggle mode). Each shows: pill label + one-line description of what it surfaces.
4. **Other roles:** Collapsed section showing pills from other roles (e.g., an operations user can see billing pills). These are "off-role" — adding one is allowed but it's clear they're from a different domain.
5. **Custom:** "Create custom pill" option at the bottom — this one still opens Milo conversation for free-text pill creation.

**Tapping an available pill:**
- Immediately adds it to the bar
- Pill animates into position at the end of the bar
- Picker stays open (operator can add multiple pills)
- Added pill shows as checked/disabled in the picker

**Acceptance criteria:**
- Picker renders as a bottom sheet on mobile, center modal on desktop (responsive)
- Full catalog of 30 unique pills across all roles is browsable (deduplicated by id)
- Already-active pills are visually distinct (checked/grayed)
- Adding a pill persists via existing `createCustomPill` tool (reuse the storage path)
- OR: adding a role pill restores it from `hidden_pills` if it was hidden, or adds to `pill_order` if it's from another role
- "Create custom pill" option at bottom opens Milo conversation (existing flow)
- Picker is dismissible via close button, tap-outside, or Escape key
- No search/filter needed for v4.0 (30 pills fit in a scrollable list)

---

## File-level scope

| File | Change | ~LOC |
|---|---|---|
| `src/components/operator/PillPicker.tsx` | **New file.** Renders the picker overlay. Receives all pills (full catalog), active pill ids, and callbacks for add/create-custom. Groups pills by role. Responsive: bottom sheet (mobile) vs modal (desktop). | ~140 |
| `src/components/operator/PillBar.tsx` | Change `onAddClick` to open PillPicker instead of dispatching to Milo. Pass full catalog and active ids. | ~10 |
| `src/app/operator/page.tsx` | Add state for picker open/close. Pass `allPills` (full catalog from all roles) and `onAddPill` callback. | ~15 |
| `src/lib/operator-pills.ts` | Export `ALL_PILLS`: deduplicated flat list of all pills across all roles, with a `role` field added for grouping. Export pill descriptions (new `PILL_DESCRIPTIONS` map). | ~40 |
| `src/app/api/operator/route.ts` | Return `all_pills` in the response (full catalog for the picker). | ~5 |

**Total: ~210 LOC across 5 files.**

---

## Data dependencies

**No new schema.** The picker uses existing data:
- Active pills come from the existing `/api/operator` response (`pills[]`)
- Full catalog comes from `ROLE_PILLS` in `operator-pills.ts`
- Adding a role pill either un-hides it (`hidden_pills` update, if spec 03 ships first) or adds it as a pseudo-custom pill via the existing `custom_pills` JSONB path

**New data in operator-pills.ts:**
```typescript
export const PILL_DESCRIPTIONS: Record<string, string> = {
  qa_queue: 'Calls needing QA review right now',
  disputes_signoff: 'Billing disputes awaiting your sign-off',
  today_pulse: 'Today\'s calls, revenue, margin, conversions',
  // ... one line per pill
};
```

This is a static map, not a schema change.

---

## Visual requirements

**No HTML design reference exists.** D144 never landed.

**Assumed layout (flagged for Mark):**

Desktop (center modal, 480px wide):
```
┌─────────────────────────────────────────┐
│ Add a pill                           ✕  │
├─────────────────────────────────────────┤
│ YOUR PILLS                              │
│ ☑ QA queue                    (active)  │
│ ☑ Ping health                 (active)  │
├─────────────────────────────────────────┤
│ AVAILABLE                               │
│ ☐ MediaRite xref — Missing CIDs        │
│ ☐ Needs attention — Alerts needing...  │
│ ☐ Calls — Call-quality alerts           │
├─────────────────────────────────────────┤
│ OTHER ROLES                        ▸    │
│   ☐ Overdue — Overdue invoices          │
│   ☐ Pipeline — Prospect pipeline        │
├─────────────────────────────────────────┤
│ + Create custom pill                    │
└─────────────────────────────────────────┘
```

Mobile (bottom sheet, full width, 60vh max):
Same content, slides up from bottom with drag-to-dismiss handle.

**Styling:**
- Background: `#1c1c1e`
- Border: `0.5px solid #38383a`
- Border radius: 16px (modal), 16px top corners (bottom sheet)
- Section headers: `#636366`, 11px, uppercase, letter-spacing 0.5px
- Pill rows: 44px height, full-width tap target
- Checked pills: label in `#636366` (dimmed)
- Available pills: label in `#f5f5f7`, description in `#8e8e93`
- Backdrop: `rgba(0,0,0,0.6)` with blur if performant

---

## Blast radius

**Medium.** New component + modifications to PillBar and operator page. No schema changes.

**Risk 1:** The `+ Add pill` button currently opens Milo conversation. Changing it to open a picker changes user muscle memory for anyone who's used the current flow. **Mitigation:** The "Create custom pill" option at the bottom of the picker preserves the Milo path. The picker is strictly additive.

**Risk 2:** Showing pills from other roles might confuse operators ("why do I see billing pills?"). **Mitigation:** The "Other Roles" section is collapsed by default and labeled clearly. Most operators will only interact with the "Available" section.

**Risk 3:** Adding an off-role pill that has a drawer predicate the user's role doesn't normally surface. The pill opens a drawer, the drawer fetches data — works fine. But the predicate may return data scoped to a domain the operator doesn't normally see. **Mitigation:** This is a feature, not a bug — if an operations user wants to peek at disputes, they can. Supabase RLS handles actual data access.

---

## Build order rationale

Ranked 3 of 6 because:
- Directly addresses Mark's "no free text" directive — the current `+ Add pill` flow is free-text only
- No schema changes needed (uses existing custom_pills or hidden_pills paths)
- Pairs with spec 03 (jiggle) — hidden pills need the picker to be restored
- Can ship independently: even without jiggle mode, the picker replaces the inferior Milo-conversation-based pill creation
- Visual design is straightforward (modal/sheet with a list)

---

## Open questions

**Q1 (assumed answer flagged):** Should the picker show pill badge counts (like the PillBar does)? **Assumed: no** — the picker is for browsing/adding, not monitoring. Showing live counts in the picker would require fetching all pill counts on open, which is expensive (30 parallel API calls).

**Q2 (assumed answer flagged):** Should the "Other Roles" section be hidden entirely for admin users (who already see all pills)? **Assumed: yes** — admin role already has 15 pills spanning all domains. The "Other Roles" section would just duplicate what's already in "Available."
