# Conductor Architecture Research — 2026-04-28

> 5 parallel research threads consolidated. Pure research + design — no code changes.

---

## Thread 1: Morgan's Conductor Pattern

### Repo Location & Structure

Morgan's conductor lives at `/Users/markymark/Conductormark/conductor/` — pure coordination, no application code. Covers 9 registered projects (5 ConvoQC repos, ScaleFlow, milo-ops, Momi'Snacks, claude-context-kit).

Key dirs: `routines/`, `scripts/`, `audits/`, `archives/`, `references/`, `content-pipeline/`, `dashboard/`, `.claude/commands/`, `.claude/skills/`

### Lead vs Worker Separation

**Morgan:** Single operator model. Morgan IS the lead — Claude Code is one conversational partner that dispatches subagents on demand. No persistent "Strategy-Claude" role. Each dispatch is ephemeral: fresh subagent per task, dies when done.

**Mark:** Explicit 5-agent hierarchy with `strategy-claude.md` as persistent coordinator + 4 named Coders with strict lane boundaries.

### Agent Definitions

Morgan has **no agent definition files**. Uses `.claude/commands/session-start.md` (startup ritual) and `.claude/skills/` (status, content-market-intel, video-script-writer). Agent behavior is defined at dispatch time via `DISPATCH-PATTERNS.md` templates, not persistent personas.

Mark has 9 agent definitions in `.claude/agents/` with full persona rules, drift guards, and lane checks.

### Directive Flow

**Morgan:** Copy-paste dispatch templates from `DISPATCH-PATTERNS.md`. 12 modes: Build, Fix, Review, Research, Standup, Brainstorm, TDD Build, Debug, Verify, Visual Brainstorm, Trigger Audit, Scope. Quality gates (Standard for Opus, Enhanced for Sonnet with Phase 0-3) appended to code-writing dispatches. No inbox/outbox.

**Mark:** Filesystem relay system. Strategy-Claude writes to `directives/inbox/coder-N/`, Coders poll, execute, report to `outbox/coder-N/`. Front-matter with id, lane, priority, gate, depends_on.

### Session/Pane Setup

Morgan uses a VS Code workspace (`conductor.code-workspace`) with conductor + milo-ops folders. Single Claude Code session. `npm run nexus` runs a local dashboard (localhost:3333) rendering CURRENT.md as project cards. No tmux.

### What Morgan Has That Mark Doesn't

1. **Cloud-scheduled routines** (`routines/`): Morning Ops Digest (daily 7am ET, Supabase/Stripe/Railway → Slack DM), Pipeline Failure Triage (webhook-triggered, circuit breaker, draft PRs), Inbox Drafter (daily 6:30am, drafts Gmail replies). Currently paused for cost, but fully spec'd.
2. **Context generation script** (`scripts/generate-context.js`): Auto-appends "Live Context" to each project's CLAUDE.md with branch, commits, TODO counts, HANDOFF freshness. Run via `npm run context`.
3. **Nexus dashboard** (`dashboard/server.js`): Local web dashboard parsing CURRENT.md into visual project cards.
4. **Lessons Learned log**: Date, project, root cause, pattern fix for each incident. Feeds back into quality gates.
5. **Two-pass as default**: Explicit rule: Sonnet builds + Opus reviews for math, billing, auth, state machine code.

### What Mark Has That Morgan Doesn't

1. Persistent agent personas with hard scope boundaries and 23 drift guards
2. Filesystem relay system eliminating copy-paste relay
3. Formal approval gates with prelim/approved/rejected file protocol
4. Decision numbering system with TBD protocol
5. Visual verification protocol with stitched screenshots

### Reproducible Patterns for Mark

1. **Dispatch mode templates** — Morgan's 12-mode library is more granular. The quality gates caught a Sentry DSN leak and service-role key exfil. Worth extracting into conductor-mark.
2. **Session-start command** — Morgan's runs parallel git sweep across all repos + live prod queries, summarizes in 5 bullets in <45s. Mark's is prose in agent definition, not runnable.
3. **Scheduled routines spec format** — prompt, trigger, connectors, env vars, expected runtime, failure modes, success criteria. 48-hour validation checklist.
4. **Lessons Learned feedback loop** — structured incident-to-pattern-fix pipeline. Mark's reports land in outbox but no systematic pattern update.

---

## Thread 2: VS Code Multi-Pane Reality Check

### Tabs vs Split Panes vs Terminal Panes

- **Tabs**: VS Code extension opens Claude Code as editor tabs. Multiple conversations = separate tabs with independent histories. This is what Mark has today.
- **Split panes**: NOT natively supported for multiple Claude Code instances side-by-side within the extension.
- **Terminal panes**: The `claude` CLI can run in VS Code's integrated terminal. Multiple CLI instances require terminal splitting or tmux.

### The Agent Tool (Subagent Spawning)

| Question | Answer |
|---|---|
| Same conversation or fork? | Separate context window. Parent sees only the returned summary. |
| Parallel spawning? | Yes — multiple subagents can run concurrently (foreground or background). |
| Shared working directory? | Yes — subagents start in parent's cwd. |
| Can write files / commit? | Yes — same permissions as parent. |
| Background vs foreground? | Background runs concurrently; foreground blocks. Background auto-denies missing permissions. |
| Concurrent limit? | No hard limit documented. Token costs scale linearly. |
| **Can subagents nest?** | **No. Subagents cannot spawn their own subagents. Only leads can spawn.** |
| Worktree isolation? | `isolation: "worktree"` gives isolated git checkout on separate branch. Auto-cleans if no changes. |

### Agent Definitions (.claude/agents/*.md)

Format: YAML frontmatter (`name`, `description`, `tools`, `model`, `isolation`, `memory`, `permissionMode`, etc.) + markdown system prompt body. Tool control via allowlist (`tools: Read, Bash`) or denylist (`disallowedTools: Write`).

Priority: managed settings > `--agents` CLI flag > `.claude/agents/` (project) > `~/.claude/agents/` (user) > plugin agents.

### Recommended Setup

**Option C (Hybrid) — recommended for Mark:**

- **One VS Code extension tab per project** = the lead agent. Mark talks to leads.
- **Leads spawn subagents** via Agent tool for parallel work (foreground for blocking deps, background for independent tasks).
- **CLI in terminal** for heavy research/audits when needed.
- **`isolation: worktree`** for subagents doing parallel implementation to prevent file conflicts.
- Aligns with existing directive system. No tmux required. Scale from 1 subagent to 5+ as needed.

**Key constraint:** Subagents can't nest. So a lead can spawn workers, but workers can't delegate further. This means leads must plan the full fan-out — workers execute atomically.

---

## Thread 3: Platform Constitution

**Written to:** `~/mark conduct/conductor-mark/PLATFORM-CONSTITUTION.md`

8 sections, ~850 words. Distilled from STRATEGY-CLAUDE-DRIFT-GUARDS.md (23 patterns → top 5), DECISION-LOG.md (Decisions 5, 16, 20, 24, 34, 38, 77), CURRENT-STATE.md (8 primitives), VISION-MILO-V1-FUNNEL.md (philosophy/voice).

| Section | Summary |
|---|---|
| 1. Milo Engine Philosophy | Horizontal platform, vertical brands. Public face = vertical, never engine. |
| 2. Primitive List & Ownership | 8 shipped @milo/* packages with table prefixes. Extract, don't fork. |
| 3. Tenant Isolation | TEXT slug tenant_id on every row. No cross-tenant aggregation. |
| 4. Brand Voice | First-person Milo. No emojis. Dark theme mandatory. Branding in vertical.config.ts. |
| 5. Schema-Change Protocol | SCHEMA.md amendment committed BEFORE implementation (Decision 38). |
| 6. Drift Guards (Top 5) | Verify before trusting, no infra without consumers, schema-first, flag TBDs, never silently reduce scope. |
| 7. Escalation | Self-serve (bug fixes, env vars) / Ask Mark (scope, schema, billing) / Don't escalate (routine infra). |
| 8. Cross-Project Coordination | Filesystem relay, Agent Teams primary, strict lane boundaries, commit-before-idle. |

---

## Thread 4: OMADe and PPC Fight Club Discovery

### OMADe (ItsOMADe)

**What:** Physical consumer product — high-protein baking flour (24g protein/quarter cup, 8g net carbs). "Flour for Fuel." Grown on Joey Russell's farm in Coquille, Oregon. Brand plays on OMAD (One Meal A Day). Mark's own venture, not a client.

**Repo:** None. No GitHub repo, no Vercel deployment.

**Local artifacts:**
- `/Users/markymark/Desktop/OMADe/` — logo + screenshot (2026-04-23)
- `/Users/markymark/Downloads/itsomade-influencer-kit.html` — polished standalone HTML page (dark theme, gold accents)
- `/Users/markymark/Downloads/itsomade-site.zip` (+ copies) — downloaded site bundles, not deployed

**Stack:** Nothing built in Milo ecosystem yet. Influencer kit is static HTML.

**Milo primitive consumption:** None yet. Plan (per architecture report) is to reuse `@milo/ai-client`, `@milo/blacklist`, add `omade` tenant to milo-outreach for creator outreach (Instagram/TikTok DM discovery + drafting).

**Research completed:**
1. `reports/2026-04-27-coder-3-omade-engine-audit.md` — 14-section audit, 25 reusable primitives cataloged, 30 green-field capabilities needed.
2. `reports/2026-04-27-coder-3-omade-architecture-stress-test.md` — 4 architecture options evaluated, recommends **Option D (hybrid)**: add `omade` tenant to milo-outreach now, extract `@milo/outreach` only when complexity forces it.

**Status:** Decision TBD in DECISION-LOG.md. 8 open questions need Mark's answers before implementation (tenant naming, DM platforms for v1, Apify account, Shopify timing, verified sender domain, extraction trigger, post-detection scope, leads table collision).

**Day-one agent needs:** Read both Coder-3 reports, understand Option D hybrid architecture, know that no code exists yet, first directive = adding `omade` tenant row + creator lead columns to milo-outreach.

### PPC Fight Club

**Does not exist.** Zero evidence across: filesystem (all of `~` to depth 5), all GitHub repos (200 listed), all Vercel deployments, all conductor-mark coordination docs, all reports, all directives. Searched variations: "PPC Fight Club", "ppc-fight", "fight-club", "fightclub", "PPCFC", "fight". No trace.

If Mark has discussed this verbally or in an external channel, there is no local artifact.

---

## Thread 5: Sub-Agent Dispatch Rules

### Decision Tree (for lead agent definitions)

1. **Inline (no sub-agent).** Task touches 1-2 files, no ambiguity, coordination-layer work only (doc update, directive write). Lead never implements app code inline (Pattern 19) — if the "simple" task is app code, spawn 1 worker.

2. **Single worker.** Sequential build with file dependencies — later steps depend on earlier output within same repo. 1 builder owns all files. Optionally spawn 1 auditor in parallel after builder commits.

3. **2-3 parallel workers, disjoint files.** Tasks are independent — different files, different repos, no shared imports. **Pre-spawn: compute file write intersection. If non-empty → Rule 4.** (Pattern 23)

4. **Batched sequential + parallel across batches.** 3+ tasks, some share files. Group by file ownership. Within group: 1 worker, sequential. Across groups: parallel. Document file-ownership map before spawning.

5. **Research fan-out.** Question spans multiple repos. 1 sub-agent per 1-2 repos, narrow question. Lead synthesizes. Never send 1 agent to read 6 repos.

### Quick Reference

| Signal | Workers | Pattern |
|---|---|---|
| 1-2 files, coordination only | 0 (lead inline) | Rule 1 |
| Sequential build, file deps | 1 builder + optional auditor | Rule 2 |
| Independent tasks, disjoint files | 2-3 parallel | Rule 3 |
| Overlapping files across tasks | Batch by file owner, parallelize batches | Rule 4 |
| Multi-repo research question | 1 per 1-2 repos | Rule 5 |

### Pre-Spawn Checklist (mandatory)

1. List every file each sub-agent will read and write.
2. Confirm write-target intersection across sub-agents is empty.
3. If non-empty: restructure into batches (Rule 4) or sequence in single worker (Rule 2).
4. Include file-ownership map in each sub-agent's prompt.

### Anti-Patterns

- Don't spawn 12 when 3 will do (coordination overhead grows non-linearly with 4+ workers)
- Don't sequence when parallel is safe (wastes Mark's wall-clock time)
- Don't let lead implement app code (Pattern 19 — lead coordinates, workers build)
- Don't fan out before Mark sees scope decisions
- Don't spawn research agents for single-file answers (use Explore agent)
- Don't fan out without declaring each worker's file write scope (Pattern 23 — last-write-wins is silent data loss)

### Cost/Benefit

Each sub-agent costs ~$0.50-2.00 in tokens. Spawn parallel workers when it saves Mark more than 5 minutes of waiting. Under 5 minutes → sequential or inline is cheaper.

---

## Recommended Architecture

Based on all 5 threads, here's what Mark's setup should look like.

### Daily Startup Flow

1. Open VS Code. One tab per active project lead (e.g., `lead-ppc`, `lead-ohmade`, `lead-conductor`).
2. Each lead reads `PLATFORM-CONSTITUTION.md` on startup (baked into agent definition).
3. Lead runs a session-start command (adapt from Morgan's pattern) — parallel git sweep of its project repos, read CURRENT-STATE.md, summarize in 5 bullets.
4. Mark tells a lead what to do. Lead spawns workers via Agent tool.

### File Structure

```
conductor-mark/
├── PLATFORM-CONSTITUTION.md          # Every lead reads on startup (written today)
├── CURRENT-STATE.md                  # Global state snapshot
├── DECISION-LOG.md                   # Architecture decisions
├── .claude/
│   ├── agents/
│   │   ├── lead-ppc.md               # Lead for milo-for-ppc (spawns workers)
│   │   ├── lead-ohmade.md            # Lead for OMADe (when ready)
│   │   ├── lead-conductor.md         # Lead for conductor-mark ops
│   │   ├── worker-builder.md         # Generic implementation worker
│   │   ├── worker-researcher.md      # Read-only research worker
│   │   └── worker-auditor.md         # Review/test worker
│   ├── commands/
│   │   └── session-start.md          # Runnable startup sweep (adapt from Morgan)
│   └── settings.json
├── directives/
│   ├── inbox/                        # Relay system (fallback for non-Agent-Teams mode)
│   └── outbox/
└── reports/
```

### Lead Agent Template

Each `lead-*.md` includes:
- Reference to `PLATFORM-CONSTITUTION.md` (read on startup)
- Project-specific repos and file scope
- Sub-Agent Dispatch Rules (from Thread 5)
- Session-start instructions (git sweep, state read)
- Escalation rules (from constitution)

### Key Design Decisions

1. **One lead per project, not one lead per lane.** Current 4-Coder setup is lane-oriented (extraction, migration, research, ops). New model is project-oriented — lead-ppc owns all lanes for PPC, spawns typed workers as needed.

2. **Workers are generic, not named.** Instead of "Coder-1" through "Coder-4", workers are `worker-builder`, `worker-researcher`, `worker-auditor`. A lead spawns whichever type it needs. Workers die when their task is done.

3. **Constitution replaces per-agent drift guards.** Instead of copying 150 lines of rules into every agent definition, leads read `PLATFORM-CONSTITUTION.md`. Agent definitions stay lean (30-50 lines of project-specific context).

4. **Relay system is fallback only.** In Agent Teams mode (primary), leads spawn workers directly — no inbox/outbox needed. Relay system remains for tmux fallback and cross-project coordination.

5. **Subagent nesting is impossible.** Workers can't spawn sub-workers (Claude Code limitation). Leads must plan the full fan-out. This makes the dispatch decision tree critical — leads can't delegate decomposition.

### Migration Path (from current to recommended)

1. Write `lead-ppc.md` agent definition (incorporates current Coder-1/2/3/4 scope for PPC)
2. Write generic worker definitions (`worker-builder.md`, `worker-researcher.md`, `worker-auditor.md`)
3. Write `.claude/commands/session-start.md` (adapt Morgan's git sweep pattern)
4. Test: open one VS Code tab as lead-ppc, spawn 3 workers in parallel for a real directive
5. When OMADe is ready: write `lead-ohmade.md`, same template + OMADe-specific context
6. Retire Coder-1/2/3/4 definitions once project-oriented leads are proven

---

*Research completed 2026-04-28. No code changes. 5 threads, 5 sub-agents, ~$4-8 total token cost.*
