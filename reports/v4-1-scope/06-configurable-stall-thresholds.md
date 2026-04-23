# V4.1: Configurable Sales Stall Thresholds

> Coder-3 stub | D164b | Source: D136 Sales drawer spec

## What

Promote the `SALES_STALL_THRESHOLDS` constant (hardcoded per-stage day thresholds) from the pill route to a user-configurable JSONB column on `user_profiles.pill_config`.

## Why

D136 spec defined stall thresholds: outreach 14d, qualifying 21d, drip 30d, onboarding 14d, activation 7d. These are v4.0 hardcoded constants. D136 noted: "Can promote to `user_profiles.pill_config` JSONB in v4.1 if operators want different sensitivity."

Different operators may have different pipeline velocity expectations. A billing-focused operator might not care about outreach stalls; a prospecting operator might want tighter thresholds.

## Scope

| File | Change | ~LOC |
|---|---|---|
| Schema: `user_profiles` | Add `pill_config JSONB DEFAULT '{}'` if not already present | Migration |
| `src/app/api/operator/pill/route.ts` | Read `pill_config.sales_stall_days` from user profile, merge with defaults | ~15 |
| Settings UI (TBD) | A simple form in the picker or a Milo conversation tool: "Set my qualifying stall threshold to 30 days" | ~30-50 |

## Rough LOC

~50 LOC + schema migration. The settings UI could be a Milo tool (reusing `createCustomPill` pattern) rather than a dedicated page.

## Dependencies

- V4.0 Sales drawer must ship first
- Decide: UI form vs Milo conversation tool for threshold editing
