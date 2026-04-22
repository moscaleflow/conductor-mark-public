---
directive: "Milo-for-PPC vertical rule pack spec"
lane: research
coder: Coder-3
started: 2026-04-20 ~15:45 MDT
completed: 2026-04-20 ~17:00 MDT
supplements:
  - reports/2026-04-20-coder-3-ppc-rules-dedup-mapping.md (254794c)
  - reports/2026-04-20-coder-3-contract-analysis-v02-spec.md (a338922)
  - ~/Miloops3.18/milo-ops/src/lib/contract-analysis-prompt.ts (source)
  - ~/MOP/lib/contract-checker/rules/registry.ts (source)
  - ~/MOP/lib/contract-checker/issue-postprocessing.ts (source)
  - ~/MOP/lib/contract-checker/profiles.ts (source)
  - ~/contract-checker/src/lib/playbook-rules.ts (source)
---

# Milo-for-PPC Vertical Rule Pack Spec

## 1. Executive Summary

| Metric | Value |
|--------|-------|
| PPC vertical rules | 32 (full RuleDefinition[] code below) |
| Horizontal severity overrides | 2 rules escalated |
| Horizontal threshold overrides | 1 rule (PPC-specific trigger) |
| Mandatory checks | 17 rule IDs |
| PatternPlugins | 3 (two-party consent, catch-all injection, indemnity cap guardrail) |
| Estimated LOC | ~1,800 across 6 files |
| Files | `src/lib/ppc/{rules,overrides,mandatory,patterns,display-config,index}.ts` |

**Count correction:** The dedup mapping summary (254794c) states "32 horizontal, 45 PPC-specific." The actual master table counts are **45 horizontal + 32 PPC = 77 total**. The summary numbers were transposed. This spec follows the master table (each row's Layer column is correct). The v0.2 spec BASE_RULES listing also contains 45 rules (9+9+9+5+6+7), confirming the correct split.

---

## 2. Full 32 PPC Rules as RuleDefinition[]

### File: `src/lib/ppc/rules.ts`

```typescript
import type { RuleDefinition } from '@milo/contract-analysis';

/**
 * 32 PPC-specific rules for the Milo-for-PPC vertical.
 * These rules address pay-per-call / lead-gen industry concerns
 * that do not generalize to other B2B contract verticals.
 *
 * Registered alongside BASE_RULES (45 horizontal) via VerticalRulePack.
 */
export const PPC_RULES: RuleDefinition[] = [

  // ═══════════════════════════════════════════════════════════════
  // PPC PUBLISHER FINANCIAL (7 rules)
  // ═══════════════════════════════════════════════════════════════

  {
    rule_id: 'PUB_FIN_FINALITY_MISSING',
    name: 'Lead/Call Finality Missing',
    description: 'No defined rejection window or explicit finality after window closes for leads/calls',
    category: 'FIN',
    severity: 'CRITICAL',
    weight: 40,
    isExistential: true,
    applies_to: 'publisher',
    tags: ['payment', 'finality', 'rejection', 'existential'],
    keywords: ['deemed final', 'not subject to future rejection', 'final and billable', 'rejection window', 'acceptance period'],
    prompt_instruction: 'Is there a defined rejection window AND explicit finality after the window closes? Look for "deemed final", "not subject to future rejection", "final and billable". If missing or unclear: flag as CRITICAL. IMPORTANT: While reading the rejection clause for finality, ALSO check for catch-all rejection reasons separately (PUB_FIN_REJECTION_CATCH_ALL). Finality and catch-all are TWO different problems — always flag both if both exist.',
    plainEnglishHint: 'They can reject your calls or leads at any time — even months later — and claw back money you already earned.',
    impactTemplate: 'This affects {company} because without finality, {counterparty} can retroactively reject calls/leads and recoup payments indefinitely.',
    recommendationFormat: 'PUSH_BACK',
    riderRefs: ['Finality Rider §1'],
  },

  {
    rule_id: 'PUB_FIN_REPORTING_BINDING_NO_AUDIT',
    name: 'Binding Reporting Without Audit Rights',
    description: 'Advertiser reporting is binding or sole source of truth without audit/dispute rights',
    category: 'FIN',
    severity: 'CRITICAL',
    weight: 35,
    isExistential: true,
    applies_to: 'publisher',
    tags: ['payment', 'reporting', 'audit', 'existential'],
    keywords: ['binding', 'sole source of truth', 'reporting', 'audit rights', 'dispute right'],
    prompt_instruction: 'If advertiser reporting is "binding" or sole source of truth, is there an audit/dispute right and documentation requirement? If binding without audit rights: flag as CRITICAL.',
    plainEnglishHint: 'Their numbers are the only numbers that matter — and you can\'t check or challenge them.',
    impactTemplate: 'This affects {company} because {counterparty}\'s tracking system becomes the sole arbiter of call/lead counts, with no mechanism to dispute undercounts.',
    recommendationFormat: 'PUSH_BACK',
    riderRefs: ['Audit Rights Rider §2'],
  },

  {
    rule_id: 'PUB_FIN_HOLDBACK_NO_RELEASE',
    name: 'Holdback Without Release Timeline',
    description: 'Holdbacks or reserves on call/lead revenue without a defined release date',
    category: 'FIN',
    severity: 'HIGH',
    weight: 20,
    isExistential: false,
    applies_to: 'publisher',
    tags: ['payment', 'holdback', 'reserve'],
    keywords: ['holdback', 'reserve', 'retention', 'withheld', 'escrow', 'release timeline'],
    prompt_instruction: 'Are holdbacks/reserves allowed without a release timeline? If no release deadline specified: flag as HIGH/CRITICAL depending on scope.',
    plainEnglishHint: 'They can hold back a chunk of your money indefinitely — no deadline to return it.',
    impactTemplate: 'This affects {company} because {counterparty} can withhold revenue reserves with no obligation to release them by a specific date.',
    recommendationFormat: 'PUSH_BACK',
    threshold: { strict: 5, standard: 10, relaxed: 15 },
  },

  {
    rule_id: 'PUB_FIN_CLAWBACK_OPEN_ENDED',
    name: 'Open-Ended Clawback',
    description: 'Recoupment, chargebacks, or offsets without strict time limits',
    category: 'FIN',
    severity: 'CRITICAL',
    weight: 40,
    isExistential: true,
    applies_to: 'publisher',
    tags: ['payment', 'clawback', 'chargeback', 'existential'],
    keywords: ['clawback', 'reversal', 'chargeback', 'recoupment', 'offset', 'withholding', 'sole discretion'],
    prompt_instruction: 'Any clause allowing recoupment, reclaiming, chargebacks, offsets, or withholding without strict time limits and documentation? If open-ended or "at any time" or "sole discretion": flag as CRITICAL.',
    plainEnglishHint: 'They can reach back and take money from you at any time, for any reason, with no deadline.',
    impactTemplate: 'This affects {company} because {counterparty} can unilaterally reverse payments already made — even months or years later — with no time limit or documentation requirement.',
    recommendationFormat: 'PUSH_BACK',
    riderRefs: ['Clawback Limitation Rider §3'],
    threshold: { strict: 7, standard: 14, relaxed: 30 },
  },

  {
    rule_id: 'PUB_FIN_REJECTION_CATCH_ALL',
    name: 'Subjective Rejection Criteria',
    description: 'Open-ended catch-all rejection language allowing rejection for any reason',
    category: 'FIN',
    severity: 'HIGH',
    weight: 25,
    isExistential: false,
    applies_to: 'publisher',
    tags: ['payment', 'rejection', 'catch-all', 'subjective'],
    keywords: ['any other reason', 'calls into question', 'sole discretion', 'deemed appropriate', 'any reason whatsoever', 'legitimacy or appropriateness'],
    prompt_instruction: 'Read each listed rejection reason one by one. Look for: "(f) any other reason(s) that call into question the legitimacy or appropriateness", "in Company\'s sole discretion", "any other reason deemed appropriate", "for any reason whatsoever". ANY open-ended, subjective catch-all = flag as HIGH with title "Subjective Rejection Criteria". This is ALWAYS a separate issue from finality (PUB_FIN_FINALITY_MISSING) — NEVER combine them.',
    plainEnglishHint: 'They can reject your calls for basically any reason they want — the catch-all language makes the specific criteria meaningless.',
    impactTemplate: 'This affects {company} because even if your calls meet every listed quality standard, the catch-all clause lets {counterparty} reject them anyway on subjective grounds.',
    recommendationFormat: 'PUSH_BACK',
  },

  {
    rule_id: 'PUB_FIN_INVOICE_FORFEITURE',
    name: 'Automatic Invoice Forfeiture',
    description: 'Failure to invoice within X days results in automatic waiver of payment rights',
    category: 'FIN',
    severity: 'HIGH',
    weight: 20,
    isExistential: false,
    applies_to: 'publisher',
    tags: ['payment', 'invoice', 'forfeiture', 'deadline'],
    keywords: ['invoice', 'waiver', 'forfeiture', 'submit within', 'lose right to payment', 'automatic waiver'],
    prompt_instruction: 'Does the contract state that failure to submit an invoice within X days results in automatic waiver or forfeiture of payment rights? If automatic forfeiture for late invoicing: flag as HIGH. Title MUST be "Automatic Invoice Forfeiture". This is SEPARATE from finality/rejection window — this is about losing the right to bill at all.',
    plainEnglishHint: 'If you\'re even a day late sending your invoice, you lose the right to get paid — period.',
    impactTemplate: 'This affects {company} because a missed invoice deadline means {counterparty} owes nothing, regardless of calls delivered and accepted.',
    recommendationFormat: 'PUSH_BACK',
  },

  {
    rule_id: 'PUB_TERM_TRAILING_MISSING',
    name: 'Trailing Commission Missing',
    description: 'No ongoing revenue from customers acquired during contract term',
    category: 'TERM',
    severity: 'MEDIUM',
    weight: 12,
    isExistential: false,
    applies_to: 'publisher',
    tags: ['termination', 'trailing', 'commission', 'revenue'],
    keywords: ['trailing commission', 'residual', 'lifetime value', 'recurring revenue', 'ongoing payment'],
    prompt_instruction: 'Does the contract address trailing commissions or ongoing revenue from customers acquired during the term? If no provision for post-term revenue on acquired customers: flag as MEDIUM.',
    plainEnglishHint: 'When the contract ends, you stop getting paid — even for customers you brought in who are still generating revenue for them.',
    impactTemplate: 'This affects {company} because customers acquired through {company}\'s calls continue generating revenue for {counterparty} after termination, with no trailing payments to {company}.',
    recommendationFormat: 'ACCEPT_WITH_MODIFICATION',
  },

  // ═══════════════════════════════════════════════════════════════
  // PPC TCPA & COMPLIANCE (9 rules)
  // ═══════════════════════════════════════════════════════════════

  {
    rule_id: 'PUB_COMP_TCPA_STRICT_SHIFT',
    name: 'TCPA Liability Shift',
    description: 'TCPA liability shifted to publisher regardless of downstream dialing practices',
    category: 'COMP',
    severity: 'CRITICAL',
    weight: 40,
    isExistential: true,
    applies_to: 'publisher',
    tags: ['tcpa', 'compliance', 'liability', 'existential'],
    keywords: ['TCPA', 'Telephone Consumer Protection', 'strict liability', 'liability shift', 'do not call', 'DNC'],
    prompt_instruction: 'Is TCPA liability shifted to Publisher regardless of downstream dialing practices? If strict liability shift: flag as CRITICAL.',
    plainEnglishHint: 'If anyone in the call chain violates TCPA, you\'re on the hook — even if you had nothing to do with it.',
    impactTemplate: 'This affects {company} because TCPA violations carry $500-$1,500 per call in statutory damages, and this clause makes {company} liable even for downstream actions it cannot control.',
    recommendationFormat: 'PUSH_BACK',
  },

  {
    rule_id: 'PUB_COMP_TCPA_RECORD_BURDEN',
    name: 'Excessive Record Retention',
    description: 'Excessive consent record retention period beyond TCPA statute of limitations',
    category: 'COMP',
    severity: 'HIGH',
    weight: 20,
    isExistential: false,
    applies_to: 'publisher',
    tags: ['tcpa', 'compliance', 'records', 'retention'],
    keywords: ['record retention', 'consent records', 'retain for', 'years', 'storage'],
    isRecordingRelated: true,
    prompt_instruction: 'Excessive retention periods? (e.g., 5-6+ years for consent records). If >4 years: flag as HIGH. The TCPA statute of limitations is 4 years — retention beyond that is burdensome without legal justification.',
    plainEnglishHint: 'They want you to store call consent records for longer than the law requires — costing you storage and compliance overhead.',
    impactTemplate: 'This affects {company} because maintaining consent records beyond the 4-year TCPA SOL creates unnecessary storage costs and compliance burden.',
    recommendationFormat: 'ACCEPT_WITH_MODIFICATION',
  },

  {
    rule_id: 'PUB_COMP_TCPA_RECORD_PRODUCTION_FAST',
    name: 'Fast Record Production (3-10 days)',
    description: 'Consent record production required within 3-10 business days — operationally difficult',
    category: 'COMP',
    severity: 'HIGH',
    weight: 25,
    isExistential: false,
    applies_to: 'publisher',
    tags: ['tcpa', 'compliance', 'records', 'production', 'deadline'],
    keywords: ['business days', 'produce records', 'provide records', 'consent records', 'upon request'],
    isRecordingRelated: true,
    prompt_instruction: 'Record production within 3-10 business days: flag as HIGH (operationally difficult). Check the specific timeline required for producing consent records, call recordings, or compliance documentation upon request.',
    plainEnglishHint: 'They want you to produce years of call records in under two weeks — that\'s a scramble even with good systems.',
    impactTemplate: 'This affects {company} because producing consent records across multiple systems within 3-10 business days is operationally difficult, especially for historical records.',
    recommendationFormat: 'ACCEPT_WITH_MODIFICATION',
  },

  {
    rule_id: 'PUB_COMP_TCPA_RECORD_PRODUCTION_EXTREME',
    name: 'Extreme Record Production (<=2 days)',
    description: 'Consent record production required within 2 business days — impossible to comply',
    category: 'COMP',
    severity: 'CRITICAL',
    weight: 40,
    isExistential: true,
    applies_to: 'publisher',
    tags: ['tcpa', 'compliance', 'records', 'production', 'deadline', 'existential'],
    keywords: ['2 business days', '48 hours', 'immediately', 'within 1 day', 'upon demand'],
    isRecordingRelated: true,
    prompt_instruction: 'Record production within <=2 business days: flag as CRITICAL (impossible to comply). This timeline is unrealistic for consent record retrieval across distributed systems.',
    plainEnglishHint: 'Two days to produce all your call consent records? That\'s physically impossible — and failing means you\'re in breach.',
    impactTemplate: 'This affects {company} because a 2-day production deadline is operationally impossible for distributed consent records, creating automatic breach exposure.',
    recommendationFormat: 'PUSH_BACK',
  },

  {
    rule_id: 'BUY_COMP_TCPA_WARRANTY_MISSING',
    name: 'TCPA Warranty Missing',
    description: 'Publisher does not warrant TCPA/TSR/DNC compliance and proper consent',
    category: 'COMP',
    severity: 'CRITICAL',
    weight: 40,
    isExistential: true,
    applies_to: 'buyer',
    tags: ['tcpa', 'compliance', 'warranty', 'consent', 'existential'],
    keywords: ['TCPA', 'TSR', 'DNC', 'warrant', 'compliance', 'consent', 'prior express'],
    isRecordingRelated: true,
    prompt_instruction: 'Does publisher warrant TCPA/TSR/DNC compliance and proper consent? If missing: flag as CRITICAL — existential. Without this warranty, buyer has no contractual basis to hold publisher liable for TCPA violations originating from publisher\'s calls.',
    plainEnglishHint: 'The publisher isn\'t promising their calls are legal — if they\'re not, you\'re the one getting sued.',
    impactTemplate: 'This affects {company} because without a TCPA compliance warranty, {company} absorbs all regulatory exposure for calls generated by {counterparty}\'s operations.',
    recommendationFormat: 'PUSH_BACK',
  },

  {
    rule_id: 'BUY_COMP_TCPA_INDEMNITY_WEAK',
    name: 'Weak TCPA Indemnity',
    description: 'Publisher indemnification for TCPA violations is weak or missing',
    category: 'COMP',
    severity: 'CRITICAL',
    weight: 35,
    isExistential: false,
    applies_to: 'buyer',
    tags: ['tcpa', 'compliance', 'indemnity'],
    keywords: ['indemnify', 'TCPA', 'hold harmless', 'defend', 'violations'],
    prompt_instruction: 'Will publisher indemnify buyer for publisher-caused TCPA violations? If weak or missing: flag as CRITICAL. The indemnity should specifically cover TCPA/TSR violations, not just generic breach.',
    plainEnglishHint: 'If the publisher\'s calls violate TCPA and you get sued, they won\'t cover your legal costs.',
    impactTemplate: 'This affects {company} because TCPA class actions can reach millions in damages, and without publisher indemnification, {company} bears the full cost of violations it didn\'t cause.',
    recommendationFormat: 'PUSH_BACK',
  },

  {
    rule_id: 'BUY_COMP_RECORD_RETENTION_TOO_SHORT',
    name: 'Record Retention Too Short',
    description: 'Consent record retention shorter than TCPA statute of limitations (4 years)',
    category: 'COMP',
    severity: 'HIGH',
    weight: 28,
    isExistential: false,
    applies_to: 'buyer',
    tags: ['tcpa', 'compliance', 'records', 'retention'],
    keywords: ['retain', 'records', 'consent', 'years', 'destroy', 'delete'],
    isRecordingRelated: true,
    prompt_instruction: 'How long must publisher retain consent records? If <4 years: flag as HIGH. TCPA statute of limitations is 4 years — records destroyed earlier leave buyer unable to defend claims.',
    plainEnglishHint: 'They\'re allowed to delete the proof that calls were legal before the window for lawsuits closes.',
    impactTemplate: 'This affects {company} because consent records destroyed before the 4-year TCPA SOL leave {company} unable to mount a defense in class actions.',
    recommendationFormat: 'PUSH_BACK',
  },

  {
    rule_id: 'BUY_COMP_RECORD_PRODUCTION_UNREALISTIC',
    name: 'Record Production Unrealistic',
    description: 'Unrealistic timelines for buyer to produce compliance records',
    category: 'COMP',
    severity: 'HIGH',
    weight: 30,
    isExistential: false,
    applies_to: 'buyer',
    tags: ['compliance', 'records', 'production'],
    keywords: ['produce', 'provide records', 'business days', 'upon request', 'compliance records'],
    isRecordingRelated: true,
    prompt_instruction: 'Are record production timelines realistic for the buyer? If buyer must produce compliance documentation in unreasonably short periods: flag as HIGH.',
    plainEnglishHint: 'The timeline for producing compliance records is so tight you\'d have to drop everything else to meet it.',
    impactTemplate: 'This affects {company} because unrealistic production timelines create automatic breach exposure, regardless of actual compliance.',
    recommendationFormat: 'ACCEPT_WITH_MODIFICATION',
  },

  {
    rule_id: 'BUY_COMP_SUBAFFILIATE_COMPLIANCE_GAP',
    name: 'Subaffiliate Compliance Gap',
    description: 'Sub-affiliates used without compliance obligations flowing down',
    category: 'COMP',
    severity: 'CRITICAL',
    weight: 35,
    isExistential: true,
    applies_to: 'buyer',
    tags: ['compliance', 'subaffiliate', 'flow-down', 'existential'],
    keywords: ['sub-affiliate', 'subaffiliate', 'sub-publisher', 'downstream', 'flow-down', 'third party'],
    prompt_instruction: 'Does publisher use sub-affiliates? Are compliance obligations flowed down? If no flow-down: flag as CRITICAL. Without compliance flow-down, sub-affiliates can generate non-compliant calls that expose the buyer.',
    plainEnglishHint: 'The publisher uses sub-affiliates who aren\'t bound by the same rules — and their bad calls become your problem.',
    impactTemplate: 'This affects {company} because sub-affiliate calls without compliance flow-down create TCPA exposure that {counterparty} has contractually disclaimed responsibility for.',
    recommendationFormat: 'PUSH_BACK',
  },

  // ═══════════════════════════════════════════════════════════════
  // PPC OPERATIONAL (6 rules)
  // ═══════════════════════════════════════════════════════════════

  {
    rule_id: 'PUB_OPS_CALL_RECORDING_CONSENT',
    name: 'Call Recording Consent Issues',
    description: 'Call recording provisions without two-party consent state compliance',
    category: 'OPS',
    severity: 'HIGH',
    weight: 20,
    isExistential: false,
    applies_to: 'both',
    tags: ['operational', 'recording', 'consent', 'two-party'],
    keywords: ['call recording', 'recorded calls', 'recording consent', 'monitor calls', 'two-party', 'all-party'],
    isRecordingRelated: true,
    jurisdictionTriggers: ['CA', 'CT', 'FL', 'IL', 'MD', 'MA', 'MI', 'MT', 'NV', 'NH', 'PA', 'WA'],
    prompt_instruction: 'Does the contract address call recording consent? In two-party consent states (CA, CT, FL, IL, MD, MA, MI, MT, NV, NH, PA, WA), recording without all-party consent is a criminal offense. Flag if the contract requires recording in these states without explicit consent provisions.',
    plainEnglishHint: 'Recording calls in certain states without everyone\'s consent is a crime — and this contract doesn\'t handle that properly.',
    impactTemplate: 'This affects {company} because call recording in two-party consent states without proper consent exposes {company} to criminal liability and wiretapping claims.',
    recommendationFormat: 'PUSH_BACK',
  },

  {
    rule_id: 'PUB_OPS_BROKERING_BAN',
    name: 'Brokering Prohibition',
    description: 'Prohibition on lead/call brokering that restricts normal business operations',
    category: 'OPS',
    severity: 'HIGH',
    weight: 20,
    isExistential: false,
    applies_to: 'publisher',
    tags: ['operational', 'brokering', 'restriction'],
    keywords: ['broker', 'brokering', 'resell', 'redistribute', 'sub-contract', 'third party'],
    prompt_instruction: 'Does the contract prohibit or restrict lead/call brokering? If brokering is banned outright with no exceptions for standard distribution: flag as HIGH. Brokering restrictions can prevent normal publisher operations in the pay-per-call ecosystem.',
    plainEnglishHint: 'You can\'t use intermediaries to route calls — which is how most of the PPC industry actually works.',
    impactTemplate: 'This affects {company} because a brokering ban prevents {company} from using standard call routing intermediaries and distribution networks.',
    recommendationFormat: 'ACCEPT_WITH_MODIFICATION',
  },

  {
    rule_id: 'BUY_OPS_NO_CONSENT_PROOF_ACCESS',
    name: 'No Access to Consent Proof',
    description: 'Buyer cannot access Jornaya/TrustedForm logs, consent language, or landing pages',
    category: 'OPS',
    severity: 'CRITICAL',
    weight: 40,
    isExistential: true,
    applies_to: 'buyer',
    tags: ['operational', 'consent', 'proof', 'jornaya', 'trustedform', 'existential'],
    keywords: ['Jornaya', 'TrustedForm', 'consent proof', 'consent records', 'landing page', 'consent language', 'verification'],
    isRecordingRelated: true,
    prompt_instruction: 'Can buyer get Jornaya/TrustedForm logs, consent language, landing pages on request? If refused or unreasonable timelines: flag as CRITICAL. Without access to consent proof, buyer cannot independently verify TCPA compliance or defend against class actions.',
    plainEnglishHint: 'You can\'t see proof that callers actually consented — so when a TCPA lawsuit hits, you have no evidence to defend yourself.',
    impactTemplate: 'This affects {company} because without access to consent proof (Jornaya, TrustedForm), {company} cannot independently verify compliance or mount a defense in TCPA litigation.',
    recommendationFormat: 'PUSH_BACK',
  },

  {
    rule_id: 'BUY_OPS_NO_MINIMUM_VOLUME',
    name: 'No Minimum Volume Commitment',
    description: 'No guaranteed minimum lead/call volume from publisher',
    category: 'OPS',
    severity: 'MEDIUM',
    weight: 22,
    isExistential: false,
    applies_to: 'buyer',
    tags: ['operational', 'volume', 'commitment', 'minimum'],
    keywords: ['minimum volume', 'guaranteed volume', 'commitment', 'minimum calls', 'minimum leads'],
    prompt_instruction: 'Does the contract include minimum volume commitments from the publisher? If no minimum volume guarantee: flag as MEDIUM. Without volume commitments, buyer has no recourse if publisher diverts traffic to competitors.',
    plainEnglishHint: 'They don\'t promise to send you any minimum number of calls — they could send zero and still be in compliance.',
    impactTemplate: 'This affects {company} because without volume commitments, {counterparty} can redirect traffic to competitors while {company} maintains infrastructure capacity for calls that never come.',
    recommendationFormat: 'ACCEPT_WITH_MODIFICATION',
  },

  {
    rule_id: 'BUY_OPS_BROKERING_ALLOWED_UNDISCLOSED',
    name: 'Undisclosed Brokering Allowed',
    description: 'Sub-affiliate or brokering allowed without disclosure or approval',
    category: 'OPS',
    severity: 'HIGH',
    weight: 28,
    isExistential: false,
    applies_to: 'buyer',
    tags: ['operational', 'brokering', 'sub-affiliate', 'disclosure'],
    keywords: ['broker', 'sub-affiliate', 'undisclosed', 'third party', 'without approval', 'without consent'],
    prompt_instruction: 'Can the publisher use intermediaries, sub-affiliates, or brokers without disclosing or getting buyer approval? If undisclosed brokering is allowed: flag as HIGH. Buyer needs to know the actual source of calls for quality control and compliance.',
    plainEnglishHint: 'They can hand your campaign off to unknown sub-affiliates without telling you — and you won\'t know where your calls are really coming from.',
    impactTemplate: 'This affects {company} because undisclosed sub-affiliates bypass {company}\'s quality controls and compliance verification, creating hidden exposure.',
    recommendationFormat: 'PUSH_BACK',
  },

  {
    rule_id: 'BUY_OPS_CALL_TRANSFER_NO_REJECT',
    name: 'Call Transfer Rejection Prohibited',
    description: 'Buyer cannot reject, dispute, or disqualify bad call transfers',
    category: 'OPS',
    severity: 'CRITICAL',
    weight: 40,
    isExistential: true,
    applies_to: 'buyer',
    tags: ['operational', 'call transfer', 'rejection', 'quality', 'existential'],
    keywords: ['disqualification requests', 'not permitted', 'may not reject', 'all transferred calls are final', 'no disputes on live transfers'],
    prompt_instruction: 'Does the contract prohibit or severely limit the buyer\'s ability to reject, dispute, or disqualify call transfers? Look for: "disqualification requests of transfer calls are not permitted", "buyer may not reject transferred calls", "all transferred calls are final", "no disputes on live transfers". If buyer CANNOT reject bad call transfers (except narrow exceptions like outside operating hours): flag as CRITICAL. This is a major cost trap — buyer pays for every call regardless of quality.',
    plainEnglishHint: 'You have to pay for every single call transfer — even garbage calls that don\'t meet any quality standard. No disputes allowed.',
    impactTemplate: 'This affects {company} because paying for all call transfers regardless of quality, caller qualification, or compliance turns every bad call into a direct cost with no remedy.',
    recommendationFormat: 'PUSH_BACK',
  },

  // ═══════════════════════════════════════════════════════════════
  // PPC BUYER FINANCIAL (7 rules)
  // ═══════════════════════════════════════════════════════════════

  {
    rule_id: 'BUY_FIN_QUALITY_STANDARDS_VAGUE',
    name: 'Vague Lead Quality Standards',
    description: 'No clear, objective acceptance criteria for leads/calls',
    category: 'FIN',
    severity: 'HIGH',
    weight: 30,
    isExistential: false,
    applies_to: 'buyer',
    tags: ['payment', 'quality', 'acceptance', 'criteria'],
    keywords: ['quality standard', 'lead quality', 'call quality', 'acceptance criteria', 'eligible', 'qualified'],
    prompt_instruction: 'Are there clear, objective acceptance criteria? (duplicate handling, fraud checks, consent proof, eligibility, geo, duration, etc.) If vague or undefined: flag as HIGH/CRITICAL.',
    plainEnglishHint: 'There\'s no clear definition of what counts as a "good" call — so when you try to reject a bad one, they\'ll say it met the standard.',
    impactTemplate: 'This affects {company} because without objective quality criteria, disputes over call/lead quality become he-said-she-said with no contractual anchor.',
    recommendationFormat: 'PUSH_BACK',
  },

  {
    rule_id: 'BUY_FIN_NO_REJECTION_WINDOW',
    name: 'No Rejection Window',
    description: 'Missing or too-short rejection window for validating leads/calls',
    category: 'FIN',
    severity: 'CRITICAL',
    weight: 40,
    isExistential: true,
    applies_to: 'buyer',
    tags: ['payment', 'rejection', 'window', 'validation', 'existential'],
    keywords: ['rejection window', 'dispute period', 'acceptance period', 'business days', 'reject within'],
    prompt_instruction: 'Is there a defined rejection window? Is it long enough to validate leads operationally? If missing or too short (<7 days): flag as CRITICAL.',
    plainEnglishHint: 'You have almost no time to check if calls are valid before you\'re locked into paying for them.',
    impactTemplate: 'This affects {company} because without adequate rejection time, {company} must pay for calls before it can verify quality, consent, or eligibility.',
    recommendationFormat: 'PUSH_BACK',
  },

  {
    rule_id: 'BUY_FIN_NO_REPLACEMENT_OR_CREDIT',
    name: 'No Replacement or Credit Remedy',
    description: 'No remedy for rejected leads — no replacement, credit, or refund',
    category: 'FIN',
    severity: 'CRITICAL',
    weight: 40,
    isExistential: true,
    applies_to: 'buyer',
    tags: ['payment', 'remedy', 'replacement', 'credit', 'refund', 'existential'],
    keywords: ['replacement', 'credit', 'refund', 'remedy', 'rejected leads', 'bad calls'],
    prompt_instruction: 'What remedy exists for rejected leads? Replacement, credit, or refund? If no remedy: flag as CRITICAL.',
    plainEnglishHint: 'When you get bad calls, you have no remedy — no replacement, no credit, no refund. You just eat the cost.',
    impactTemplate: 'This affects {company} because without a remedy mechanism, every bad call is a direct unrecoverable cost.',
    recommendationFormat: 'PUSH_BACK',
  },

  {
    rule_id: 'BUY_FIN_REJECTION_PROCESS_NOT_DEFINED',
    name: 'Rejection Process Undefined',
    description: 'No documented process for how to submit rejections',
    category: 'FIN',
    severity: 'HIGH',
    weight: 30,
    isExistential: false,
    applies_to: 'buyer',
    tags: ['payment', 'rejection', 'process', 'documentation'],
    keywords: ['rejection process', 'how to reject', 'dispute process', 'submit rejection', 'evidence required'],
    prompt_instruction: 'Is the rejection process documented (how to submit, evidence required, dispute timeline)? If undefined: flag as HIGH.',
    plainEnglishHint: 'There\'s no defined process for rejecting bad calls — so when you try, they can claim you didn\'t follow the right procedure.',
    impactTemplate: 'This affects {company} because without a documented rejection process, {counterparty} can dismiss rejections on procedural grounds.',
    recommendationFormat: 'PUSH_BACK',
  },

  {
    rule_id: 'BUY_FIN_CHARGEBACK_RIGHTS_TOO_NARROW',
    name: 'Chargeback Rights Too Narrow',
    description: 'Chargeback or clawback rights too narrow for lead/call quality issues',
    category: 'FIN',
    severity: 'HIGH',
    weight: 32,
    isExistential: false,
    applies_to: 'buyer',
    tags: ['payment', 'chargeback', 'clawback', 'quality'],
    keywords: ['chargeback', 'quality deduction', 'fraud liability', 'refund responsibility', 'dispute right'],
    prompt_instruction: 'Are chargeback/clawback rights adequate for addressing lead quality issues? If chargeback rights are too narrow (e.g., only for fraud, not quality failures): flag as HIGH.',
    plainEnglishHint: 'You can only dispute calls in very narrow circumstances — most quality problems don\'t qualify for a chargeback.',
    impactTemplate: 'This affects {company} because narrow chargeback rights mean {company} absorbs the cost of low-quality leads that fall outside the narrow dispute criteria.',
    recommendationFormat: 'ACCEPT_WITH_MODIFICATION',
  },

  {
    rule_id: 'BUY_FIN_PAYMENT_TERMS_TOO_FAST',
    name: 'Payment Due Before Validation',
    description: 'Payment required before buyer can validate lead/call quality',
    category: 'FIN',
    severity: 'MEDIUM',
    weight: 30,
    isExistential: false,
    applies_to: 'buyer',
    tags: ['payment', 'terms', 'validation', 'timing'],
    keywords: ['payment terms', 'net 7', 'due upon receipt', 'immediate payment', 'prepay'],
    prompt_instruction: 'Are payment terms so fast that buyer must pay before lead validation is complete? If payment due before quality validation window closes: flag as MEDIUM.',
    plainEnglishHint: 'You have to pay for the calls before you even finish checking if they\'re any good.',
    impactTemplate: 'This affects {company} because paying before validation completes means {company} funds non-compliant calls with no pre-payment quality gate.',
    recommendationFormat: 'ACCEPT_WITH_MODIFICATION',
  },

  {
    rule_id: 'BUY_FIN_REPORTING_NOT_BINDING',
    name: 'Reporting Non-Binding',
    description: 'Publisher reporting is "informational only" with no recourse for buyer',
    category: 'FIN',
    severity: 'HIGH',
    weight: 32,
    isExistential: false,
    applies_to: 'buyer',
    tags: ['payment', 'reporting', 'non-binding', 'recourse'],
    keywords: ['informational only', 'non-binding', 'reporting', 'no recourse', 'best efforts'],
    prompt_instruction: 'Is reporting "informational only" or must buyer rely solely on publisher records? If non-binding with no recourse: flag as HIGH.',
    plainEnglishHint: 'Their call reports are just "informational" — even if the numbers don\'t add up, you have no contractual basis to challenge them.',
    impactTemplate: 'This affects {company} because non-binding reporting means {counterparty}\'s numbers are the only truth, and {company} has no contractual basis to dispute discrepancies.',
    recommendationFormat: 'PUSH_BACK',
  },

  // ═══════════════════════════════════════════════════════════════
  // PPC BUYER DATA & LIABILITY (3 rules)
  // ═══════════════════════════════════════════════════════════════

  {
    rule_id: 'BUY_DATA_LICENSE_BACK_TO_PUBLISHER',
    name: 'Publisher Retains Data License',
    description: 'Publisher retains rights to reuse or resell leads buyer paid for',
    category: 'DATA',
    severity: 'HIGH',
    weight: 30,
    isExistential: false,
    applies_to: 'buyer',
    tags: ['data', 'license', 'resale', 'exclusivity'],
    keywords: ['license', 'retain rights', 'reuse', 'resell', 'data rights', 'non-exclusive'],
    prompt_instruction: 'Does publisher retain rights to reuse/resell leads buyer paid for? If broad license retained: flag as HIGH.',
    plainEnglishHint: 'You paid for these leads, but the publisher can turn around and sell the same leads to your competitors.',
    impactTemplate: 'This affects {company} because paying for leads that {counterparty} can resell to competitors eliminates any competitive advantage from the acquisition.',
    recommendationFormat: 'PUSH_BACK',
  },

  {
    rule_id: 'BUY_DATA_EXCLUSIVITY_MISSING',
    name: 'Exclusivity Missing',
    description: 'No exclusive lead purchase — publisher can sell same leads to competitors',
    category: 'DATA',
    severity: 'MEDIUM',
    weight: 22,
    isExistential: false,
    applies_to: 'buyer',
    tags: ['data', 'exclusivity', 'competition'],
    keywords: ['exclusive', 'exclusivity', 'non-exclusive', 'shared leads', 'multi-buyer'],
    prompt_instruction: 'Does the contract provide for exclusive lead purchases? If no exclusivity clause and publisher can sell the same leads to multiple buyers: flag as MEDIUM.',
    plainEnglishHint: 'Your competitors are getting the exact same leads you are — there\'s no exclusivity.',
    impactTemplate: 'This affects {company} because non-exclusive leads are simultaneously sold to competitors, reducing conversion rates and increasing cost-per-acquisition.',
    recommendationFormat: 'ACCEPT_WITH_MODIFICATION',
  },

  {
    rule_id: 'BUY_LIAB_CAP_TOO_LOW_FOR_COMPLIANCE',
    name: 'Liability Cap Inadequate for Compliance Exposure',
    description: 'Liability cap too low relative to TCPA class action exposure',
    category: 'LIAB',
    severity: 'HIGH',
    weight: 32,
    isExistential: false,
    applies_to: 'buyer',
    tags: ['liability', 'cap', 'compliance', 'tcpa'],
    keywords: ['liability cap', 'maximum liability', 'aggregate liability', 'TCPA exposure'],
    prompt_instruction: 'Is there a liability cap? Is it adequate for compliance exposure? If cap is too low for TCPA exposure (e.g., capped at 1 month fees when TCPA exposure could be millions): flag as HIGH.',
    plainEnglishHint: 'The publisher\'s liability is capped so low that if they cause a TCPA problem, their maximum payout wouldn\'t cover a fraction of your legal costs.',
    impactTemplate: 'This affects {company} because a liability cap of {threshold} is meaningless against TCPA class action exposure of $500-$1,500 per call across thousands of calls.',
    recommendationFormat: 'PUSH_BACK',
  },

];
```

---

## 3. Severity Overrides on Horizontal Rules

### File: `src/lib/ppc/overrides.ts`

```typescript
import type { RiskLevel, ThresholdConfig } from '@milo/contract-analysis';

/**
 * PPC severity overrides for horizontal rules.
 * These escalate the default severity for rules where the PPC industry
 * context makes the risk more acute than in general B2B contracts.
 */
export const PPC_SEVERITY_OVERRIDES: Record<string, RiskLevel> = {
  // #72: Default HIGH → PPC CRITICAL
  // Lead/call data is existential in PPC — if seller owns ALL PII,
  // buyer cannot independently defend TCPA claims or verify consent.
  'BUY_DATA_OWNERSHIP_UNCLEAR': 'CRITICAL',

  // #37: Default HIGH → PPC CRITICAL
  // Non-circumvention in PPC/brokerage can destroy a publisher's
  // entire business model by locking them out of their own relationships.
  'PUB_OPS_NON_CIRCUMVENT_BROAD': 'CRITICAL',
};

/**
 * PPC threshold overrides for horizontal rules.
 * PPC industry operates on faster payment cycles than general B2B.
 */
export const PPC_THRESHOLD_OVERRIDES: Record<string, ThresholdConfig> = {
  // #1: Default strict:15/standard:30/relaxed:45 → PPC strict:10/standard:15/relaxed:30
  // PPC operators flag anything >15 days. Industry norm is Net 7 or Net 14.
  'PUB_FIN_PAYMENT_TERMS_SLOW': { strict: 10, standard: 15, relaxed: 30 },
};
```

### Override Rationale Table

| Rule ID | Horizontal Default | PPC Override | Rationale |
|---------|-------------------|-------------|-----------|
| `BUY_DATA_OWNERSHIP_UNCLEAR` | HIGH (weight 38) | **CRITICAL** | In PPC, data ownership determines who can defend TCPA claims. If seller owns all PII, buyer loses all leverage and cannot independently verify consent. Existential in PPC context. |
| `PUB_OPS_NON_CIRCUMVENT_BROAD` | HIGH (weight 25) | **CRITICAL** | Non-circumvention clauses in PPC/brokerage can permanently lock publishers out of buyer relationships they built. "Known or should reasonably be known" language captures the entire publisher's contact network. |
| `PUB_FIN_PAYMENT_TERMS_SLOW` | threshold 30d | **15d** (standard) | PPC industry standard is Net 7–14. Net 30 is already slow for PPC. Other verticals (HVAC=45d, attorney=60d) have different norms. |

---

## 4. Mandatory Checks

### File: `src/lib/ppc/mandatory.ts`

```typescript
/**
 * 17 rule IDs that MUST fire on every PPC contract analysis.
 *
 * Source: milo-ops contract-analysis-prompt.ts "MANDATORY CHECKS CHECKLIST"
 *
 * Post-processing enforcement: if any mandatory rule was NOT flagged by the AI,
 * the engine injects a "not detected — verify manually" advisory note.
 * This prevents silent misses on the most important checks.
 */
export const PPC_MANDATORY_CHECKS: string[] = [
  // Publisher mandatory
  'PUB_FIN_FINALITY_MISSING',              // 1. Missing finality/rejection window
  'PUB_FIN_POST_TERM_PAYMENT_MISSING',     // 3. Unclear post-termination payment
  'PUB_LIAB_INDEMNITY_ONE_SIDED',          // 4. One-sided indemnification
  'PUB_LIAB_INDEMNITY_NO_CAP',             // 5. Indemnity excluded from cap
  'PUB_LIAB_INDEMNITY_TRIGGER_LOW',        // 6. Low trigger indemnification
  'PUB_COMP_TCPA_STRICT_SHIFT',            // 7. TCPA liability shift
  'PUB_COMP_TCPA_RECORD_BURDEN',           // 8. Excessive record retention
  'PUB_OPS_NON_CIRCUMVENT_BROAD',          // 9. Overly broad non-circumvention
  'PUB_FIN_REJECTION_CATCH_ALL',           // 10. Subjective catch-all rejection
  'PUB_OPS_INSURANCE_ADDITIONAL_INSURED',  // 11. Insurance additional insured
  'PUB_OPS_VENUE_JURY_WAIVER',            // 12. Venue/jury waiver
  'PUB_FIN_INVOICE_FORFEITURE',            // 13. Automatic invoice forfeiture
  'PUB_OPS_CROSS_REF_ERROR',              // 14. Cross-reference errors

  // Buyer mandatory
  'BUY_OPS_NO_AUDIT_RIGHTS',              // 2. No audit rights
  'BUY_OPS_CALL_TRANSFER_NO_REJECT',      // 15. Call transfer rejection prohibited
  'BUY_OPS_TERMINATION_IMBALANCE',        // 16. Asymmetric termination rights
  'BUY_DATA_OWNERSHIP_UNCLEAR',           // 17. Seller owns ALL data/PII
];
```

### Mandatory Check Enforcement Logic

When the AI analysis returns, post-processing checks each mandatory rule:

```typescript
function enforceMandatoryChecks(
  issues: AnalysisIssue[],
  mandatoryChecks: string[],
  role: 'publisher' | 'buyer',
): AnalysisIssue[] {
  const flaggedRuleIds = new Set(issues.map(i => i.rule_id));

  const relevantChecks = mandatoryChecks.filter(ruleId => {
    if (role === 'publisher') return ruleId.startsWith('PUB_');
    if (role === 'buyer') return ruleId.startsWith('BUY_');
    return true;
  });

  const missingAdvisories = relevantChecks
    .filter(ruleId => !flaggedRuleIds.has(ruleId))
    .map(ruleId => ({
      rule_id: ruleId,
      severity: 'advisory' as const,
      title: `Mandatory check not triggered: ${ruleId}`,
      plain_english: 'This mandatory check was not flagged by the AI analysis. Verify manually that this area was adequately covered in the contract.',
      is_advisory: true,
    }));

  return [...issues, ...missingAdvisories];
}
```

---

## 5. PatternPlugin Definitions

### File: `src/lib/ppc/patterns.ts`

```typescript
import type { PatternPlugin } from '@milo/contract-analysis';

// ── Two-Party Consent States ─────────────────────────────────────────────────

export const TWO_PARTY_CONSENT_STATES = [
  'CA', 'CT', 'FL', 'IL', 'MD', 'MA', 'MI', 'MT', 'NV', 'NH', 'PA', 'WA',
] as const;

/**
 * Plugin 1: Two-Party Consent Escalation
 *
 * In two-party consent states, recording-related rules escalate to CRITICAL.
 * Source: MOP scoring.ts applyTwoPartyConsentEscalation() + jurisdiction.ts
 */
export const twoPartyConsentPlugin: PatternPlugin = {
  name: 'two-party-consent-escalation',
  apply(issues, metadata) {
    if (!metadata?.jurisdiction?.state) return issues;

    const state = metadata.jurisdiction.state.toUpperCase();
    const isTwoParty = TWO_PARTY_CONSENT_STATES.includes(state as any);

    if (!isTwoParty) return issues;

    // Only escalate if jurisdiction confidence is HIGH or MEDIUM
    if (metadata.jurisdiction.confidence === 'LOW' ||
        metadata.jurisdiction.confidence === 'UNKNOWN') {
      return issues.map(issue =>
        issue.isRecordingRelated
          ? {
              ...issue,
              advisoryNote: `Jurisdiction ${state} is a two-party consent state (confidence: ${metadata.jurisdiction.confidence}). Recording-related risks may be elevated. Verify jurisdiction before finalizing.`,
            }
          : issue
      );
    }

    return issues.map(issue => {
      if (!issue.isRecordingRelated) return issue;
      if (issue.severity === 'CRITICAL') return issue; // already max

      return {
        ...issue,
        severity: 'CRITICAL' as const,
        weight: Math.max(issue.weight ?? 20, 40),
        escalationReason: `Two-party consent state: ${state}. Recording without all-party consent is a criminal offense.`,
      };
    });
  },
};

/**
 * Plugin 2: Catch-All Rejection Injection
 *
 * Source: MOP issue-postprocessing.ts "Inject Missing Issues"
 *
 * Scans contract text for subjective rejection language patterns.
 * If PUB_FIN_REJECTION_CATCH_ALL was not flagged by the AI but the
 * contract text contains catch-all patterns, injects the issue.
 */
export const catchAllRejectionPlugin: PatternPlugin = {
  name: 'catch-all-rejection-injection',
  apply(issues, metadata) {
    const contractText = metadata?.contractText ?? '';
    if (!contractText) return issues;

    const alreadyFlagged = issues.some(
      i => i.rule_id === 'PUB_FIN_REJECTION_CATCH_ALL'
    );
    if (alreadyFlagged) return issues;

    const catchAllPatterns = [
      /any\s+other\s+reason/i,
      /sole\s+discretion/i,
      /calls?\s+into\s+question\s+the\s+legitimacy/i,
      /deemed\s+appropriate/i,
      /any\s+reason\s+whatsoever/i,
      /for\s+any\s+reason/i,
    ];

    const hasPattern = catchAllPatterns.some(p => p.test(contractText));
    if (!hasPattern) return issues;

    return [
      ...issues,
      {
        rule_id: 'PUB_FIN_REJECTION_CATCH_ALL',
        severity: 'HIGH',
        weight: 25,
        title: 'Subjective Rejection Criteria (Pattern-Detected)',
        plain_english: 'The contract contains open-ended catch-all rejection language that lets them reject calls for essentially any reason.',
        is_pattern_injected: true,
      },
    ];
  },
};

/**
 * Plugin 3: Indemnity Cap Guardrail
 *
 * Source: MOP issue-postprocessing.ts "Enforce Minimum Severity"
 *
 * Ensures PUB_LIAB_INDEMNITY_NO_CAP is always CRITICAL severity.
 * Prevents the AI from downgrading this existential risk.
 */
export const indemnityCaptGuardrailPlugin: PatternPlugin = {
  name: 'indemnity-cap-guardrail',
  apply(issues, _metadata) {
    return issues.map(issue => {
      if (issue.rule_id === 'PUB_LIAB_INDEMNITY_NO_CAP' &&
          issue.severity !== 'CRITICAL') {
        return {
          ...issue,
          severity: 'CRITICAL' as const,
          weight: 40,
          escalationReason: 'Indemnity excluded from liability cap is always CRITICAL — existential risk.',
        };
      }
      return issue;
    });
  },
};

/** All PPC PatternPlugins for registration */
export const PPC_PATTERN_PLUGINS: PatternPlugin[] = [
  twoPartyConsentPlugin,
  catchAllRejectionPlugin,
  indemnityCaptGuardrailPlugin,
];
```

### Additional Patterns from MOP (deferred to Phase 2)

MOP's `issue-postprocessing.ts` (1,113 LOC) contains several more post-processing steps. These are NOT included in the initial rule pack because they modify issue content rather than detection:

| MOP Post-Processing Step | Include Now? | Reason |
|-------------------------|-------------|--------|
| Canonicalization + role filtering | No | Engine-level concern, not vertical plugin |
| Deduplication (same clause + concept) | No | Engine-level concern |
| Liability cap reclassification | **Deferred** | Complex (detects capped vs uncapped indemnity). Add when engine v0.2 stabilizes. |
| Data retention fix | **Deferred** | Prevents recommending reduction of consent records. Add as revision guardrail. |
| Indemnity trigger revision fix | **Deferred** | Rewrites "finally adjudicated" suggestions. Add as revision guardrail. |
| Broker-safe revision templates | **Deferred** | Templates for clawback/indemnity/holdback. Add when revision templating is built. |
| Buyer-safe revision templates | **Deferred** | Templates for rejection/TCPA/consent. Same timing. |
| Indemnity cap inclusion guardrail | **Yes** (Plugin 3) | Simple severity enforcement. |
| Placeholder normalization | No | Engine-level concern |

---

## 6. DisplayConfig

### File: `src/lib/ppc/display-config.ts`

```typescript
import type { DisplayConfig } from '@milo/contract-analysis';

/**
 * PPC Display Configuration (locked, Directive #35)
 *
 * Milo-for-PPC shows per-issue severity ONLY.
 * No aggregate scores, letter grades, or risk bucket labels.
 * The engine still computes scores internally for sorting and
 * for other verticals that want aggregate displays.
 */
export const PPC_DISPLAY_CONFIG: DisplayConfig = {
  aggregateScore: false,
  letterGrade: false,
  riskBucket: false,
  issueOrdering: 'severity-desc',
};
```

---

## 7. File Structure

```
src/lib/ppc/
├── rules.ts            # 32 PPC RuleDefinition[] (~650 LOC)
├── overrides.ts        # severity + threshold overrides (~40 LOC)
├── mandatory.ts        # 17 mandatory check IDs (~30 LOC)
├── patterns.ts         # 3 PatternPlugin instances (~180 LOC)
├── display-config.ts   # no-score config (~15 LOC)
├── profiles.ts         # 4 scoring profiles (~120 LOC, see §8)
└── index.ts            # registerPpcRulePack() (~60 LOC)
```

**Total estimated LOC:** ~1,095 (rules dominate at ~650)

---

## 8. Scoring Profiles

### File: `src/lib/ppc/profiles.ts`

```typescript
import type { ScoringModifier } from '@milo/contract-analysis';

/**
 * PPC Scoring Profiles — from MOP profiles.ts
 *
 * Four named profiles with category/rule weight multipliers.
 * Operators select a profile before analysis to weight rules
 * based on context (testing a publisher vs scaling vs broker audit).
 */
export const PPC_PROFILES: Record<string, ScoringModifier> = {

  TEST: {
    name: 'test',
    description: 'Test/trial publisher — relaxed termination and dispute weights',
    categoryMultipliers: {
      TERM: 0.7,
      DISP: 0.8,
    },
    ruleMultipliers: {
      'PUB_TERM_NOTICE_TOO_SHORT': 0.5,
      'PUB_TERM_AUTO_RENEW_TRAP': 0.6,
      'PUB_TERM_MINIMUM_TERM_LONG': 0.5,
      'PUB_DISP_VENUE_INCONVENIENT': 0.6,
      'PUB_DISP_ARBITRATION_UNFAIR': 0.7,
      'PUB_DISP_CLASS_ACTION_WAIVER': 0.5,
      'PUB_OPS_EXCLUSIVITY': 0.6,
      'PUB_OPS_NON_COMPETE_BROAD': 0.7,
    },
    autoEscalate: [],
  },

  SCALE: {
    name: 'scale',
    description: 'Scaling publisher — elevated financial and compliance weights',
    categoryMultipliers: {
      FIN: 1.3,
      LIAB: 1.2,
      COMP: 1.2,
    },
    ruleMultipliers: {
      'PUB_FIN_FINALITY_MISSING': 1.5,
      'PUB_FIN_CLAWBACK_OPEN_ENDED': 1.5,
      'PUB_FIN_REPORTING_BINDING_NO_AUDIT': 1.3,
      'PUB_LIAB_INDEMNITY_NO_CAP': 1.5,
      'PUB_LIAB_INDEMNITY_TRIGGER_LOW': 1.3,
      'PUB_LIAB_PERSONAL_GUARANTY': 1.5,
      'PUB_COMP_TCPA_STRICT_SHIFT': 1.5,
      'PUB_COMP_TCPA_RECORD_PRODUCTION_EXTREME': 1.3,
      'PUB_OPS_LIQUIDATED_DAMAGES_100': 1.5,
    },
    autoEscalate: [
      'PUB_FIN_FINALITY_MISSING',
      'PUB_FIN_CLAWBACK_OPEN_ENDED',
      'PUB_LIAB_INDEMNITY_NO_CAP',
      'PUB_COMP_TCPA_STRICT_SHIFT',
      'PUB_LIAB_PERSONAL_GUARANTY',
    ],
  },

  BROKER: {
    name: 'broker',
    description: 'Broker audit — elevated payment flow and non-circumvention weights',
    categoryMultipliers: {
      FIN: 1.4,
    },
    ruleMultipliers: {
      'PUB_FIN_FINALITY_MISSING': 1.6,
      'PUB_FIN_CLAWBACK_OPEN_ENDED': 1.6,
      'PUB_FIN_REPORTING_BINDING_NO_AUDIT': 1.5,
      'PUB_FIN_HOLDBACK_NO_RELEASE': 1.5,
      'PUB_DISP_DISPUTE_DEADLINE_SHORT': 1.4,
      'PUB_OPS_NON_CIRCUMVENT_BROAD': 1.5,
    },
    autoEscalate: [
      'PUB_FIN_FINALITY_MISSING',
      'PUB_FIN_CLAWBACK_OPEN_ENDED',
    ],
  },

  BUYER: {
    name: 'buyer',
    description: 'Buyer perspective — elevated compliance and quality protections',
    categoryMultipliers: {
      COMP: 1.4,
      FIN: 1.3,
    },
    ruleMultipliers: {
      'BUY_COMP_TCPA_WARRANTY_MISSING': 1.4,
      'BUY_COMP_TCPA_INDEMNITY_WEAK': 1.4,
      'BUY_COMP_SUBAFFILIATE_COMPLIANCE_GAP': 1.5,
      'BUY_OPS_NO_CONSENT_PROOF_ACCESS': 1.6,
      'BUY_FIN_QUALITY_STANDARDS_VAGUE': 1.4,
      'BUY_FIN_NO_REJECTION_WINDOW': 1.5,
      'BUY_FIN_NO_REPLACEMENT_OR_CREDIT': 1.5,
      'BUY_LIAB_WARRANTY_DISCLAIMER_AS_IS': 1.6,
      'BUY_OPS_NO_AUDIT_RIGHTS': 1.4,
    },
    autoEscalate: [
      'BUY_COMP_TCPA_WARRANTY_MISSING',
      'BUY_OPS_NO_CONSENT_PROOF_ACCESS',
      'BUY_FIN_NO_REJECTION_WINDOW',
      'BUY_FIN_NO_REPLACEMENT_OR_CREDIT',
      'BUY_LIAB_WARRANTY_DISCLAIMER_AS_IS',
      'BUY_COMP_SUBAFFILIATE_COMPLIANCE_GAP',
    ],
  },
};
```

---

## 9. Registration Entry Point

### File: `src/lib/ppc/index.ts`

```typescript
import type { VerticalRulePack } from '@milo/contract-analysis';
import { PPC_RULES } from './rules.js';
import { PPC_SEVERITY_OVERRIDES, PPC_THRESHOLD_OVERRIDES } from './overrides.js';
import { PPC_MANDATORY_CHECKS } from './mandatory.js';
import { PPC_PATTERN_PLUGINS } from './patterns.js';
import { PPC_DISPLAY_CONFIG } from './display-config.js';
import { PPC_PROFILES } from './profiles.js';

/**
 * Construct the PPC VerticalRulePack for registration with
 * @milo/contract-analysis v0.2's plugin interface.
 */
export function registerPpcRulePack(): VerticalRulePack {
  return {
    name: 'ppc',
    tenantId: 'tlp',
    rules: PPC_RULES,
    severityOverrides: PPC_SEVERITY_OVERRIDES,
    thresholdOverrides: PPC_THRESHOLD_OVERRIDES,
    mandatoryChecks: PPC_MANDATORY_CHECKS,
    displayConfig: PPC_DISPLAY_CONFIG,
  };
}

export {
  PPC_RULES,
  PPC_SEVERITY_OVERRIDES,
  PPC_THRESHOLD_OVERRIDES,
  PPC_MANDATORY_CHECKS,
  PPC_PATTERN_PLUGINS,
  PPC_DISPLAY_CONFIG,
  PPC_PROFILES,
};
```

---

## 10. Sample Registration Code

How Milo-for-PPC consumes the rule pack at startup:

```typescript
// src/lib/analysis-engine.ts (Milo-for-PPC)

import { createAnalysisEngine, BASE_RULES } from '@milo/contract-analysis';
import {
  registerPpcRulePack,
  PPC_PATTERN_PLUGINS,
  PPC_PROFILES,
} from './ppc/index.js';
import { supabaseAdmin } from './supabase-admin.js';

const ppcPack = registerPpcRulePack();

export function createPpcAnalysisEngine(profile: keyof typeof PPC_PROFILES = 'SCALE') {
  return createAnalysisEngine({
    // Infrastructure
    supabase: supabaseAdmin(),
    tenantId: 'tlp',

    // Rules: 45 horizontal + 32 PPC = 77 total
    rules: [...BASE_RULES, ...ppcPack.rules],

    // Vertical plugin (overrides, mandatory checks, display config)
    verticalPack: ppcPack,

    // Pattern plugins (two-party consent, catch-all injection, indemnity guardrail)
    patternPlugins: PPC_PATTERN_PLUGINS,

    // Scoring profile
    scoringModifiers: profile ? [PPC_PROFILES[profile]] : [],

    // AI config
    systemPrompt: buildPpcSystemPrompt(),
    aiConfig: {
      model: 'claude-sonnet-4-6',
      maxTokens: 16000,
      temperature: 0.2,
    },

    // Store active rule snapshot in analysis metadata
    includeRuleSnapshot: true,
  });
}
```

### Usage in API Route

```typescript
// src/app/api/contract/analyze/route.ts

import { createPpcAnalysisEngine } from '@/lib/analysis-engine';

export async function POST(req: Request) {
  const { contractText, role, profile, jurisdiction } = await req.json();

  const engine = createPpcAnalysisEngine(profile);
  const result = await engine.analyze({
    text: contractText,
    role,
    metadata: { jurisdiction },
  });

  // result.issues are severity-ordered (CRITICAL first)
  // result.score/letterGrade/riskBucket are undefined (PPC display config)
  return Response.json(result);
}
```

---

## 11. Testing Strategy

### Unit Tests

| Test | Covers | LOC |
|------|--------|-----|
| Rule count validation | `PPC_RULES.length === 32` | ~10 |
| Rule ID uniqueness | No duplicate rule_ids across BASE + PPC | ~15 |
| Severity override targets exist | All keys in PPC_SEVERITY_OVERRIDES are in BASE_RULES | ~15 |
| Threshold override targets exist | All keys in PPC_THRESHOLD_OVERRIDES are in BASE_RULES | ~10 |
| Mandatory checks exist | All 17 IDs exist in either BASE_RULES or PPC_RULES | ~15 |
| Display config values | All three flags are false for PPC | ~5 |
| Registration shape | `registerPpcRulePack()` returns valid VerticalRulePack | ~10 |

### Pattern Plugin Tests

| Test | Input | Expected |
|------|-------|---------|
| Two-party consent: CA publisher | Issue with `isRecordingRelated: true`, jurisdiction `CA` | Severity escalated to CRITICAL |
| Two-party consent: TX publisher | Same issue, jurisdiction `TX` | No escalation |
| Two-party consent: low confidence | Jurisdiction `FL` with `confidence: 'LOW'` | Advisory note, no hard escalation |
| Catch-all injection: present | Contract text with "any other reason", no PUB_FIN_REJECTION_CATCH_ALL in issues | Issue injected |
| Catch-all injection: already flagged | Same text, issue already present | No duplicate |
| Indemnity guardrail: downgraded | PUB_LIAB_INDEMNITY_NO_CAP at HIGH | Forced to CRITICAL |

### Integration Tests (Against Real Contracts)

Run the engine against the 33 existing analysis records (migrated in Decision 53) and verify:
1. All 17 mandatory checks either fire or produce advisories
2. Severity overrides apply correctly (BUY_DATA_OWNERSHIP_UNCLEAR → CRITICAL)
3. Two-party consent escalation fires for FL/CA contracts
4. No regression in issue detection count vs. MOP's original analysis
5. Display config strips scores from output

---

## 12. Open Questions

| # | Question | Options | Urgency |
|---|----------|---------|---------|
| 1 | **System prompt source** | (a) Port milo-ops' `buildContractAnalysisPrompt()` as-is (496 LOC, production-proven) (b) Let @milo/contract-analysis v0.2 generate prompt from RuleDefinition[] (cleaner, but untested) | HIGH — determines whether prompt is declarative or hand-crafted |
| 2 | **Revision template system** | (a) Defer MOP's broker-safe/buyer-safe revision templates (b) Include as a 4th PatternPlugin that post-processes suggested_revision text | LOW — default AI revisions are acceptable |
| 3 | **Profile selection UI** | (a) Operator selects profile before upload (contract-checker preset pattern) (b) Auto-detect from contract metadata (publisher vs buyer auto-selects profile) | MEDIUM — affects settings page design |
| 4 | **Rule snapshot format** | (a) Store full rule pack in analysis_records.metadata (b) Store only overrides + delta from default | LOW — metadata JSONB is flexible |

### Question 1 Detail (Most Urgent)

The milo-ops prompt (`buildContractAnalysisPrompt()`) is 496 LOC of hand-crafted AI instructions with:
- Alex persona with communication style directives
- Role-specific check sequences (A-K publisher, A-I buyer)
- Mandatory check emphasis with repetition for frequently-missed rules
- Output format with 20+ fields per issue
- Plain English writing style enforcement
- Audit rights role direction safeguard

**Option A** (port as-is): Production-proven, covers edge cases discovered over months of use. Risk: hand-crafted prompt is tightly coupled to MOP's issue structure.

**Option B** (generate from rules): Cleaner architecture — the engine reads `RuleDefinition.prompt_instruction` fields and assembles the prompt. Risk: loses the hard-won nuances in the hand-crafted prompt (e.g., "NEVER flag favorable terms", "ALWAYS check for TWO separate issues in rejection clauses").

**Recommendation:** Option A for Phase 1, migrate to Option B in Phase 2 once we verify the generated prompt produces equivalent results.

---

## Summary

| Deliverable | Count/Value | Status |
|-------------|------------|--------|
| PPC vertical rules (RuleDefinition[]) | 32 | Complete (full code above) |
| Severity overrides | 2 rules escalated | Complete |
| Threshold overrides | 1 rule (payment terms 30d→15d) | Complete |
| Mandatory checks | 17 rule IDs | Complete |
| PatternPlugins | 3 (consent, catch-all, indemnity) | Complete |
| Scoring profiles | 4 (TEST, SCALE, BROKER, BUYER) | Complete |
| DisplayConfig | no-score PPC config | Complete |
| File structure | 7 files in `src/lib/ppc/` | Complete |
| Registration code | `registerPpcRulePack()` + usage sample | Complete |
| Estimated LOC | ~1,095 | — |
| Count correction | 45 horizontal + 32 PPC = 77 (summary was transposed) | Noted |
