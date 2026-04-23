# D166 — Agent Teams Migration Spec

**Coder:** 4 (Ops)  
**Directive:** D166  
**Date:** 2026-04-22  
**Status:** COMPLETE (spec only — no implementation)

---

## Executive Summary

Claude Code Agent Teams (v2.1.32+, experimental) can replace our 4-terminal manual relay with a single team-lead session that spawns and coordinates 4 teammates automatically. Mark's setup (v2.1.117, tmux 3.6a, macOS) meets all prerequisites.

**The single biggest blocker is cross-repo coordination.** Teammates are full Claude Code sessions that can `cd` and run Bash freely, but they load CLAUDE.md only from the team lead's working directory at spawn time. Our work spans 5 repos. The honest answer: Agent Teams can do this today with workarounds (spawn prompts that include repo paths + lane context), but the ergonomics are rougher than our current setup for cross-repo implementation work.

**Recommendation:** Option B — pilot one scoped task (V4 shell refinements: footer + picker + jiggle) through Agent Teams, validate cross-repo behavior, then decide on cutover vs. hybrid.

---

## 1. Setup Audit

| Check | Result | Status |
|---|---|---|
| Claude Code version | `2.1.117` (≥ 2.1.32) | PASS |
| tmux | `3.6a` via Homebrew (`/opt/homebrew/bin/tmux`) | PASS |
| iTerm2 | Not installed (`/Applications/iTerm.app` absent) | N/A |
| `it2` CLI | Not installed | N/A |
| `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS` | Not set | NEEDS CONFIG |
| Terminal | VS Code integrated terminal (`CLAUDE_CODE_ENTRYPOINT=claude-vscode`) | LIMITATION |
| `~/.claude/settings.json` | No conflicting env vars. Model: `claude-opus-4-6`. `defaultMode: bypassPermissions` in local settings. 8 PreToolUse auto-accept hooks (all empty matchers). | CLEAN |
| `~/.claude.json` | No `teammateMode` set. `promptQueueUseCount: 181`. No team-related config. | CLEAN |
| Existing task dirs | 2 dirs in `~/.claude/tasks/`. One has 110 task JSONs (prior experiment — onboarding walkthrough tasks). | CLEAN UP BEFORE PILOT |

### Critical finding: VS Code terminal limitation

From the docs:

> Split-pane mode isn't supported in VS Code's integrated terminal, Windows Terminal, or Ghostty.

Mark currently runs Claude Code from VS Code's integrated terminal. **Split-pane (tmux) mode will not work from within VS Code.** Options:

1. **Run team lead from a standalone terminal** (Terminal.app, Warp, or Alacritty) with tmux — gets split-pane visibility
2. **Use in-process mode from VS Code** — works but all teammates are invisible unless you Shift+Down cycle through them
3. **Install iTerm2** — enables `tmux -CC` integration (recommended by docs for macOS) plus native split panes

**Recommendation:** Run the team lead from a standalone terminal with tmux. Keep VS Code open for code editing. This matches how Mark already uses 4 terminals today.

### Setup commands (when ready)

```bash
# Enable Agent Teams
# Add to ~/.claude/settings.json under "env":
#   "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"

# Or set in shell:
export CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1

# Clean up stale task dirs from prior experiments:
rm -rf ~/.claude/tasks/ce87a579-577b-4e1b-b501-db7deecd5114
rm -rf ~/.claude/tasks/fd72ed8d-e28a-4b96-8978-2061d7991f57

# Start team lead with tmux backend from standalone terminal:
cd ~/mark\ conduct/conductor-mark
claude --teammate-mode tmux
```

---

## 2. Design Questions

### Q1: Team Lead Session Location

**Recommendation: `~/mark conduct/conductor-mark`** (coordination repo)

**Rationale:**
- Team lead's job is coordination, not implementation. It reads TASKS.md, DIRECTIVE-LEDGER.md, AGENT-SCOPES/*, and dispatches work.
- conductor-mark's CLAUDE.md contains the global rules all teammates need: timezone patterns, billing type awareness, directive protocol, visual verification protocol.
- Teammates `cd` to implementation repos (milo-for-ppc, milo-engine, etc.) as needed. The lead stays put.
- This matches our current model: Strategy-Claude (the lead) operates from conductor-mark context; Coders (teammates) operate in their target repos.

**Alternative considered:** Starting from `~/Documents/GitHub/milo-for-ppc`. Rejected because the lead would need to `cd` to conductor-mark for every coordination task, and milo-for-ppc's CLAUDE.md (if it has one) wouldn't contain multi-agent coordination rules.

### Q2: Teammate Spawn Strategy

**Recommendation: Option C (Hybrid) — 2 permanent + 2 on-demand**

| Teammate | Permanence | Rationale |
|---|---|---|
| Research (Coder-3) | Permanent | Always has audit/investigation work. Low file-conflict risk. Reads code, writes reports. |
| Ops (Coder-4) | Permanent | Continuous coordination: TASKS.md updates, ledger maintenance, health checks, mirror verification. |
| Extraction (Coder-1) | On-demand | Only active during implementation bursts. Touches many files. Spawn per directive, destroy after report. |
| Migration (Coder-2) | On-demand | Only active when Extraction ships something to consume. Often idle for hours. |

**Tradeoffs:**

| Strategy | Pros | Cons |
|---|---|---|
| A: All 4 permanent | Simple mental model, always ready | 2 idle teammates burning context tokens, higher base cost |
| B: All on-demand | Minimal token waste | Spawn latency on every task, lose teammate context between tasks |
| **C: Hybrid** | Research+Ops always warm, Extraction+Migration only pay when working | Slightly more complex spawn logic for lead |

**Token impact:** Permanent teammates consume tokens even when idle (context window stays open). With Option C, idle cost is ~2x base instead of ~4x. Active cost is identical across all options.

### Q3: Cross-Repo Coordination

**This is the critical architectural question.**

#### What the docs say

Teammates are "full, independent Claude Code session[s]" that can run Bash, cd, read files, etc. The docs do not explicitly restrict teammates to one directory. However:

> Each teammate has its own context window. When spawned, a teammate loads the same project context as a regular session: CLAUDE.md, MCP servers, and skills. It also receives the spawn prompt from the lead.

This means:
1. **CLAUDE.md is loaded once at spawn from the lead's working directory** — not dynamically when a teammate `cd`s elsewhere.
2. **Teammates CAN `cd` into sibling directories** and run git, npm, etc. This is confirmed by the Bash tool being available.
3. **But they won't automatically pick up another repo's CLAUDE.md** after changing directories.

#### Our repo layout

```
~/mark conduct/conductor-mark/     ← coordination (team lead lives here)
~/Documents/GitHub/milo-for-ppc/   ← primary product repo (Coder-1, Coder-2)
~/Documents/GitHub/milo-engine/    ← extracted packages (Coder-1)
~/Documents/GitHub/milo-outreach/  ← outreach service (Coder-2)
~/mysecretary/                     ← secondary product (Coder-2)
```

#### Assessment

**Cross-repo works but is imperfect.** Teammates can `cd ~/Documents/GitHub/milo-for-ppc && git pull && npm run build`. What they miss:

- Target repo's CLAUDE.md rules (if different from conductor-mark's)
- Target repo's `.claude/` project settings
- Pre-commit hooks configured via the target repo's `.husky/`

**Mitigations:**

1. **Fat spawn prompts.** When spawning Coder-1 for a milo-for-ppc task, include the critical rules from that repo's CLAUDE.md in the spawn prompt. Our current coder-N.md scope files already contain this — paste the scope file content into the spawn prompt.

2. **additionalDirectories already configured.** Mark's settings.json includes:
   ```json
   "additionalDirectories": [
     "/Users/markymark/Documents/GitHub/milo-engine/.husky",
     "/Users/markymark/Documents/GitHub/milo-outreach/.husky"
   ]
   ```
   Teammates inherit the lead's permission settings, so they'll have read access to these paths.

3. **Subagent definitions.** The docs support using subagent type definitions for teammates. We can create `.claude/agents/coder-1-extraction.md` etc. with the lane-specific instructions baked in.

#### Alternatives considered

| Approach | Verdict |
|---|---|
| Multiple team-lead sessions (one per repo) | Rejected — defeats the point. Lead can't coordinate across teams. One team per session limit. |
| Monorepo restructure | Rejected — massive effort, Mark has decided against this (Decision Log). |
| Git worktrees | Partial fit — useful for Extraction+Migration working on same repo simultaneously, but doesn't solve the multi-repo problem. |

**Bottom line:** Cross-repo works with workarounds. The workarounds are: (a) fat spawn prompts with repo-specific rules, (b) subagent definitions per lane, (c) teammates explicitly `cd` before starting work. Not as clean as single-repo teams, but functional.

### Q4: Task List Format

**Current format:**

```
TASKS.md:
- Coder-1: Extracting @milo/ai-client (started 2026-04-19 14:32)
- Coder-4: BLOCKED on D162 Item 1 — waiting on Mark

DIRECTIVE-LEDGER.md:
| D162 | 4 | Resend domain + ledger gap close | PARTIAL | 79a19e7 | [d162](...) |
```

**Agent Teams format** (from `~/.claude/tasks/{team-name}/`):

```json
{
  "id": "159",
  "subject": "US publisher onboarding walkthrough",
  "description": "Walk all 20 steps of the US publisher onboarding flow...",
  "activeForm": "Walking US publisher onboarding flow",
  "status": "pending",
  "blocks": [],
  "blockedBy": []
}
```

**What translates directly:**
- Subject → directive topic
- Description → directive body
- Status → pending/in-progress/completed maps to our pending/claimed/complete
- blockedBy → our dependency tracking

**What needs to change:**
- DIRECTIVE-LEDGER.md stays as the permanent record (task JSONs are ephemeral — cleaned up with team)
- TASKS.md becomes a snapshot view that the lead or Ops teammate updates, not the primary coordination mechanism
- Directives from Strategy-Claude (chat) → Mark pastes into the lead session → lead creates tasks for teammates

**Recommended workflow:**

```
Strategy-Claude writes directive in chat
  → Mark pastes directive text to team lead
    → Lead creates tasks in shared task list
      → Lead assigns or teammates self-claim
        → Teammates execute, mark complete
          → Lead synthesizes, updates TASKS.md + DIRECTIVE-LEDGER.md
            → Report committed with mirror URL
```

**Key change:** Strategy-Claude does NOT write directly to the task list. The team lead translates directives into tasks. This preserves the human gate (Mark reviews before work starts) and avoids format mismatches.

**DIRECTIVE-LEDGER.md convention:** Add column `Team Task ID` for traceability. When a directive is executed via Agent Teams, the ledger row gets the task ID from `~/.claude/tasks/`.

---

## 3. Token Cost Estimate

### Current 4-terminal model

Each terminal is a separate Claude Code session. Typical directive load: 4 directives in flight simultaneously (e.g., D159, D151, D164, D162).

| Component | Tokens/hour (est.) |
|---|---|
| 4 active sessions × context window maintenance | ~800K input tokens/hour |
| Tool calls (git, grep, read, write) | ~200K output tokens/hour |
| Strategy-Claude relay (Mark copy-paste overhead) | ~50K tokens/hour |
| **Total** | **~1.05M tokens/hour** |

### Agent Teams model (Option C hybrid)

From docs: "token costs scale linearly" per teammate.

| Component | Tokens/hour (est.) |
|---|---|
| Team lead (coordination only) | ~150K input tokens/hour |
| 2 permanent teammates (Research + Ops) | ~400K input tokens/hour |
| 2 on-demand teammates (Extraction + Migration, when active) | ~400K input tokens/hour |
| Inter-teammate messaging | ~50K tokens/hour |
| **Total (all 4 active)** | **~1.0M tokens/hour** |
| **Total (2 permanent only)** | **~550K tokens/hour** |

### Comparison

| Metric | 4-Terminal | Agent Teams (all active) | Agent Teams (2 perm) |
|---|---|---|---|
| Tokens/hour | ~1.05M | ~1.0M | ~550K |
| Mark relay overhead | High (copy-paste every message) | Zero (lead coordinates) | Zero |
| Idle token burn | Zero (Mark closes terminals) | ~150K/hr per idle teammate | ~150K/hr per idle teammate |

**Order of magnitude:** Roughly equivalent when all 4 are active. Agent Teams saves the human relay cost (Mark's time, not tokens). When only 2 teammates are active, token cost drops ~45%.

**The real savings are in Mark's time, not token spend.** Current model requires Mark to relay every message between Strategy-Claude and each Coder. Agent Teams eliminates this entirely.

---

## 4. Risks + Mitigations

### Risk 1: No session resumption

> `/resume` and `/rewind` do not restore in-process teammates. After resuming a session, the lead may attempt to message teammates that no longer exist.

**Impact:** If Mark's laptop sleeps, network drops, or the lead session crashes, all teammate work in progress is lost. Teammates die with the lead.

**Mitigation:**
1. **Commit discipline (existing rule).** Our protocol already requires "commit small and often." Teammates commit after every logical unit. Lost work = at most the current uncommitted change.
2. **TASKS.md as backup state.** Even if the team dies, TASKS.md + DIRECTIVE-LEDGER.md persist on disk. A new team can be spawned and pick up from the last committed state.
3. **Ops teammate writes checkpoints.** The permanent Ops teammate updates TASKS.md every 15 minutes with current state. If the team dies, Mark reads TASKS.md to see exactly where each lane was.
4. **TeammateIdle hook.** Add a hook that forces teammates to commit before going idle:
   ```bash
   # hooks/teammate-idle-commit-check.sh
   # Exit code 2 = reject idle, send feedback
   if [ -n "$(git status --porcelain)" ]; then
     echo "You have uncommitted changes. Commit before going idle."
     exit 2
   fi
   ```

### Risk 2: Task status lag

> Teammates sometimes fail to mark tasks as completed, which blocks dependent tasks.

**Impact:** Extraction teammate finishes but doesn't mark complete → Migration teammate blocked indefinitely → Mark has to manually intervene.

**Mitigation:**
1. **TaskCompleted hook.** Enforce that task completion requires a commit SHA or report path:
   ```bash
   # hooks/task-completed-check.sh
   # Verify the task has deliverable evidence
   ```
2. **Lead check-in cadence.** Tell the lead: "Every 10 minutes, check all in-progress tasks. If any has been in-progress for >20 minutes with no messages, nudge the teammate."
3. **Manual override.** Mark can Shift+Down to the stuck teammate and directly tell it to mark the task complete.

### Risk 3: Lead dies = everything dies

**Impact:** Single point of failure. Lead crash kills all coordination.

**Mitigation:**
1. **tmux session persistence.** With tmux backend, teammates run in tmux panes. If the lead process dies but tmux survives, panes stay alive. Mark can `tmux attach` and interact with teammates directly.
2. **Failsafe: fall back to manual terminals.** If Agent Teams proves unstable, Mark opens terminals in the surviving tmux panes and continues manually. This is our current model — it's the proven fallback.

### Risk 4: Cross-repo CLAUDE.md blindness

**Impact:** Teammate working in milo-for-ppc doesn't have that repo's CLAUDE.md rules. Could violate repo-specific patterns.

**Mitigation:**
1. **Subagent definitions.** Define `.claude/agents/coder-1-extraction.md` with repo-specific rules baked in. When spawned as a teammate, these instructions append to the system prompt.
2. **Spawn prompt includes `cat` of target CLAUDE.md.** Lead runs `cat ~/Documents/GitHub/milo-for-ppc/CLAUDE.md` and includes the output in the spawn prompt.

### Risk 5: Permission prompts bubble up

> Teammate permission requests bubble up to the lead, which can create friction.

**Impact:** If teammates hit permission prompts, the lead has to approve each one, creating a bottleneck.

**Mitigation:** Already handled. Mark's `settings.local.json` has `"defaultMode": "bypassPermissions"` and the auto-accept hooks. Teammates inherit these settings. Permission prompts should be rare.

---

## 5. Migration Path

### Option A: Cutover now
- **Pro:** Immediate benefit, no parallel maintenance.
- **Con:** Cross-repo behavior untested in our specific setup. If Agent Teams breaks mid-directive, Mark has to scramble.
- **Risk:** HIGH. Experimental feature + multi-repo workflow = too many unknowns.

### Option B: Pilot one scoped task (RECOMMENDED)
- **Pro:** Validates cross-repo, tmux, task flow, and teammate coordination with a real but bounded task.
- **Con:** Requires ~1 hour of Mark's time to set up and observe.
- **Risk:** LOW. Failure = fall back to manual terminals, no work lost.
- **Pilot task:** V4 shell refinements — 3 remaining specs (footer, picker, jiggle). Self-contained, spans conductor-mark (spec reading) + milo-for-ppc (implementation). Perfect for 2-3 teammates.

### Option C: Run both in parallel
- **Pro:** Zero disruption to in-flight work.
- **Con:** Double the management overhead. Mark runs 4 terminals AND monitors Agent Teams. Defeats the purpose.
- **Risk:** MEDIUM. Confusion about which system is authoritative.

**Recommendation: Option B.** One pilot, one task, one hour. If it works, cutover. If it doesn't, we have the honest data to decide between tmux-scripted manual (Option C from the addendum) or waiting for Agent Teams to mature.

---

## 6. Documents Requiring Updates (Post-Migration)

| Document | Change Required | Effort |
|---|---|---|
| **AGENT-SCOPES/README.md** | Lane charters → subagent definition files (`.claude/agents/coder-N.md`). Protocol section rewritten for Agent Teams task flow instead of file-based message bus. | HIGH |
| **AGENT-SCOPES/coder-{1-4}.md** | Convert to subagent definitions at `.claude/agents/coder-{1-4}-{lane}.md`. Same content, different format (frontmatter: `name`, `model`, `tools`). | MEDIUM |
| **TASKS.md** | Becomes a read-only snapshot generated by Ops teammate. Not the primary coordination mechanism — shared task list replaces it. Keep for Strategy-Claude consumption. | LOW |
| **DIRECTIVE-LEDGER.md** | Add `Team Task ID` column. Convention stays otherwise. | LOW |
| **STRATEGY-CLAUDE-DRIFT-GUARDS.md** | Guards 1-3 (lane check, idle coder, brief responses) still apply — the lead IS Strategy-Claude's proxy. Add: Guard 11: "verify teammate count matches active directive count." Remove: any guards about paste-relay timing. | LOW |
| **CLAUDE.md** | Add section: "Agent Teams protocol — teammates read this file at spawn. Cross-repo rules embedded in subagent definitions." | LOW |
| **Mirror URL convention** | Unchanged. Teammates commit reports with the mirror URL line. The lead verifies. | NONE |

---

## 7. Addendum: Three-Setup Comparison

### Setup A: Agent Teams + In-Process Backend

```
┌──────────────────────────────────────┐
│ VS Code Terminal                      │
│ Team Lead (visible)                   │
│ Shift+Down to cycle 4 hidden mates   │
└──────────────────────────────────────┘
```

| Metric | Value |
|---|---|
| Paste reduction from current | ~95% (lead coordinates, Mark only pastes initial directives) |
| Setup effort | 5 minutes (set env var, start session) |
| Visibility | LOW — one pane, Shift+Down to peek at teammates |
| Failure modes | Lead crash kills all. No visual indicator of teammate progress. Mark can't see if a teammate is stuck without cycling through. |

### Setup B: Agent Teams + tmux Backend

```
┌──────────┬──────────┬──────────┬──────────┬──────────┐
│ Lead     │ Coder-1  │ Coder-2  │ Coder-3  │ Coder-4  │
│ conducts │ extract  │ migrate  │ research │ ops      │
│          │          │          │          │          │
└──────────┴──────────┴──────────┴──────────┴──────────┘
tmux session (standalone terminal, NOT VS Code)
```

| Metric | Value |
|---|---|
| Paste reduction from current | ~95% (same as A) |
| Setup effort | 15 minutes (set env var, exit VS Code terminal, open standalone terminal, start tmux, start session) |
| Visibility | HIGH — all 5 panes visible simultaneously |
| Failure modes | Lead crash: tmux panes may survive (tmux session persists), but teammates lose coordination. Cross-repo: teammates lack target CLAUDE.md. VS Code terminal incompatible — must use standalone terminal. |

### Setup C: tmux-Scripted Manual (No Agent Teams)

```
┌──────────┬──────────┬──────────┬──────────┐
│ Coder-1  │ Coder-2  │ Coder-3  │ Coder-4  │
│ extract  │ migrate  │ research │ ops      │
│          │          │          │          │
└──────────┴──────────┴──────────┴──────────┘
tmux session with send-keys scripts

# Example: send directive to Coder-1's pane without copy-paste
tmux send-keys -t coders:1 "D167: Extract @milo/billing from milo-ops..." Enter
```

| Metric | Value |
|---|---|
| Paste reduction from current | ~70% (scripts send text to panes, but Mark still reads output and decides next steps manually) |
| Setup effort | 30 minutes (write tmux session script, write send-directive helper, test) |
| Visibility | HIGH — all 4 panes visible |
| Failure modes | Pane crash: only affects that one Coder. Other 3 continue. No single point of failure. Cross-repo: each pane starts in its own repo with that repo's CLAUDE.md. Most resilient option. |

### Side-by-Side Summary

| | A: In-Process | B: tmux Teams | C: tmux Manual |
|---|---|---|---|
| Paste reduction | ~95% | ~95% | ~70% |
| Setup effort | 5 min | 15 min | 30 min |
| Visibility | Low | High | High |
| Cross-repo | Workarounds needed | Workarounds needed | Native (each pane in its repo) |
| Single point of failure | Lead | Lead | None |
| Auto-coordination | Yes | Yes | No (Mark dispatches) |
| Session resilience | Poor | Medium (tmux persists) | High (independent panes) |
| VS Code compatible | Yes | No | No |
| Feature maturity | Experimental | Experimental | Proven (tmux is 30 years old) |

### Honest Assessment

**If cross-repo is the primary concern** (and it is — our work spans 5 repos), **Setup C is the most reliable today.** Each Coder pane starts in its target repo, reads that repo's CLAUDE.md, has that repo's git context. No workarounds needed.

**If Mark's time is the primary concern** (relay fatigue from 8+ hour sessions), **Setup B is the biggest win.** Eliminates manual relay entirely.

**The pragmatic path:** Pilot Setup B (Agent Teams + tmux) for one bounded task. If cross-repo friction is tolerable, migrate. If not, implement Setup C (tmux-scripted manual) as the immediate upgrade — it still eliminates most copy-paste via `tmux send-keys` scripts, and it works today with zero experimental features.

---

## 8. Test Plan: Pilot Task

### Task: V4 Shell Refinement — SyncFooter (Spec 02)

**Why this task:** Self-contained, spans 2 repos (conductor-mark for spec, milo-for-ppc for implementation), has a clear deliverable (working SyncFooter component), and Mark just approved the visual assumptions.

### Step-by-step command sequence

```bash
# 1. Open standalone terminal (NOT VS Code)
# 2. Enable Agent Teams
export CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1

# 3. Clean up stale task dirs
rm -rf ~/.claude/tasks/ce87a579-577b-4e1b-b501-db7deecd5114
rm -rf ~/.claude/tasks/fd72ed8d-e28a-4b96-8978-2061d7991f57

# 4. Start team lead with tmux backend from conductor-mark
cd ~/mark\ conduct/conductor-mark
claude --teammate-mode tmux
```

**Step 5: Give the lead this prompt:**

```text
Create an agent team for a V4 shell refinement task. Spawn 2 teammates:

1. "implementer" — Coder-1 role. Works in ~/Documents/GitHub/milo-for-ppc.
   Task: Implement the SyncFooter component per spec at
   reports/v4-shell-refinements/00-visual-assumptions-batch.md, Section 2.
   Must follow CLAUDE.md rules (TypeScript strict, no emojis, dark mode design system).
   Commit small and often. Run npm run build before reporting done.

2. "verifier" — Coder-4 role. Works in ~/mark conduct/conductor-mark.
   Task: After implementer commits, visually verify the SyncFooter on
   tlp.justmilo.app (or localhost:3000). Take desktop + mobile screenshots,
   save to ~/Desktop/. Write verification report.
   
Make verifier's task depend on implementer's task.
```

### Expected output at each step

| Step | Expected |
|---|---|
| 1-4 | tmux session opens with lead pane |
| 5 (prompt) | Lead creates team, spawns 2 tmux panes (implementer + verifier) |
| — | Lead creates 2 tasks in shared task list, verifier blocked on implementer |
| — | Implementer claims task, `cd`s to milo-for-ppc, reads spec, starts coding |
| — | Implementer commits SyncFooter component, marks task complete |
| — | Verifier unblocks, runs visual verification, saves screenshots to Desktop |
| — | Verifier marks task complete, lead synthesizes results |
| — | Lead reports: "Team complete. SyncFooter implemented and verified." |

### Pass criteria

1. Implementer successfully `cd`s to milo-for-ppc and runs `npm run build` clean
2. Implementer commits code without Mark relaying any messages
3. Verifier runs independently after implementer finishes (dependency respected)
4. Screenshots appear on `~/Desktop/`
5. Mark did not have to relay a single message between teammates
6. Total wall time ≤ 45 minutes for the pair

### Fail criteria (triggers fallback to Setup C)

- Teammate cannot `cd` to milo-for-ppc (permission error or path issue)
- Teammate ignores conductor-mark's CLAUDE.md rules (TypeScript strict, etc.)
- Lead crashes and takes both teammates with it
- Task dependency not respected (verifier starts before implementer finishes)
- Mark has to intervene more than once to unstick the workflow

---

## Commits

| SHA | Description |
|---|---|
| (this commit) | D166 Agent Teams migration spec (Coder-4) |

---

Report at: https://raw.githubusercontent.com/moscaleflow/conductor-mark-public/main/reports/2026-04-22-d166-agent-teams-migration-spec.md
