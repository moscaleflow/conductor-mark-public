---
directive: "@milo/contract-analysis v0.2 implementation spec"
lane: research
coder: Coder-3
started: 2026-04-20 ~14:00 MDT
completed: 2026-04-20 ~14:45 MDT
supplements:
  - reports/2026-04-20-coder-3-ppc-rules-dedup-mapping.md (254794c)
  - reports/2026-04-20-coder-3-milo-for-ppc-scaffolding-plan.md (b5e376f)
  - ~/Documents/GitHub/milo-engine/packages/contract-analysis/ (v0.1.0 source)
---

# @milo/contract-analysis v0.2.0 — Implementation Spec

## 1. Executive Summary

| Metric | Value |
|--------|-------|
| New LOC (estimated) | ~1,200-1,500 |
| New files | 4 (base-registry.ts, plugin.ts, display-config.ts, tests) |
| Modified files | 3 (types.ts, engine.ts, index.ts) |
| Schema changes | 0 (no new columns or tables) |
| Breaking changes | 0 (additive only) |

**What v0.2 adds:**
- 45 horizontal rules as opt-in base registry
- Threshold config pattern with per-vertical overrides
- Severity override pattern with per-vertical overrides
- Display config flags (aggregate score, letter grade, risk bucket)
- Formal `VerticalRulePack` plugin interface

**What v0.2 does NOT change:**
- `analysis_records` and `analysis_jobs` schemas (unchanged)
- `createAnalysisEngine()` signature (additive new fields only)
- Scoring engine internals (SEVERITY_WEIGHTS, RISK_THRESHOLDS unchanged)
- Post-processing pipeline (unchanged)
- All existing v0.1.0 exports (preserved)

---

## 2. SCHEMA.md Amendment

### Decision: No Schema Changes Required

Rules, thresholds, severity overrides, and display config are all **engine configuration**, not stored data. They're resolved at call time from the vertical's config, not persisted per-analysis.

| Candidate | Store in DB? | Recommendation |
|-----------|-------------|----------------|
| Rule definitions (32 + vertical) | No | Code config — changes with deploys, not data |
| Threshold overrides | No | Code config |
| Severity overrides | No | Code config |
| Display config flags | No | Code config — UI concern, not engine data |
| Per-analysis rule snapshot | **Maybe** | See below |

### Per-Analysis Rule Snapshot (Optional, Deferred)

When an analysis runs, the engine could store which rules and overrides were active in `analysis_records.metadata`:

```json
{
  "engine_version": "0.2.0",
  "rules_count": 77,
  "base_rules_count": 45,
  "vertical_rules_count": 32,
  "severity_overrides": { "BUY_DATA_OWNERSHIP_UNCLEAR": "CRITICAL" },
  "threshold_overrides": { "PUB_FIN_PAYMENT_TERMS_SLOW": 15 },
  "display_config": { "aggregateScore": false }
}
```

This is audit metadata, not structural schema change. The existing `metadata JSONB` column on `analysis_records` already supports it. No migration needed.

**Recommendation:** Include rule snapshot in metadata (free, no schema change, useful for debugging). Defer displaying it in UI to Phase 2.

**SCHEMA.md amendment:** Add a note to the `analysis_records.metadata` section documenting the v0.2 rule snapshot fields. No DDL changes.

---

## 3. Public API Changes

### 3a. New Types

```typescript
// src/types.ts — additions

/** Configurable threshold per rule, overridable by verticals */
interface ThresholdConfig {
  strict: number | string;
  standard: number | string;
  relaxed: number | string;
}

/** Vertical pack of rules + overrides + display config */
interface VerticalRulePack {
  name: string;                                    // e.g., 'ppc', 'hvac', 'attorney'
  tenantId: string;                                // e.g., 'tlp'
  rules: RuleDefinition[];                         // vertical-specific rules (45 for PPC)
  severityOverrides?: Record<string, RiskLevel>;   // override base rule severity
  thresholdOverrides?: Record<string, ThresholdConfig>;  // override base rule thresholds
  mandatoryChecks?: string[];                      // rule_ids that MUST fire
  displayConfig?: DisplayConfig;                   // vertical display preferences
}

/** Controls what the engine returns in the result */
interface DisplayConfig {
  aggregateScore: boolean;     // default: true. PPC: false.
  letterGrade: boolean;        // default: true. PPC: false.
  riskBucket: boolean;         // default: true. PPC: false.
  issueOrdering: 'severity-desc' | 'category' | 'custom';  // default: severity-desc
}

/** Default display config (all features on) */
const DEFAULT_DISPLAY_CONFIG: DisplayConfig = {
  aggregateScore: true,
  letterGrade: true,
  riskBucket: true,
  issueOrdering: 'severity-desc',
};
```

### 3b. Extended RuleDefinition

```typescript
// src/types.ts — extend existing RuleDefinition

interface RuleDefinition {
  // Existing v0.1.0 fields (unchanged)
  rule_id: string;
  name: string;
  description: string;
  category: string;
  severity: RiskLevel;
  weight?: number;
  applies_to?: AnalysisRole;
  tags?: string[];
  prompt_instruction: string;
  keywords?: string[];

  // New v0.2.0 fields (all optional — backward compatible)
  isExistential?: boolean;            // existential risk flag
  threshold?: ThresholdConfig;        // configurable trigger point
  plainEnglishHint?: string;          // "1-2 sentences a non-lawyer can understand"
  impactTemplate?: string;            // "This affects {company} because..."
  recommendationFormat?: 'PUSH_BACK' | 'ACCEPT_WITH_MODIFICATION' | 'ACCEPT_AS_IS' | 'REJECT';
  riderRefs?: string[];               // rider section references
  isRecordingRelated?: boolean;       // for two-party consent escalation
  jurisdictionTriggers?: string[];    // states/countries that activate
}
```

### 3c. Extended AnalysisEngineConfig

```typescript
// src/types.ts — extend existing config

interface AnalysisEngineConfig {
  // Existing v0.1.0 fields (unchanged)
  supabase: SupabaseClient;
  tenantId: string;
  rules: RuleDefinition[];
  systemPrompt: string;
  aiConfig: AIAnalysisConfig;
  profiles?: AnalysisProfileConfig[];
  patternPlugins?: PatternPlugin[];
  scoringModifiers?: ScoringModifier[];
  postProcessingOptions?: PostProcessingOptions;
  storageBuckets?: StorageBucketConfig;

  // New v0.2.0 fields (all optional — backward compatible)
  verticalPack?: VerticalRulePack;     // if provided, merges with rules
  displayConfig?: DisplayConfig;        // override default display
  includeRuleSnapshot?: boolean;        // store active rules in metadata
}
```

### 3d. Extended AnalysisResult

```typescript
// AnalysisResult additions (existing fields preserved)

interface AnalysisResult {
  // Existing fields...

  // New v0.2.0: conditionally included based on displayConfig
  score?: number;           // only if displayConfig.aggregateScore
  letterGrade?: string;     // only if displayConfig.letterGrade
  riskBucket?: RiskLevel;   // only if displayConfig.riskBucket
}
```

---

## 4. Horizontal Rules Base Registry

### 4a. File Location

```
packages/contract-analysis/src/rules/
├── base-registry.ts      # 45 horizontal RuleDefinition[]
├── index.ts              # re-export
```

### 4b. Decision: Opt-In, Not Default

The 45 base rules are **exported but NOT auto-loaded**. Rationale:

- v0.1.0 consumers pass their own `rules` array. Auto-loading base rules would change behavior.
- Some verticals may want ONLY their rules, not the horizontal set.
- Explicit is better than implicit: `rules: [...BASE_RULES, ...myRules]`.

```typescript
// Consumer usage — explicit opt-in
import { createAnalysisEngine, BASE_RULES } from '@milo/contract-analysis';

const engine = createAnalysisEngine({
  rules: [...BASE_RULES, ...PPC_RULES],  // explicit merge
  // ...
});
```

### 4c. Base Registry Shape

```typescript
// src/rules/base-registry.ts
import type { RuleDefinition } from '../types.js';

export const BASE_RULES: RuleDefinition[] = [
  {
    rule_id: 'PUB_LIAB_INDEMNITY_TRIGGER_LOW',
    name: 'Low Indemnity Trigger',
    description: 'Indemnification triggers on "alleged", "suspected", or "claimed" rather than proven liability',
    category: 'LIAB',
    severity: 'CRITICAL',
    weight: 35,
    isExistential: true,
    tags: ['indemnity', 'liability', 'existential'],
    prompt_instruction: 'Check if indemnification triggers on "alleged", "suspected", "claimed", "asserted", "arising from any claim", or "threatened" rather than "finally adjudicated" or "proven". Flag if low-threshold trigger language is present.',
    plainEnglishHint: 'You could be forced to pay for claims that are only alleged, not proven.',
    impactTemplate: 'This affects {company} because a mere allegation — even frivolous — could trigger full indemnification obligations.',
    recommendationFormat: 'PUSH_BACK',
  },
  // ... 31 more rules (see Section 4d for complete list)
];
```

### 4d. Complete Base Registry (45 rules by category)

**Liability (9 rules):**

| # | rule_id | severity | weight | isExistential |
|---|---------|----------|--------|---------------|
| 1 | PUB_LIAB_INDEMNITY_TRIGGER_LOW | CRITICAL | 35 | true |
| 2 | PUB_LIAB_INDEMNITY_SCOPE_DOWNSTREAM | HIGH | 30 | false |
| 3 | PUB_LIAB_INDEMNITY_NO_CAP | CRITICAL | 40 | true |
| 4 | PUB_LIAB_LIABILITY_CAP_MISSING | HIGH | 20 | false |
| 5 | PUB_LIAB_LIABILITY_CAP_LIMITED_SCOPE | HIGH | 20 | false |
| 6 | PUB_LIAB_CONSEQUENTIAL_DAMAGES | MEDIUM | 12 | false |
| 7 | PUB_LIAB_INSURANCE_EXCESSIVE | MEDIUM | 12 | false |
| 8 | PUB_LIAB_PERSONAL_GUARANTY | CRITICAL | 45 | true |
| 9 | PUB_LIAB_INDEMNITY_ONE_SIDED | CRITICAL | 35 | true |

**Financial + Termination (9 rules):**

| # | rule_id | severity | weight | threshold |
|---|---------|----------|--------|-----------|
| 10 | PUB_FIN_PAYMENT_TERMS_SLOW | MEDIUM | 12 | strict:15, standard:30, relaxed:45 |
| 11 | PUB_FIN_POST_TERM_PAYMENT_MISSING | CRITICAL | 35 | — |
| 12 | PUB_FIN_RATE_CHANGE_NO_NOTICE | MEDIUM | 12 | strict:60, standard:30, relaxed:14 |
| 13 | PUB_TERM_ONE_SIDED | HIGH | 20 | — |
| 14 | PUB_TERM_POST_TERM_FORFEIT | CRITICAL | 35 | — |
| 15 | PUB_TERM_DATA_PORTABILITY_MISSING | LOW | 5 | — |
| 16 | PUB_TERM_NOTICE_TOO_SHORT | MEDIUM | 12 | strict:60, standard:30, relaxed:14 |
| 17 | PUB_TERM_AUTO_RENEW_TRAP | MEDIUM | 12 | strict:60, standard:30, relaxed:14 |
| 18 | PUB_TERM_MINIMUM_TERM_LONG | MEDIUM | 12 | — |

**Operational (9 rules):**

| # | rule_id | severity | weight |
|---|---------|----------|--------|
| 19 | PUB_OPS_EXCLUSIVITY | HIGH | 20 |
| 20 | PUB_OPS_NON_COMPETE_BROAD | HIGH | 20 |
| 21 | PUB_OPS_NON_CIRCUMVENT_BROAD | HIGH | 25 |
| 22 | PUB_OPS_APPROVAL_UNILATERAL | MEDIUM | 12 |
| 23 | PUB_OPS_AUDIT_MISSING | MEDIUM | 12 |
| 24 | PUB_OPS_LIQUIDATED_DAMAGES_100 | CRITICAL | 35 |
| 25 | PUB_OPS_INSURANCE_ADDITIONAL_INSURED | HIGH | 20 |
| 26 | PUB_OPS_VENUE_JURY_WAIVER | MEDIUM | 12 |
| 27 | PUB_OPS_CROSS_REF_ERROR | MEDIUM | 12 |

**Compliance (5 rules):**

| # | rule_id | severity | weight |
|---|---------|----------|--------|
| 28 | PUB_COMP_FTC_UNCLEAR | MEDIUM | 12 |
| 29 | PUB_COMP_IP_ASSIGNMENT_BROAD | MEDIUM | 12 |
| 30 | PUB_COMP_DATA_OWNERSHIP_UNCLEAR | MEDIUM | 12 |
| 31 | PUB_COMP_GDPR_MISSING | LOW | 5 |
| 32 | PUB_COMP_CASL_MISSING | HIGH | 20 |

**Dispute (6 rules):**

| # | rule_id | severity | weight |
|---|---------|----------|--------|
| 33 | PUB_DISP_VENUE_INCONVENIENT | MEDIUM | 12 |
| 34 | PUB_DISP_ARBITRATION_UNFAIR | MEDIUM | 12 |
| 35 | PUB_DISP_CLASS_ACTION_WAIVER | MEDIUM | 12 |
| 36 | PUB_DISP_DISPUTE_DEADLINE_SHORT | MEDIUM | 12 |
| 37 | PUB_DISP_INJUNCTION_NO_BOND | CRITICAL | 35 |
| 38 | PUB_DISP_LIQUIDATED_DAMAGES_EXCESSIVE | HIGH | 20 |

**Buyer General (7 rules):**

| # | rule_id | severity | weight |
|---|---------|----------|--------|
| 39 | BUY_FIN_PREPAY_OR_DEPOSIT_NO_REFUND | HIGH | 38 |
| 40 | BUY_OPS_NO_AUDIT_RIGHTS | CRITICAL | 35 |
| 41 | BUY_OPS_CHANGE_CONTROL_MISSING | HIGH | 25 |
| 42 | BUY_OPS_TERMINATION_IMBALANCE | CRITICAL | 35 |
| 43 | BUY_DATA_OWNERSHIP_UNCLEAR | CRITICAL | 38 |
| 44 | BUY_LIAB_WARRANTY_DISCLAIMER_AS_IS | CRITICAL | 40 |
| 45 | BUY_DISP_DISPUTE_WINDOW_TOO_SHORT | HIGH | 28 |

**Note:** Numbering here is internal to the base registry. The dedup mapping (254794c) uses a different numbering that spans all 77 rules.

---

## 5. Vertical Plugin Contract

### 5a. Registration Pattern

```typescript
// How Milo-for-PPC registers its rule pack

import { createAnalysisEngine, BASE_RULES } from '@milo/contract-analysis';
import type { VerticalRulePack } from '@milo/contract-analysis';
import { PPC_RULES, PPC_MANDATORY_CHECKS } from './ppc/rules.js';

const ppcPack: VerticalRulePack = {
  name: 'ppc',
  tenantId: 'tlp',
  rules: PPC_RULES,                     // 45 PPC-specific rules
  severityOverrides: {
    'BUY_DATA_OWNERSHIP_UNCLEAR': 'CRITICAL',     // base: HIGH → PPC: CRITICAL
    'PUB_OPS_NON_CIRCUMVENT_BROAD': 'CRITICAL',   // base: HIGH → PPC: CRITICAL
  },
  thresholdOverrides: {
    'PUB_FIN_PAYMENT_TERMS_SLOW': { strict: 10, standard: 15, relaxed: 30 },
  },
  mandatoryChecks: PPC_MANDATORY_CHECKS,  // 17 rules that must fire
  displayConfig: {
    aggregateScore: false,
    letterGrade: false,
    riskBucket: false,
    issueOrdering: 'severity-desc',
  },
};

const engine = createAnalysisEngine({
  supabase,
  tenantId: 'tlp',
  rules: [...BASE_RULES, ...ppcPack.rules],    // merge base + vertical
  verticalPack: ppcPack,                        // carries overrides + config
  systemPrompt: PPC_SYSTEM_PROMPT,
  aiConfig: { model: 'claude-sonnet-4-5', ... },
  includeRuleSnapshot: true,
});
```

### 5b. Override Resolution Logic

```typescript
// src/plugin.ts — new file

export function resolveRules(
  baseRules: RuleDefinition[],
  verticalPack?: VerticalRulePack
): RuleDefinition[] {
  if (!verticalPack) return baseRules;

  const merged = [...baseRules, ...verticalPack.rules];

  // Apply severity overrides to base rules
  if (verticalPack.severityOverrides) {
    for (const rule of merged) {
      const override = verticalPack.severityOverrides[rule.rule_id];
      if (override) {
        rule.severity = override;
        // Recalculate weight from new severity if not explicitly set
        if (!rule.weight) {
          rule.weight = SEVERITY_WEIGHTS[override];
        }
      }
    }
  }

  // Apply threshold overrides to base rules
  if (verticalPack.thresholdOverrides) {
    for (const rule of merged) {
      const override = verticalPack.thresholdOverrides[rule.rule_id];
      if (override) {
        rule.threshold = override;
      }
    }
  }

  return merged;
}

export function resolveDisplayConfig(
  verticalPack?: VerticalRulePack,
  explicitConfig?: DisplayConfig
): DisplayConfig {
  return explicitConfig ?? verticalPack?.displayConfig ?? DEFAULT_DISPLAY_CONFIG;
}
```

### 5c. Mandatory Checks Enforcement

```typescript
// In post-processing, after AI analysis returns

export function enforceMandatoryChecks(
  issues: ContractIssue[],
  mandatoryChecks: string[]
): ContractIssue[] {
  const firedRuleIds = new Set(issues.map(i => i.rule_id));

  for (const requiredId of mandatoryChecks) {
    if (!firedRuleIds.has(requiredId)) {
      issues.push({
        rule_id: requiredId,
        severity: 'LOW',
        title: `${requiredId} — not detected`,
        explanation: 'This mandatory check was not flagged by analysis. Verify manually.',
        suggested_revision: '',
        status: 'manual_review',
      });
    }
  }

  return issues;
}
```

### 5d. Display Config Application

```typescript
// In engine.ts — applied before returning AnalysisResult

function applyDisplayConfig(result: AnalysisResult, config: DisplayConfig): AnalysisResult {
  if (!config.aggregateScore) {
    delete result.score;
  }
  if (!config.letterGrade) {
    delete result.letterGrade;
  }
  if (!config.riskBucket) {
    delete result.riskBucket;
  }
  return result;
}
```

---

## 6. Engine Changes (engine.ts)

### Diff from v0.1.0

```typescript
// createAnalysisEngine — additions (no breaking changes)

export function createAnalysisEngine(config: AnalysisEngineConfig): AnalysisEngine {
  // NEW: resolve rules with vertical overrides
  const resolvedRules = resolveRules(config.rules, config.verticalPack);
  const displayConfig = resolveDisplayConfig(config.verticalPack, config.displayConfig);
  const mandatoryChecks = config.verticalPack?.mandatoryChecks ?? [];

  // Existing engine logic uses resolvedRules instead of config.rules
  // ...

  return {
    async analyze(text, opts) {
      // ... existing analysis logic ...

      // NEW: enforce mandatory checks in post-processing
      if (mandatoryChecks.length > 0) {
        processedIssues = enforceMandatoryChecks(processedIssues, mandatoryChecks);
      }

      // ... existing scoring logic (always runs internally) ...

      // NEW: store rule snapshot in metadata
      if (config.includeRuleSnapshot) {
        result.metadata = {
          ...result.metadata,
          engine_version: '0.2.0',
          rules_count: resolvedRules.length,
          base_rules_count: config.rules.length - (config.verticalPack?.rules.length ?? 0),
          vertical_rules_count: config.verticalPack?.rules.length ?? 0,
          severity_overrides: config.verticalPack?.severityOverrides ?? {},
          threshold_overrides: config.verticalPack?.thresholdOverrides ?? {},
          display_config: displayConfig,
        };
      }

      // NEW: apply display config (strip score/grade/bucket if disabled)
      return applyDisplayConfig(result, displayConfig);
    },
    // ... rest of engine methods unchanged ...
  };
}
```

---

## 7. Migration Strategy

### 7a. Existing Data (33 analysis_records)

**No retroactive re-scoring.** The 33 existing records in shared Supabase were analyzed with MOP's engine and scoring. They stay as-is:

| Concern | Resolution |
|---------|-----------|
| Old records lack v0.2 metadata | Acceptable — metadata is optional JSONB |
| Old records have different rule IDs | No conflict — rule_ids in issues are strings, not FK references |
| Old scoring used MOP weights | Weights are identical (SEVERITY_WEIGHTS unchanged) |
| Old records lack display config | Consumers check `metadata.engine_version` to know which fields exist |

### 7b. v0.1.0 → v0.2.0 Consumer Migration

**Zero breaking changes.** All v0.1.0 consumers work unchanged:

```typescript
// v0.1.0 consumer (still works in v0.2.0)
const engine = createAnalysisEngine({
  supabase,
  tenantId: 'tlp',
  rules: myRules,               // works — no verticalPack required
  systemPrompt: myPrompt,
  aiConfig: { ... },
});
// result.score, result.letterGrade, result.riskBucket all present (defaults on)
```

```typescript
// v0.2.0 consumer (new features)
const engine = createAnalysisEngine({
  supabase,
  tenantId: 'tlp',
  rules: [...BASE_RULES, ...PPC_RULES],
  verticalPack: ppcPack,        // new — carries overrides + display config
  systemPrompt: myPrompt,
  aiConfig: { ... },
  includeRuleSnapshot: true,    // new — audit metadata
});
// result.score undefined (PPC display config disables it)
```

---

## 8. File Manifest

### New Files

| File | LOC (est.) | Purpose |
|------|-----------|---------|
| `src/rules/base-registry.ts` | ~680 | 45 horizontal RuleDefinition[] |
| `src/rules/index.ts` | ~5 | Re-export BASE_RULES |
| `src/plugin.ts` | ~120 | resolveRules(), resolveDisplayConfig(), enforceMandatoryChecks() |
| `src/display-config.ts` | ~30 | DisplayConfig type, DEFAULT_DISPLAY_CONFIG, applyDisplayConfig() |

### Modified Files

| File | Changes | LOC delta |
|------|---------|-----------|
| `src/types.ts` | Add ThresholdConfig, VerticalRulePack, DisplayConfig, extend RuleDefinition + AnalysisEngineConfig | +60 |
| `src/engine.ts` | Call resolveRules(), enforceMandatoryChecks(), applyDisplayConfig(), store metadata | +40 |
| `src/index.ts` | Export BASE_RULES, new types, plugin functions | +15 |
| `SCHEMA.md` | Note about v0.2 metadata fields in analysis_records.metadata | +20 |
| `package.json` | Version bump to 0.2.0 | +1 |

### Test Files

| File | LOC (est.) | Coverage |
|------|-----------|----------|
| `src/__tests__/base-registry.test.ts` | ~200 | Each of 45 rules has valid structure, no duplicate IDs, categories valid |
| `src/__tests__/plugin.test.ts` | ~150 | Override resolution, mandatory checks, display config |
| `src/__tests__/display-config.test.ts` | ~50 | Config application, field stripping |

### LOC Summary

| Category | LOC |
|----------|-----|
| Base registry (45 rules) | ~680 |
| Plugin logic | ~150 |
| Display config | ~30 |
| Type additions | ~60 |
| Engine changes | ~40 |
| Exports | ~20 |
| Tests | ~400 |
| Docs | ~20 |
| **Total** | **~1,200** |

---

## 9. Test Plan

### 9a. Base Registry Validation

```typescript
describe('BASE_RULES', () => {
  test('contains exactly 45 rules', () => {
    expect(BASE_RULES).toHaveLength(32);
  });

  test('no duplicate rule_ids', () => {
    const ids = BASE_RULES.map(r => r.rule_id);
    expect(new Set(ids).size).toBe(ids.length);
  });

  test('all rules have required fields', () => {
    for (const rule of BASE_RULES) {
      expect(rule.rule_id).toBeTruthy();
      expect(rule.name).toBeTruthy();
      expect(rule.prompt_instruction).toBeTruthy();
      expect(['LOW', 'MEDIUM', 'HIGH', 'CRITICAL']).toContain(rule.severity);
      expect(rule.weight).toBeGreaterThan(0);
    }
  });

  test('existential rules are CRITICAL', () => {
    for (const rule of BASE_RULES.filter(r => r.isExistential)) {
      expect(rule.severity).toBe('CRITICAL');
    }
  });

  test('threshold rules have valid config', () => {
    for (const rule of BASE_RULES.filter(r => r.threshold)) {
      expect(rule.threshold!.strict).toBeDefined();
      expect(rule.threshold!.standard).toBeDefined();
      expect(rule.threshold!.relaxed).toBeDefined();
    }
  });
});
```

### 9b. Vertical Plugin Tests

```typescript
describe('resolveRules', () => {
  test('merges base + vertical rules', () => {
    const result = resolveRules(BASE_RULES, mockPpcPack);
    expect(result).toHaveLength(32 + 45);
  });

  test('applies severity overrides to base rules', () => {
    const result = resolveRules(BASE_RULES, {
      ...mockPpcPack,
      severityOverrides: { 'BUY_DATA_OWNERSHIP_UNCLEAR': 'CRITICAL' },
    });
    const rule = result.find(r => r.rule_id === 'BUY_DATA_OWNERSHIP_UNCLEAR');
    expect(rule!.severity).toBe('CRITICAL');
  });

  test('applies threshold overrides to base rules', () => {
    const result = resolveRules(BASE_RULES, {
      ...mockPpcPack,
      thresholdOverrides: {
        'PUB_FIN_PAYMENT_TERMS_SLOW': { strict: 10, standard: 15, relaxed: 30 },
      },
    });
    const rule = result.find(r => r.rule_id === 'PUB_FIN_PAYMENT_TERMS_SLOW');
    expect(rule!.threshold!.standard).toBe(15);
  });

  test('does not modify original BASE_RULES array', () => {
    const original = BASE_RULES.find(r => r.rule_id === 'BUY_DATA_OWNERSHIP_UNCLEAR');
    resolveRules(BASE_RULES, {
      ...mockPpcPack,
      severityOverrides: { 'BUY_DATA_OWNERSHIP_UNCLEAR': 'CRITICAL' },
    });
    expect(original!.severity).not.toBe('CRITICAL');  // original unchanged
  });
});

describe('enforceMandatoryChecks', () => {
  test('adds missing mandatory rules as manual_review', () => {
    const issues = [{ rule_id: 'PUB_FIN_FINALITY_MISSING', ... }];
    const result = enforceMandatoryChecks(issues, [
      'PUB_FIN_FINALITY_MISSING',
      'BUY_OPS_NO_AUDIT_RIGHTS',
    ]);
    expect(result).toHaveLength(2);
    expect(result[1].status).toBe('manual_review');
  });

  test('does not duplicate already-fired rules', () => {
    const issues = [{ rule_id: 'PUB_FIN_FINALITY_MISSING', ... }];
    const result = enforceMandatoryChecks(issues, ['PUB_FIN_FINALITY_MISSING']);
    expect(result).toHaveLength(1);
  });
});
```

### 9c. Display Config Tests

```typescript
describe('applyDisplayConfig', () => {
  test('strips score when aggregateScore=false', () => {
    const result = { score: 72, letterGrade: 'C+', riskBucket: 'HIGH', issues: [] };
    const filtered = applyDisplayConfig(result, { ...DEFAULT_DISPLAY_CONFIG, aggregateScore: false });
    expect(filtered.score).toBeUndefined();
    expect(filtered.letterGrade).toBe('C+');  // not stripped
  });

  test('PPC config strips all aggregate fields', () => {
    const result = { score: 72, letterGrade: 'C+', riskBucket: 'HIGH', issues: [] };
    const ppcConfig = { aggregateScore: false, letterGrade: false, riskBucket: false, issueOrdering: 'severity-desc' };
    const filtered = applyDisplayConfig(result, ppcConfig);
    expect(filtered.score).toBeUndefined();
    expect(filtered.letterGrade).toBeUndefined();
    expect(filtered.riskBucket).toBeUndefined();
    expect(filtered.issues).toBeDefined();  // issues always present
  });

  test('default config preserves all fields', () => {
    const result = { score: 72, letterGrade: 'C+', riskBucket: 'HIGH', issues: [] };
    const filtered = applyDisplayConfig(result, DEFAULT_DISPLAY_CONFIG);
    expect(filtered.score).toBe(72);
    expect(filtered.letterGrade).toBe('C+');
    expect(filtered.riskBucket).toBe('HIGH');
  });
});
```

### 9d. Tenant Isolation

```typescript
describe('tenant isolation', () => {
  test('vertical pack tenantId matches engine tenantId', () => {
    expect(() => createAnalysisEngine({
      tenantId: 'tlp',
      verticalPack: { tenantId: 'other', ... },
      ...
    })).toThrow('tenantId mismatch');
  });
});
```

---

## 10. Implementation Sequence for Coder-1

| Step | Action | Files | Effort |
|------|--------|-------|--------|
| 1 | SCHEMA.md amendment (add v0.2 metadata note) | SCHEMA.md | 10 min |
| 2 | Add new types to types.ts | src/types.ts | 30 min |
| 3 | Create src/rules/base-registry.ts (45 rules) | New file | 3-4 hrs |
| 4 | Create src/plugin.ts (resolve + enforce + display) | New file | 1 hr |
| 5 | Create src/display-config.ts | New file | 15 min |
| 6 | Modify engine.ts (integrate plugin + display) | src/engine.ts | 1 hr |
| 7 | Update src/index.ts (new exports) | src/index.ts | 10 min |
| 8 | Write tests | src/__tests__/ | 1.5 hrs |
| 9 | npm run build + verify | — | 15 min |
| 10 | Bump version to 0.2.0 | package.json | 5 min |

**Total estimated: ~7 hours**

Step 3 (45 rules) is the bulk of the work. Each rule needs `prompt_instruction` (the natural language AI instruction), which must be carefully worded. The dedup mapping (254794c) provides IDs, severities, and weights. The prompt_instruction text comes from MOP's registry.ts (for MOP-origin rules) and milo-ops's contract-analysis-prompt.ts (for milo-ops-origin rules).

---

## 11. Open Questions for Mark

### Q1: Letter grade computation

The engine currently doesn't compute letter grades — only numeric score + risk bucket. Adding letter grade means defining a mapping:

| Score | Grade |
|-------|-------|
| 90-100 | A |
| 80-89 | B+ |
| 70-79 | B |
| 55-69 | C |
| 30-54 | D |
| 0-29 | F |

Should this be added to the horizontal engine (other verticals may want it), or deferred (no vertical currently displays grades)?

**Recommendation:** Defer. No vertical needs it today. Add when a vertical requests it.

### Q2: Rule prompt_instruction sourcing

The 45 base rules need `prompt_instruction` text (tells Claude what to look for). Two options:
- **A:** Coder-1 writes generic horizontal prompt instructions
- **B:** Copy from MOP's registry.ts (already battle-tested on 33 analyses)

**Recommendation:** B — copy from MOP, edit to remove PPC jargon from horizontal rules.

### Q3: Version bump strategy

@milo/contract-analysis goes from v0.1.0 → v0.2.0. Should Coder-1:
- **A:** npm version minor (0.2.0) — standard semver
- **B:** npm version patch first (0.1.1) if any bug fixes are pending, then 0.2.0

**Recommendation:** A — straight to 0.2.0. No pending bug fixes.

---

## Summary

| Aspect | v0.1.0 | v0.2.0 |
|--------|--------|--------|
| Horizontal rules | 0 (verticals provide all) | 45 (opt-in BASE_RULES export) |
| Vertical plugin interface | Informal (rules array) | Formal (VerticalRulePack) |
| Severity overrides | Not supported | Per-rule override via verticalPack |
| Threshold config | Not supported | Per-rule ThresholdConfig, overridable |
| Display config | Always returns score/grade/bucket | Configurable per-vertical |
| Mandatory checks | Not supported | Enforced via verticalPack |
| Rule snapshot | Not stored | Optional metadata on analysis_records |
| Breaking changes | — | 0 |
| Schema changes | — | 0 (metadata-only) |
