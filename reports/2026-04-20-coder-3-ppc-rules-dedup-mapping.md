---
directive: "PPC contract rules consolidated dedup mapping"
lane: research
coder: Coder-3
started: 2026-04-20 ~12:00 MDT
completed: 2026-04-20 ~12:45 MDT
supplements:
  - reports/2026-04-19-coder-3-ppc-contract-three-way.md (40e2811)
  - reports/2026-04-20-coder-4-milo-for-ppc-consolidation-plan.md (71a29b4)
  - ~/MOP/lib/contract-checker/rules/registry.ts (source)
  - ~/Miloops3.18/milo-ops/src/lib/contract-analysis-prompt.ts (source)
  - ~/contract-checker/src/lib/playbook-rules.ts (source)
---

# PPC Contract Rules — Consolidated Dedup Mapping

## 1. Summary

| Metric | Count |
|--------|-------|
| MOP registry rules (active) | 70 |
| MOP legacy (deprecated) | 2 |
| milo-ops novel additions | 6 |
| contract-checker novel additions | 1 |
| **Total consolidated** | **77** |
| Classified as horizontal | **45** |
| Classified as TLP/PPC-specific | **32** |

**Plugin structure:**
- `@milo/contract-analysis` base registry: 45 horizontal rules (reusable by any vertical)
- Milo-for-PPC vertical plugin (`src/lib/ppc/rules.ts`): 32 PPC-specific rules

**Judgment calls locked (Directive #35):**
- #1 PUB_FIN_PAYMENT_TERMS_SLOW: horizontal + per-vertical threshold config (PPC=15d)
- #37 PUB_OPS_NON_CIRCUMVENT_BROAD: moved H→horizontal (brokerage pattern, not PPC-unique)
- #72 BUY_DATA_OWNERSHIP_UNCLEAR: stays horizontal, PPC overrides severity HIGH→CRIT
- #57 BUY_FIN_PREPAY_OR_DEPOSIT_NO_REFUND: stays horizontal, no change
- 6 milo-ops novel: all canonical. #18/#43/#44/#45 = horizontal. #8/#9 = PPC.

**PPC no-score directive:** Milo-for-PPC displays per-issue severity only. No aggregate score, no letter grade, no risk bucket label. Engine retains scoring capability for other verticals.

---

## 1b. Vertical Override Patterns

### Threshold Config (per-vertical defaults)

Horizontal rules with thresholds ship with a `defaultThreshold`. Each vertical overrides:

```typescript
// In @milo/contract-analysis base registry
{ ruleId: 'PUB_FIN_PAYMENT_TERMS_SLOW', threshold: { strict: 15, standard: 30, relaxed: 45 } }

// In Milo-for-PPC vertical config
export const PPC_THRESHOLD_OVERRIDES: Record<string, ThresholdConfig> = {
  'PUB_FIN_PAYMENT_TERMS_SLOW': { strict: 10, standard: 15, relaxed: 30 },
  // PPC operators flag anything >15 days
};
```

### Severity Override (vertical escalation)

Horizontal rules ship with a `defaultSeverity`. PPC can escalate:

```typescript
export const PPC_SEVERITY_OVERRIDES: Record<string, Severity> = {
  'BUY_DATA_OWNERSHIP_UNCLEAR': 'CRITICAL',    // default HIGH → PPC CRITICAL
  'PUB_OPS_NON_CIRCUMVENT_BROAD': 'CRITICAL',  // default HIGH → PPC CRITICAL
};
```

### No-Score Display Config

```typescript
// src/lib/ppc/vertical-display.ts
export const PPC_DISPLAY_CONFIG = {
  displayAggregateScore: false,
  displayLetterGrade: false,
  displayRiskBucket: false,
  issueOrdering: 'severity-desc',  // CRITICAL first, LOW last
  perIssueFields: ['severity', 'plainEnglish', 'recommendation', 'riderText'],
} as const;
```

The engine still computes scores internally (used for sorting, for other verticals, for future analytics). Milo-for-PPC simply never renders them to operators.

---

## 2. Master Rules Table

Legend:
- **MOP**: present in MOP registry (registry.ts)
- **OPS**: present in milo-ops prompt (contract-analysis-prompt.ts)
- **CC**: maps to contract-checker PlaybookRule
- **Layer**: H = horizontal (@milo/contract-analysis), V = vertical (Milo-for-PPC)
- **Sev**: proposed canonical severity
- **Wt**: proposed canonical baseWeight

---

### PUBLISHER FINANCIAL (PUB_FIN_*)

| # | Canonical ID | Title | MOP | OPS | CC | Sev | Wt | Layer | Notes |
|---|---|---|---|---|---|---|---|---|---|
| 1 | PUB_FIN_PAYMENT_TERMS_SLOW | Slow Payment Terms | Y | — | `payment_terms` | MEDIUM | 12 | H | CC adds configurable threshold (30d default) |
| 2 | PUB_FIN_FINALITY_MISSING | Payment Finality Missing | Y | Y | — | CRITICAL | 40 | V | PPC-specific: lead/call finality after rejection window. milo-ops mandatory check. |
| 3 | PUB_FIN_REPORTING_BINDING_NO_AUDIT | Binding Reporting Without Audit | Y | Y | — | CRITICAL | 35 | V | PPC-specific: advertiser reporting as sole truth |
| 4 | PUB_FIN_HOLDBACK_NO_RELEASE | Holdback Without Release Timeline | Y | Y | `holdback_cap` | HIGH | 20 | V | PPC-specific: reserve holdbacks on call revenue |
| 5 | PUB_FIN_CLAWBACK_OPEN_ENDED | Open-Ended Clawback | Y | Y | `clawback_period` | CRITICAL | 40 | V | PPC-specific: recoupment of lead/call payments |
| 6 | PUB_FIN_POST_TERM_PAYMENT_MISSING | Post-Termination Payment Missing | Y | Y | `post_termination_payments` | CRITICAL | 35 | H | General contract concern |
| 7 | PUB_FIN_RATE_CHANGE_NO_NOTICE | Rate Change Without Notice | Y | — | `rate_change_notice` | MEDIUM | 12 | H | General contract concern |
| 8 | PUB_FIN_REJECTION_CATCH_ALL | Catch-All Rejection Language | — | Y | — | HIGH | 25 | V | **milo-ops novel.** "any other reason", "sole discretion" in rejection clause. PPC-specific. |
| 9 | PUB_FIN_INVOICE_FORFEITURE | Invoice Submission Forfeiture | — | Y | — | HIGH | 20 | V | **milo-ops novel.** Failure to invoice within X days = forfeiture. PPC-specific. |

---

### PUBLISHER LIABILITY (PUB_LIAB_*)

| # | Canonical ID | Title | MOP | OPS | CC | Sev | Wt | Layer | Notes |
|---|---|---|---|---|---|---|---|---|---|
| 10 | PUB_LIAB_INDEMNITY_TRIGGER_LOW | Low Indemnity Trigger | Y | Y | — | CRITICAL | 35 | H | "alleged", "suspected" triggers |
| 11 | PUB_LIAB_INDEMNITY_SCOPE_DOWNSTREAM | Downstream Indemnity Scope | Y | Y | — | HIGH | 30 | H | Covers third parties |
| 12 | PUB_LIAB_INDEMNITY_NO_CAP | Uncapped Indemnity | Y | Y | `indemnification_balance` | CRITICAL | 40 | H | Indemnity excluded from cap |
| 13 | PUB_LIAB_LIABILITY_CAP_MISSING | General Liability Cap Missing | Y | — | `liability_cap` | HIGH | 20 | H | Universal concern |
| 14 | PUB_LIAB_LIABILITY_CAP_LIMITED_SCOPE | Liability Cap Has Limited Scope | Y | — | — | HIGH | 20 | H | Cap exists but carve-outs negate it |
| 15 | PUB_LIAB_CONSEQUENTIAL_DAMAGES | Consequential Damages Not Waived | Y | — | `consequential_damages` | MEDIUM | 12 | H | General contract concern |
| 16 | PUB_LIAB_INSURANCE_EXCESSIVE | Excessive Insurance Requirements | Y | — | `insurance_requirements` | MEDIUM | 12 | H | General concern |
| 17 | PUB_LIAB_PERSONAL_GUARANTY | Personal Guaranty Required | Y | Y | — | CRITICAL | 45 | H | Existential — any industry |
| 18 | PUB_LIAB_INDEMNITY_ONE_SIDED | One-Sided Indemnification | — | Y | — | CRITICAL | 35 | H | **milo-ops novel.** Asymmetric obligations. General concern. |

---

### PUBLISHER COMPLIANCE (PUB_COMP_*)

| # | Canonical ID | Title | MOP | OPS | CC | Sev | Wt | Layer | Notes |
|---|---|---|---|---|---|---|---|---|---|
| 19 | PUB_COMP_TCPA_STRICT_SHIFT | TCPA Liability Shift | Y | Y | `tcpa_indemnification` | CRITICAL | 40 | V | PPC-specific: TCPA is pay-per-call industry regulation |
| 20 | PUB_COMP_TCPA_RECORD_BURDEN | Excessive Record Retention | Y | Y | — | HIGH | 20 | V | PPC-specific: consent record retention |
| 21 | PUB_COMP_TCPA_RECORD_PRODUCTION_FAST | Fast Record Production (3-10 days) | Y | Y | — | HIGH | 25 | V | PPC-specific |
| 22 | PUB_COMP_TCPA_RECORD_PRODUCTION_EXTREME | Extreme Record Production (<=2 days) | Y | Y | — | CRITICAL | 40 | V | PPC-specific |
| 23 | PUB_COMP_FTC_UNCLEAR | FTC Compliance Unclear | Y | — | `ftc_compliance` | MEDIUM | 12 | H | General marketing compliance |
| 24 | PUB_COMP_IP_ASSIGNMENT_BROAD | Broad IP Assignment | Y | — | `ip_assignment` | MEDIUM | 12 | H | General concern |
| 25 | PUB_COMP_DATA_OWNERSHIP_UNCLEAR | Data Ownership Unclear | Y | — | `data_ownership` | MEDIUM | 12 | H | General concern |
| 26 | PUB_COMP_GDPR_MISSING | GDPR Provisions Missing | Y | — | `gdpr_compliance` | LOW | 5 | H | General concern |
| 27 | PUB_COMP_CASL_MISSING | CASL Compliance Missing | — | — | `casl_compliance` | HIGH | 20 | H | **contract-checker novel.** Canadian Anti-Spam. General concern. |

---

### PUBLISHER TERMINATION (PUB_TERM_*)

| # | Canonical ID | Title | MOP | OPS | CC | Sev | Wt | Layer | Notes |
|---|---|---|---|---|---|---|---|---|---|
| 28 | PUB_TERM_ONE_SIDED | One-Sided Termination | Y | — | `termination_convenience` | HIGH | 20 | H | General concern |
| 29 | PUB_TERM_POST_TERM_FORFEIT | Post-Termination Forfeiture | Y | — | `post_termination_payments` | CRITICAL | 35 | H | Existential — any industry |
| 30 | PUB_TERM_TRAILING_MISSING | Trailing Commission Missing | Y | — | `trailing_commissions` | MEDIUM | 12 | V | PPC-specific: ongoing revenue from acquired customers |
| 31 | PUB_TERM_DATA_PORTABILITY_MISSING | Data Portability Missing | Y | — | `data_portability` | LOW | 5 | H | General concern |
| 32 | PUB_TERM_NOTICE_TOO_SHORT | Termination Notice Too Short | Y | — | `termination_notice` | MEDIUM | 12 | H | General concern |
| 33 | PUB_TERM_AUTO_RENEW_TRAP | Auto-Renewal Trap | Y | — | `auto_renewal` | MEDIUM | 12 | H | General concern |
| 34 | PUB_TERM_MINIMUM_TERM_LONG | Long Minimum Term | Y | — | `minimum_term` | MEDIUM | 12 | H | General concern |

---

### PUBLISHER OPERATIONAL (PUB_OPS_*)

| # | Canonical ID | Title | MOP | OPS | CC | Sev | Wt | Layer | Notes |
|---|---|---|---|---|---|---|---|---|---|
| 35 | PUB_OPS_EXCLUSIVITY | Exclusivity Requirement | Y | — | `exclusive_dealing` | HIGH | 20 | H | General concern |
| 36 | PUB_OPS_NON_COMPETE_BROAD | Broad Non-Compete | Y | — | `non_compete_scope` | HIGH | 20 | H | General concern |
| 37 | PUB_OPS_NON_CIRCUMVENT_BROAD | Broad Non-Circumvention | Y | Y | — | HIGH | 25 | H | Brokerage pattern (not PPC-unique). PPC overrides severity to CRITICAL. |
| 38 | PUB_OPS_APPROVAL_UNILATERAL | Unilateral Approval Rights | Y | — | `approval_rights` | MEDIUM | 12 | H | General concern |
| 39 | PUB_OPS_CALL_RECORDING_CONSENT | Call Recording Consent Issues | Y | — | `call_recording_consent` | HIGH | 20 | V | PPC-specific: two-party consent states for call recording |
| 40 | PUB_OPS_AUDIT_MISSING | Audit Rights Missing | Y | — | `audit_rights` | MEDIUM | 12 | H | General concern |
| 41 | PUB_OPS_BROKERING_BAN | Brokering Prohibition | Y | — | — | HIGH | 20 | V | PPC-specific: lead/call brokering |
| 42 | PUB_OPS_LIQUIDATED_DAMAGES_100 | 100% Liquidated Damages | Y | Y | — | CRITICAL | 35 | H | Existential — any industry |
| 43 | PUB_OPS_INSURANCE_ADDITIONAL_INSURED | Insurance + Additional Insured | — | Y | — | HIGH | 20 | H | **milo-ops novel.** Must name company as additional insured. General concern. |
| 44 | PUB_OPS_VENUE_JURY_WAIVER | Venue + Jury Waiver Combo | — | Y | — | MEDIUM | 12 | H | **milo-ops novel.** Distant venue AND jury waiver without arbitration. General concern. |
| 45 | PUB_OPS_CROSS_REF_ERROR | Cross-Reference Errors | — | Y | — | MEDIUM | 12 | H | **milo-ops novel.** Section numbers pointing to wrong sections. General concern. |

---

### PUBLISHER DISPUTE (PUB_DISP_*)

| # | Canonical ID | Title | MOP | OPS | CC | Sev | Wt | Layer | Notes |
|---|---|---|---|---|---|---|---|---|---|
| 46 | PUB_DISP_VENUE_INCONVENIENT | Inconvenient Venue | Y | — | `venue_selection` | MEDIUM | 12 | H | General concern |
| 47 | PUB_DISP_ARBITRATION_UNFAIR | Unfair Arbitration | Y | — | `arbitration_fairness` | MEDIUM | 12 | H | General concern |
| 48 | PUB_DISP_CLASS_ACTION_WAIVER | Class Action Waiver | Y | — | `class_action_waiver` | MEDIUM | 12 | H | General concern |
| 49 | PUB_DISP_DISPUTE_DEADLINE_SHORT | Short Dispute Deadline | Y | — | `dispute_resolution_timeline` | MEDIUM | 12 | H | General concern |
| 50 | PUB_DISP_INJUNCTION_NO_BOND | Injunction Without Bond | Y | Y | — | CRITICAL | 35 | H | Existential — any industry |
| 51 | PUB_DISP_LIQUIDATED_DAMAGES_EXCESSIVE | Excessive Liquidated Damages | Y | — | — | HIGH | 20 | H | General concern |

---

### BUYER FINANCIAL (BUY_FIN_*)

| # | Canonical ID | Title | MOP | OPS | CC | Sev | Wt | Layer | Notes |
|---|---|---|---|---|---|---|---|---|---|
| 52 | BUY_FIN_QUALITY_STANDARDS_VAGUE | Vague Lead Quality Standards | Y | Y | `quality_standards` | HIGH | 30 | V | PPC-specific: lead/call quality acceptance criteria |
| 53 | BUY_FIN_NO_REJECTION_WINDOW | No Rejection Window | Y | Y | — | CRITICAL | 40 | V | PPC-specific: lead/call rejection timeline |
| 54 | BUY_FIN_NO_REPLACEMENT_OR_CREDIT | No Replacement or Credit Remedy | Y | Y | — | CRITICAL | 40 | V | PPC-specific: rejected lead remedy |
| 55 | BUY_FIN_REJECTION_PROCESS_NOT_DEFINED | Rejection Process Undefined | Y | Y | — | HIGH | 30 | V | PPC-specific: how to submit rejections |
| 56 | BUY_FIN_CHARGEBACK_RIGHTS_TOO_NARROW | Chargeback Rights Too Narrow | Y | — | `chargeback_liability` | HIGH | 32 | V | PPC-specific: clawback on bad leads |
| 57 | BUY_FIN_PREPAY_OR_DEPOSIT_NO_REFUND | Prepay/Deposit with No Refund | Y | — | — | HIGH | 38 | H | General concern |
| 58 | BUY_FIN_PAYMENT_TERMS_TOO_FAST | Payment Due Before Validation | Y | — | — | MEDIUM | 30 | V | PPC-specific: pay before lead validation |
| 59 | BUY_FIN_REPORTING_NOT_BINDING | Reporting Non-Binding | Y | Y | — | HIGH | 32 | V | PPC-specific: advertiser reporting disputes |

---

### BUYER OPERATIONAL (BUY_OPS_*)

| # | Canonical ID | Title | MOP | OPS | CC | Sev | Wt | Layer | Notes |
|---|---|---|---|---|---|---|---|---|---|
| 60 | BUY_OPS_NO_AUDIT_RIGHTS | No Audit Rights | Y | Y | — | CRITICAL | 35 | H | General concern |
| 61 | BUY_OPS_NO_CONSENT_PROOF_ACCESS | No Access to Consent Proof | Y | Y | — | CRITICAL | 40 | V | PPC-specific: Jornaya/TrustedForm access |
| 62 | BUY_OPS_CHANGE_CONTROL_MISSING | No Change Control | Y | Y | — | HIGH | 25 | H | General concern |
| 63 | BUY_OPS_NO_MINIMUM_VOLUME | No Minimum Volume Commitment | Y | — | — | MEDIUM | 22 | V | PPC-specific: lead volume guarantees |
| 64 | BUY_OPS_BROKERING_ALLOWED_UNDISCLOSED | Undisclosed Brokering Allowed | Y | — | — | HIGH | 28 | V | PPC-specific: sub-affiliate brokering |
| 65 | BUY_OPS_CALL_TRANSFER_NO_REJECT | Call Transfer Rejection Prohibited | Y | Y | — | CRITICAL | 40 | V | PPC-specific: buyer can't reject bad transfers |
| 66 | BUY_OPS_TERMINATION_IMBALANCE | Asymmetric Termination Rights | Y | Y | — | CRITICAL | 35 | H | General concern |

---

### BUYER COMPLIANCE (BUY_COMP_*)

| # | Canonical ID | Title | MOP | OPS | CC | Sev | Wt | Layer | Notes |
|---|---|---|---|---|---|---|---|---|---|
| 67 | BUY_COMP_TCPA_WARRANTY_MISSING | TCPA Warranty Missing | Y | Y | — | CRITICAL | 40 | V | PPC-specific: TCPA compliance warranty |
| 68 | BUY_COMP_TCPA_INDEMNITY_WEAK | Weak TCPA Indemnity | Y | Y | — | CRITICAL | 35 | V | PPC-specific |
| 69 | BUY_COMP_RECORD_RETENTION_TOO_SHORT | Record Retention Too Short | Y | Y | — | HIGH | 28 | V | PPC-specific: TCPA SOL < 4 years |
| 70 | BUY_COMP_RECORD_PRODUCTION_UNREALISTIC | Record Production Unrealistic | Y | — | — | HIGH | 30 | V | PPC-specific |
| 71 | BUY_COMP_SUBAFFILIATE_COMPLIANCE_GAP | Subaffiliate Compliance Gap | Y | Y | — | CRITICAL | 35 | V | PPC-specific: sub-affiliate flow-down |

---

### BUYER DATA (BUY_DATA_*)

| # | Canonical ID | Title | MOP | OPS | CC | Sev | Wt | Layer | Notes |
|---|---|---|---|---|---|---|---|---|---|
| 72 | BUY_DATA_OWNERSHIP_UNCLEAR | Data Ownership Unclear | Y | Y | — | CRITICAL | 38 | H | General concern (escalated in PPC context) |
| 73 | BUY_DATA_LICENSE_BACK_TO_PUBLISHER | Publisher Retains Data License | Y | Y | — | HIGH | 30 | V | PPC-specific: lead data resale |
| 74 | BUY_DATA_EXCLUSIVITY_MISSING | Exclusivity Missing | Y | — | — | MEDIUM | 22 | V | PPC-specific: exclusive lead purchase |

---

### BUYER LIABILITY (BUY_LIAB_*)

| # | Canonical ID | Title | MOP | OPS | CC | Sev | Wt | Layer | Notes |
|---|---|---|---|---|---|---|---|---|---|
| 75 | BUY_LIAB_WARRANTY_DISCLAIMER_AS_IS | "As-Is" Warranty Disclaimer | Y | Y | — | CRITICAL | 40 | H | General concern |
| 76 | BUY_LIAB_CAP_TOO_LOW_FOR_COMPLIANCE | Liability Cap Inadequate | Y | Y | — | HIGH | 32 | V | PPC-specific: TCPA exposure vs cap |

---

### BUYER DISPUTE (BUY_DISP_*)

| # | Canonical ID | Title | MOP | OPS | CC | Sev | Wt | Layer | Notes |
|---|---|---|---|---|---|---|---|---|---|
| 77 | BUY_DISP_DISPUTE_WINDOW_TOO_SHORT | Dispute Window Too Short | Y | — | — | HIGH | 28 | H | General concern |

---

## 3. Discarded Rules

| Source | ID | Reason |
|--------|---|--------|
| MOP (legacy) | BUY_QUALITY_STANDARDS_VAGUE | Duplicate of BUY_FIN_QUALITY_STANDARDS_VAGUE |
| MOP (legacy) | BUY_TCPA_COMPLIANCE_GAP | Duplicate of BUY_COMP_TCPA_WARRANTY_MISSING |

Original three-way report cited 6 phantom rules. Registry extraction did not surface them — possibly dead code in post-processing (rule IDs referenced but never matched by AI). If they surface during implementation, discard.

---

## 4. contract-checker Slug → Canonical ID Mapping

For Coder-1 implementation: every CC slug maps to a canonical ID.

| CC Slug | Canonical ID | Notes |
|---------|---|---|
| `payment_terms` | PUB_FIN_PAYMENT_TERMS_SLOW | CC adds threshold config (30d) |
| `clawback_period` | PUB_FIN_CLAWBACK_OPEN_ENDED | CC adds threshold (14d) |
| `holdback_cap` | PUB_FIN_HOLDBACK_NO_RELEASE | CC adds threshold (10%) |
| `chargeback_liability` | BUY_FIN_CHARGEBACK_RIGHTS_TOO_NARROW | Close match |
| `rate_change_notice` | PUB_FIN_RATE_CHANGE_NO_NOTICE | CC adds threshold (30d) |
| `termination_convenience` | PUB_TERM_ONE_SIDED | Same concept |
| `post_termination_payments` | PUB_FIN_POST_TERM_PAYMENT_MISSING | CC conflates with PUB_TERM_POST_TERM_FORFEIT |
| `trailing_commissions` | PUB_TERM_TRAILING_MISSING | Same |
| `data_portability` | PUB_TERM_DATA_PORTABILITY_MISSING | Same |
| `termination_notice` | PUB_TERM_NOTICE_TOO_SHORT | CC adds threshold (30d) |
| `auto_renewal` | PUB_TERM_AUTO_RENEW_TRAP | CC adds threshold (30d notice) |
| `minimum_term` | PUB_TERM_MINIMUM_TERM_LONG | Same |
| `liability_cap` | PUB_LIAB_LIABILITY_CAP_MISSING | Same |
| `indemnification_balance` | PUB_LIAB_INDEMNITY_NO_CAP + PUB_LIAB_INDEMNITY_ONE_SIDED | CC conflates two MOP rules |
| `consequential_damages` | PUB_LIAB_CONSEQUENTIAL_DAMAGES | Same |
| `insurance_requirements` | PUB_LIAB_INSURANCE_EXCESSIVE | CC adds threshold ($2M) |
| `exclusive_dealing` | PUB_OPS_EXCLUSIVITY | Same |
| `non_compete_scope` | PUB_OPS_NON_COMPETE_BROAD | CC adds jurisdiction trigger (CA) |
| `approval_rights` | PUB_OPS_APPROVAL_UNILATERAL | Same |
| `call_recording_consent` | PUB_OPS_CALL_RECORDING_CONSENT | CC adds two-party state list |
| `quality_standards` | BUY_FIN_QUALITY_STANDARDS_VAGUE | Same concept, buyer perspective |
| `tcpa_indemnification` | PUB_COMP_TCPA_STRICT_SHIFT | Same |
| `ftc_compliance` | PUB_COMP_FTC_UNCLEAR | Same |
| `ip_assignment` | PUB_COMP_IP_ASSIGNMENT_BROAD | Same |
| `data_ownership` | PUB_COMP_DATA_OWNERSHIP_UNCLEAR | Same |
| `gdpr_compliance` | PUB_COMP_GDPR_MISSING | Same |
| `casl_compliance` | PUB_COMP_CASL_MISSING | **Novel** — added as #27 |
| `venue_selection` | PUB_DISP_VENUE_INCONVENIENT | Same |
| `arbitration_fairness` | PUB_DISP_ARBITRATION_UNFAIR | Same |
| `class_action_waiver` | PUB_DISP_CLASS_ACTION_WAIVER | Same |
| `audit_rights` | PUB_OPS_AUDIT_MISSING | Same |
| `dispute_resolution_timeline` | PUB_DISP_DISPUTE_DEADLINE_SHORT | CC adds threshold (90d) |

---

## 5. Plugin Architecture

### @milo/contract-analysis Base Registry (45 horizontal rules)

These rules apply to any B2B contract regardless of industry:

```typescript
// packages/contract-analysis/src/rules/base-registry.ts
export const BASE_RULES: RuleDefinition[] = [
  // Liability (9)
  { ruleId: 'PUB_LIAB_INDEMNITY_TRIGGER_LOW', ... },
  { ruleId: 'PUB_LIAB_INDEMNITY_SCOPE_DOWNSTREAM', ... },
  { ruleId: 'PUB_LIAB_INDEMNITY_NO_CAP', ... },
  { ruleId: 'PUB_LIAB_LIABILITY_CAP_MISSING', ... },
  { ruleId: 'PUB_LIAB_LIABILITY_CAP_LIMITED_SCOPE', ... },
  { ruleId: 'PUB_LIAB_CONSEQUENTIAL_DAMAGES', ... },
  { ruleId: 'PUB_LIAB_INSURANCE_EXCESSIVE', ... },
  { ruleId: 'PUB_LIAB_PERSONAL_GUARANTY', ... },
  { ruleId: 'PUB_LIAB_INDEMNITY_ONE_SIDED', ... },
  // Financial + Termination (9)
  { ruleId: 'PUB_FIN_PAYMENT_TERMS_SLOW', ... },      // threshold config per vertical
  { ruleId: 'PUB_FIN_POST_TERM_PAYMENT_MISSING', ... },
  { ruleId: 'PUB_FIN_RATE_CHANGE_NO_NOTICE', ... },
  { ruleId: 'PUB_TERM_ONE_SIDED', ... },
  { ruleId: 'PUB_TERM_POST_TERM_FORFEIT', ... },
  { ruleId: 'PUB_TERM_DATA_PORTABILITY_MISSING', ... },
  { ruleId: 'PUB_TERM_NOTICE_TOO_SHORT', ... },
  { ruleId: 'PUB_TERM_AUTO_RENEW_TRAP', ... },
  { ruleId: 'PUB_TERM_MINIMUM_TERM_LONG', ... },
  // Operational (9) — includes #37 non-circumvent (reclassified from PPC)
  { ruleId: 'PUB_OPS_EXCLUSIVITY', ... },
  { ruleId: 'PUB_OPS_NON_COMPETE_BROAD', ... },
  { ruleId: 'PUB_OPS_NON_CIRCUMVENT_BROAD', ... },    // brokerage pattern, not PPC-unique
  { ruleId: 'PUB_OPS_APPROVAL_UNILATERAL', ... },
  { ruleId: 'PUB_OPS_AUDIT_MISSING', ... },
  { ruleId: 'PUB_OPS_LIQUIDATED_DAMAGES_100', ... },
  { ruleId: 'PUB_OPS_INSURANCE_ADDITIONAL_INSURED', ... },
  { ruleId: 'PUB_OPS_VENUE_JURY_WAIVER', ... },
  { ruleId: 'PUB_OPS_CROSS_REF_ERROR', ... },
  // Compliance (5)
  { ruleId: 'PUB_COMP_FTC_UNCLEAR', ... },
  { ruleId: 'PUB_COMP_IP_ASSIGNMENT_BROAD', ... },
  { ruleId: 'PUB_COMP_DATA_OWNERSHIP_UNCLEAR', ... },
  { ruleId: 'PUB_COMP_GDPR_MISSING', ... },
  { ruleId: 'PUB_COMP_CASL_MISSING', ... },
  // Dispute (6)
  { ruleId: 'PUB_DISP_VENUE_INCONVENIENT', ... },
  { ruleId: 'PUB_DISP_ARBITRATION_UNFAIR', ... },
  { ruleId: 'PUB_DISP_CLASS_ACTION_WAIVER', ... },
  { ruleId: 'PUB_DISP_DISPUTE_DEADLINE_SHORT', ... },
  { ruleId: 'PUB_DISP_INJUNCTION_NO_BOND', ... },
  { ruleId: 'PUB_DISP_LIQUIDATED_DAMAGES_EXCESSIVE', ... },
  // Buyer general (7)
  { ruleId: 'BUY_FIN_PREPAY_OR_DEPOSIT_NO_REFUND', ... },
  { ruleId: 'BUY_OPS_NO_AUDIT_RIGHTS', ... },
  { ruleId: 'BUY_OPS_CHANGE_CONTROL_MISSING', ... },
  { ruleId: 'BUY_OPS_TERMINATION_IMBALANCE', ... },
  { ruleId: 'BUY_DATA_OWNERSHIP_UNCLEAR', ... },       // PPC overrides severity to CRITICAL
  { ruleId: 'BUY_LIAB_WARRANTY_DISCLAIMER_AS_IS', ... },
  { ruleId: 'BUY_DISP_DISPUTE_WINDOW_TOO_SHORT', ... },
];
```

### Milo-for-PPC Vertical Plugin (45 PPC-specific rules)

```typescript
// src/lib/ppc/rules.ts
import type { RuleDefinition } from '@milo/contract-analysis';

export const PPC_RULES: RuleDefinition[] = [
  // PPC Financial (7)
  { ruleId: 'PUB_FIN_FINALITY_MISSING', ... },
  { ruleId: 'PUB_FIN_REPORTING_BINDING_NO_AUDIT', ... },
  { ruleId: 'PUB_FIN_HOLDBACK_NO_RELEASE', ... },
  { ruleId: 'PUB_FIN_CLAWBACK_OPEN_ENDED', ... },
  { ruleId: 'PUB_FIN_REJECTION_CATCH_ALL', ... },
  { ruleId: 'PUB_FIN_INVOICE_FORFEITURE', ... },
  { ruleId: 'PUB_TERM_TRAILING_MISSING', ... },
  // TCPA/Compliance (9)
  { ruleId: 'PUB_COMP_TCPA_STRICT_SHIFT', ... },
  { ruleId: 'PUB_COMP_TCPA_RECORD_BURDEN', ... },
  { ruleId: 'PUB_COMP_TCPA_RECORD_PRODUCTION_FAST', ... },
  { ruleId: 'PUB_COMP_TCPA_RECORD_PRODUCTION_EXTREME', ... },
  { ruleId: 'BUY_COMP_TCPA_WARRANTY_MISSING', ... },
  { ruleId: 'BUY_COMP_TCPA_INDEMNITY_WEAK', ... },
  { ruleId: 'BUY_COMP_RECORD_RETENTION_TOO_SHORT', ... },
  { ruleId: 'BUY_COMP_RECORD_PRODUCTION_UNREALISTIC', ... },
  { ruleId: 'BUY_COMP_SUBAFFILIATE_COMPLIANCE_GAP', ... },
  // PPC Operational (6) — #37 non-circumvent moved to horizontal
  { ruleId: 'PUB_OPS_CALL_RECORDING_CONSENT', ... },
  { ruleId: 'PUB_OPS_BROKERING_BAN', ... },
  { ruleId: 'BUY_OPS_NO_CONSENT_PROOF_ACCESS', ... },
  { ruleId: 'BUY_OPS_NO_MINIMUM_VOLUME', ... },
  { ruleId: 'BUY_OPS_BROKERING_ALLOWED_UNDISCLOSED', ... },
  { ruleId: 'BUY_OPS_CALL_TRANSFER_NO_REJECT', ... },
  // PPC Buyer Financial (7)
  { ruleId: 'BUY_FIN_QUALITY_STANDARDS_VAGUE', ... },
  { ruleId: 'BUY_FIN_NO_REJECTION_WINDOW', ... },
  { ruleId: 'BUY_FIN_NO_REPLACEMENT_OR_CREDIT', ... },
  { ruleId: 'BUY_FIN_REJECTION_PROCESS_NOT_DEFINED', ... },
  { ruleId: 'BUY_FIN_CHARGEBACK_RIGHTS_TOO_NARROW', ... },
  { ruleId: 'BUY_FIN_PAYMENT_TERMS_TOO_FAST', ... },
  { ruleId: 'BUY_FIN_REPORTING_NOT_BINDING', ... },
  // PPC Buyer Data/Liability (3)
  { ruleId: 'BUY_DATA_LICENSE_BACK_TO_PUBLISHER', ... },
  { ruleId: 'BUY_DATA_EXCLUSIVITY_MISSING', ... },
  { ruleId: 'BUY_LIAB_CAP_TOO_LOW_FOR_COMPLIANCE', ... },
];

// Severity overrides: horizontal rules that PPC escalates
export const PPC_SEVERITY_OVERRIDES: Record<string, 'LOW' | 'MEDIUM' | 'HIGH' | 'CRITICAL'> = {
  'BUY_DATA_OWNERSHIP_UNCLEAR': 'CRITICAL',       // default HIGH → PPC CRITICAL (lead data is existential)
  'PUB_OPS_NON_CIRCUMVENT_BROAD': 'CRITICAL',     // default HIGH → PPC CRITICAL (publisher/buyer protection)
};

// Threshold overrides: horizontal rules with PPC-specific trigger points
export const PPC_THRESHOLD_OVERRIDES: Record<string, { strict: number; standard: number; relaxed: number }> = {
  'PUB_FIN_PAYMENT_TERMS_SLOW': { strict: 10, standard: 15, relaxed: 30 },  // PPC: flag >15 days
};
```

### RuleDefinition Type

```typescript
export interface RuleDefinition {
  ruleId: string;
  title: string;
  category: 'FIN' | 'LIAB' | 'COMP' | 'TERM' | 'OPS' | 'DISP' | 'DATA';
  defaultSeverity: 'LOW' | 'MEDIUM' | 'HIGH' | 'CRITICAL';
  baseWeight: number;
  isExistential: boolean;
  tags: string[];
  // Prompt enrichment (from milo-ops)
  promptFragment: string;         // Natural language instruction for AI
  plainEnglishHint?: string;      // "1-2 sentences a non-lawyer can understand"
  impactTemplate?: string;        // "This affects {company} because..."
  recommendationFormat?: 'PUSH_BACK' | 'ACCEPT_WITH_MODIFICATION' | 'ACCEPT_AS_IS' | 'REJECT';
  // Registry metadata (from MOP)
  riderRefs?: string[];           // Rider section references
  isRecordingRelated?: boolean;   // For two-party consent escalation
  // Threshold config (from contract-checker)
  threshold?: {
    strict: number | string;
    standard: number | string;
    relaxed: number | string;
  };
  jurisdictionTriggers?: string[];  // States/countries that activate this rule
}
```

---

## 6. Scoring Architecture

Carry MOP's scoring engine into @milo/contract-analysis with configurable weights:

```typescript
export interface ScoringConfig {
  severityWeights: Record<Severity, number>;  // Default: LOW=5, MED=12, HIGH=20, CRIT=35
  maxScore: number;                           // Default: 200 (normalize to 0-100)
  riskBuckets: { critical: number; high: number; medium: number };  // Default: 75, 55, 30
  profileMultipliers?: Record<string, Record<string, number>>;  // Per-rule weight overrides
}

// PPC profiles (from MOP)
export const PPC_PROFILES = {
  TEST: { /* standard weights */ },
  SCALE: { /* relaxed weights for high-volume publishers */ },
  BROKER: { /* emphasize brokering + sub-affiliate rules */ },
  BUYER: { /* emphasize buyer-side rules, de-emphasize publisher */ },
};
```

---

## 7. milo-ops Mandatory Check List

milo-ops defines 17 rules as "MUST evaluate on EVERY contract". These become the `mandatoryChecks` array in the PPC plugin:

```typescript
export const PPC_MANDATORY_CHECKS: string[] = [
  'PUB_FIN_FINALITY_MISSING',
  'BUY_OPS_NO_AUDIT_RIGHTS',
  'PUB_FIN_POST_TERM_PAYMENT_MISSING',
  'PUB_LIAB_INDEMNITY_ONE_SIDED',
  'PUB_LIAB_INDEMNITY_NO_CAP',
  'PUB_LIAB_INDEMNITY_TRIGGER_LOW',
  'PUB_COMP_TCPA_STRICT_SHIFT',
  'PUB_COMP_TCPA_RECORD_BURDEN',
  'PUB_OPS_NON_CIRCUMVENT_BROAD',
  'PUB_FIN_REJECTION_CATCH_ALL',
  'PUB_OPS_INSURANCE_ADDITIONAL_INSURED',
  'PUB_OPS_VENUE_JURY_WAIVER',
  'PUB_FIN_INVOICE_FORFEITURE',
  'PUB_OPS_CROSS_REF_ERROR',
  'BUY_OPS_CALL_TRANSFER_NO_REJECT',
  'BUY_OPS_TERMINATION_IMBALANCE',
  'BUY_DATA_OWNERSHIP_UNCLEAR',
];
```

The engine's post-processing pipeline checks: if any mandatory rule was NOT flagged by the AI, inject a "not detected — verify manually" note.

---

## 8. Two-Party Consent Escalation

From MOP's `applyTwoPartyConsentEscalation()` — becomes a PatternPlugin:

```typescript
export const TWO_PARTY_CONSENT_STATES = [
  'CA', 'CT', 'FL', 'IL', 'MD', 'MA', 'MI', 'MT', 'NV', 'NH', 'PA', 'WA'
];

export const twoPartyConsentPlugin: PatternPlugin = {
  name: 'two-party-consent-escalation',
  apply(issues, metadata) {
    if (!metadata.jurisdiction) return issues;
    const state = metadata.jurisdiction.state;
    if (TWO_PARTY_CONSENT_STATES.includes(state)) {
      return issues.map(issue =>
        issue.ruleId === 'PUB_OPS_CALL_RECORDING_CONSENT'
          ? { ...issue, severity: 'CRITICAL', weight: 40, escalationReason: `Two-party consent state: ${state}` }
          : issue
      );
    }
    return issues;
  }
};
```

---

## 9. Mark Review — LOCKED (Directive #35)

All 5 judgment calls resolved:

| # | Rule | Decision | Implementation |
|---|------|----------|----------------|
| 1 | PUB_FIN_PAYMENT_TERMS_SLOW (#1) | Horizontal + per-vertical threshold | PPC threshold = 15d (flag >15). Other verticals: HVAC=45d, attorney=60d. |
| 2 | PUB_OPS_NON_CIRCUMVENT_BROAD (#37) | Move to horizontal | Brokerage pattern not PPC-unique. PPC overrides severity HIGH→CRITICAL. |
| 3 | BUY_DATA_OWNERSHIP_UNCLEAR (#72) | Keep horizontal | Default severity HIGH. PPC overrides to CRITICAL via `PPC_SEVERITY_OVERRIDES`. |
| 4 | BUY_FIN_PREPAY_OR_DEPOSIT_NO_REFUND (#57) | Keep horizontal | No change. Correctly classified. |
| 5 | 6 milo-ops novel rules | All canonical | #18/#43/#44/#45 = horizontal. #8/#9 = PPC. |

**No-score architectural directive:**
- Engine retains scoring (useful for other verticals)
- Milo-for-PPC: `displayAggregateScore: false`, `displayLetterGrade: false`, `displayRiskBucket: false`
- Per-issue severity still displayed (operators need per-issue triage)
- Issues rendered ordered by severity (CRITICAL first)

---

## 10. Implementation Sequence for Coder-1

1. Add `RuleDefinition` interface to `@milo/contract-analysis` types
2. Create `packages/contract-analysis/src/rules/base-registry.ts` (28 rules)
3. Export `BASE_RULES` from package entry point
4. In Milo-for-PPC: create `src/lib/ppc/rules.ts` (49 rules)
5. Wire: `createAnalysisEngine({ rules: [...BASE_RULES, ...PPC_RULES], ... })`
6. Add `mandatoryChecks` enforcement to post-processing pipeline
7. Add `twoPartyConsentPlugin` as PatternPlugin
8. Add scoring config with PPC_PROFILES
9. Test: run engine against 3-5 sample contracts, verify rule coverage

---

## Summary

| Deliverable | Count | Owner | Status |
|-------------|-------|-------|--------|
| Horizontal base rules | 45 | Coder-1 (in @milo/contract-analysis v0.2) | LOCKED |
| PPC vertical rules | 32 | Coder-1 (in Milo-for-PPC) | LOCKED |
| PPC severity overrides | 2 rules escalated | Coder-1 | LOCKED |
| PPC threshold overrides | 1+ rules (PPC-specific triggers) | Coder-1 | LOCKED |
| Mandatory checks list | 17 | From milo-ops prompt | LOCKED |
| PatternPlugins | 2 (two-party consent, sneaked-in detection) | Coder-1 | LOCKED |
| ScoringModifiers | 4 profiles (TEST/SCALE/BROKER/BUYER) | Coder-1 | LOCKED |
| No-score display config | Milo-for-PPC vertical config | Coder-1 | LOCKED |
| Discarded | 2 legacy + 6 phantom (if found) | — | — |
