# D172 — Agent Teams Full Cutover

**Coder:** 4 (Ops)
**Date:** 2026-04-22
**Status:** COMPLETE
**Directive:** Full Agent Teams cutover per D166 migration plan

## Summary

Converted conductor-mark's multi-agent coordination from 4-terminal manual relay to Claude Code Agent Teams as the primary workflow, with flat-file discipline preserved as the coordination backbone.

## Commits

| SHA | Repo | Description |
|-----|------|-------------|
| dec6d30 | conductor-mark | 4 subagent definitions (.claude/agents/coder-{1-4}-*.md) + idle-commit hook |
| 9353ecf | conductor-mark | Rename scripts/tmux-coders.sh → scripts/fallback-tmux-manual.sh |
| 39405f4 | conductor-mark | Coordination docs, spawn template, drift guards update |

## Deliverables

### 1. Subagent definitions (`.claude/agents/`)
- `coder-1-extraction.md` — Extraction lane. Cross-repo rules from milo-for-ppc baked in (timezone pattern, billing type awareness, drift hooks).
- `coder-2-migration.md` — Migration lane. One-consumer-at-a-time rule, build-before-commit.
- `coder-3-research.md` — Research lane. Read-only, spec file exemption from mirror URL.
- `coder-4-ops.md` — Ops lane. Decision numbering, directive hygiene, Supabase credential retrieval.

All four include: commit discipline, mirror URL convention, `--no-verify` ban, lane boundary enforcement.

### 2. Team lead spawn template
`AGENT-SCOPES/TEAM-LEAD-SPAWN.md` — Prerequisites, spawn prompt with permanent (research + ops) and on-demand (extraction + migration) teammates, mid-session spawn instructions, fallback reference.

### 3. Doc updates
- `AGENT-SCOPES/README.md` — Phase 4 status, Agent Teams integration section with subagent defs, hook, spawn template, fallback, feature flag, terminal requirement.
- `CLAUDE.md` — Multi-agent coordination section rewritten for Agent Teams primary workflow, cross-repo protocol section added (5 repos × lane assignments).
- `STRATEGY-CLAUDE-DRIFT-GUARDS.md` — Context updated from "chat-paste relay" to "Agent Teams" with fallback reference. Added Pattern 19 (lead must not implement, only coordinate) and Pattern 20 (teammate must commit before going idle).

### 4. Idle-commit hook
`.claude/hooks/teammate-idle-commit-check.sh` — Checks `git status --porcelain`, exits 2 if uncommitted changes exist. Prevents D166 Risk 1 (teammate idle with uncommitted work).

### 5. Fallback script rename
`scripts/tmux-coders.sh` → `scripts/fallback-tmux-manual.sh` — Header updated to mark as fallback, primary workflow is Agent Teams.

## Verification

- All 3 commits clean, no hook violations
- Subagent definitions reference correct repo paths and lane scopes
- Spawn template tested in D168 pilot (SyncFooter verification passed)
- Idle-commit hook is executable (`chmod +x` applied in dec6d30)

## Notes

- First write attempt was rejected (4 parallel writes). Resolved by writing sequentially.
- Feature flag required: `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`
- Team lead must run from standalone terminal, not VS Code (split-pane not supported in VS Code terminal).

Report at: https://raw.githubusercontent.com/moscaleflow/conductor-mark-public/main/reports/2026-04-22-d172-agent-teams-full-cutover.md
