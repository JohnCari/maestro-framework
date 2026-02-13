# Maestro Framework

> Agentic coding orchestrator — turns a feature queue into tested, reviewed, continuously improving code.

Drop feature files in `queue/`, run `/maestro-artist`, walk away.

---

## Architecture

Maestro runs three phases sequentially. Each phase uses [agent teams](https://code.claude.com/docs/en/agent-teams) for parallel coordination and [subagents](https://code.claude.com/docs/en/sub-agents) for parallel research and implementation within a single session.

```
  ┌──────────────────────────────────────────────────────────────────────────────────────────────┐
  │                                       MAESTRO PIPELINE                                       │
  └──────────────────────────────────────────────────────────────────────────────────────────────┘

  ┌─ PHASE 1 ──────────────────┐   ┌─ PHASE 2 ──────────────────┐   ┌─ PHASE 3 ──────────────────┐
  │                            │   │                            │   │                            │
  │  /maestro-artist           │   │  /maestro-critic           │   │  /maestro-virtuoso         │
  │  CREATE                    │   │  CRITIQUE                  │   │  PERFECT                   │
  │                            │   │                            │   │                            │
  │  queue/*.md                │   │  run full test suite       │   │  ORIENT ── study project   │
  │      │                     │   │      │                     │   │  ASSESS ── 3 analysts      │
  │      ▼                     │   │      ▼                     │   │  SELECT ── pick batch      │
  │  ┌─────────┐               │   │  ┌─────────┐               │   │  IMPLEMENT ── agent team   │
  │  │  team:  │               │   │  │  team:  │               │   │  VALIDATE ── backpressure  │
  │  │  build  │               │   │  │ reviewer│               │   │  COMMIT                    │
  │  └────┬────┘               │   │  └────┬────┘               │   │  UPDATE PLAN               │
  │       │                    │   │       │                    │   │  EXIT ── loop or done      │
  │  ┌────┼────┐               │   │  ┌────┴────┐               │   │                            │
  │  ▼    ▼    ▼               │   │  ▼         ▼               │   │  shared state:             │
  │ w-1  w-2  w-3              │   │ conflict  quality          │   │  IMPROVEMENT_PLAN.md       │
  │  │    │    │               │   │ checker   sweep            │   │                            │
  │  ▼    ▼    ▼               │   │             │              │   │  ↻ ralph-loop              │
  │ ANALYZE                    │   │        sec / qual / perf   │   │    fresh context each      │
  │ PLAN ──── coordinate       │   │             │              │   │    iteration               │
  │ IMPLEMENT ── TDD           │   │      validate + commit     │   │                            │
  │ TEST                       │   │             │              │   │                            │
  │ COMMIT                     │   │  ↻ ralph-loop              │   │                            │
  │                            │   │                            │   │                            │
  │ retries: orchestrator      │   │ retries: ralph-loop        │   │ retries: ralph-loop        │
  │ spawns fresh workers       │   │ fresh context window       │   │ fresh context window       │
  │                            │   │                            │   │                            │
  └────────────────────────────┘   └────────────────────────────┘   └────────────────────────────┘

  Parallelism: agent teams (multi-session), subagents (within-session research & impl)
```

---

## Quick Start

```bash
# 1. Set up CLAUDE.md with project principles, build/test commands, quality standards,
#    and any MCP servers, plugins, or skills available to the project

# 2. In Claude Code plan mode, create your plan and save as masterplan.md
#    Save to queue/masterplan.md

# 3. In the same session, ask Claude to break the masterplan into features
#    Creates NNN-name.md files in queue/
#    e.g. 001-auth.md, 002-dashboard.md, 003-settings.md

# 4. Build (autonomous — walk away)
/maestro-artist

# 5. Review (autonomous — cross-feature quality)
claude --dangerously-skip-permissions
/ralph-loop "/maestro-critic" --completion-promise "ALL_TESTS_PASS"

# 6. Refine (perpetual — runs for hours/days)
claude --dangerously-skip-permissions
/ralph-loop "/maestro-virtuoso" --completion-promise "ALL_IMPROVEMENTS_COMPLETE"
```

---

## Phase Details

### Phase 1 — Artist

`/maestro-artist` reads `CLAUDE.md` for project standards, then reads `queue/*.md` files, creates a `maestro-build` **agent team**, and spawns workers in parallel — one per feature. Workers communicate via `SendMessage` to coordinate shared interfaces, announce file ownership, and avoid conflicts. `masterplan.md` is prepended to every feature for shared context.

Features with `depends-on:` lines are spawned in dependency order — independent features first, dependent features after their dependencies complete.

Each worker runs 5 phases:

| Phase | What it does |
|---|---|
| **ANALYZE** | Reads `CLAUDE.md`, uses `Explore` subagents to research the codebase, reviews team roster for overlaps |
| **PLAN** | Designs implementation with parallel `Explore` subagents, broadcasts file ownership, coordinates shared interfaces |
| **IMPLEMENT** | Builds with parallel `general-purpose` subagents, each owning separate files. TDD: tests first |
| **TEST** | Runs feature-scoped tests only (not full suite — other workers are mid-build) |
| **COMMIT** | Commits changes on success for crash safety |

If tests fail, the orchestrator spawns a fresh worker with clean context (up to 3 retries, configurable).

### Phase 2 — Critic

`/maestro-critic` runs inside a Ralph Loop for fresh-context retries. Each iteration reads `CLAUDE.md` and `queue/*.md` feature files for context, runs the **full test suite** to establish a clean baseline, then spawns a `reviewer` **agent team** with 2 teammates:

| Teammate | Focus |
|---|---|
| **conflict-checker** | Duplicate routes, naming collisions, circular imports, conflicting global state, inconsistent data models |
| **quality-sweep** | **Security** (OWASP Top 10) → **Code quality** (bugs, dead code, error handling) → **Performance** (N+1 queries, bundle size, indexes) |

Both teammates fix issues they find. Final validation re-runs all tests and commits on success. If validation fails, the iteration exits and ralph-loop restarts with fresh context.

### Phase 3 — Virtuoso

`/maestro-virtuoso` runs inside a Ralph Loop for continuous improvement. `IMPROVEMENT_PLAN.md` is shared state between iterations. Each iteration runs 8 phases:

| Phase | What it does |
|---|---|
| **ORIENT** | Reads CLAUDE.md, AGENTS.md, IMPROVEMENT_PLAN.md, queue files, git history. Uses `Explore` subagents to study the codebase |
| **ASSESS** | Spawns an **agent team** with 3 read-only analysts: code, test, quality. CLAUDE.md violations are auto-Critical |
| **SELECT** | Picks highest-priority parallel batch (up to 3-4 tasks) from the plan |
| **IMPLEMENT** | **Agent team** with file-ownership boundaries — no overlap between teammates |
| **VALIDATE** | Single-agent backpressure: full test suite, typecheck, lint, CLAUDE.md gates. Up to 3 self-healing attempts |
| **COMMIT** | One commit per logical change with clear "why" messages |
| **UPDATE PLAN** | Marks tasks done, updates affected feature files and masterplan in `queue/`, creates new feature files for discoveries |
| **EXIT** | Outputs `ALL_IMPROVEMENTS_COMPLETE` only when genuinely done, otherwise loops |

---

## Setup

1. [Claude Code](https://docs.anthropic.com/en/docs/claude-code) (native install) — includes built-in [subagents](https://code.claude.com/docs/en/sub-agents) (`Explore`, `general-purpose`, `Plan`)
2. Enable [agent teams](https://code.claude.com/docs/en/agent-teams) (experimental):
   ```json
   // ~/.claude/settings.json
   { "env": { "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1" } }
   ```
3. [Ralph Loop plugin](https://marketplace.anthropic.com) (for Phases 2 & 3)
4. Install skills:
   ```bash
   git clone https://github.com/JohnCari/maestro-framework.git maestro
   cp -r maestro/skills/maestro-artist .claude/skills/maestro-artist
   cp -r maestro/skills/maestro-critic .claude/skills/maestro-critic
   cp -r maestro/skills/maestro-virtuoso .claude/skills/maestro-virtuoso
   ```

Configure MCP servers, plugins, or additional skills in your project's `CLAUDE.md` — maestro discovers and uses them automatically.

---

## License

CC0 — Public Domain
