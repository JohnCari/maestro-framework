# Maestro-Framework

> Agentic coding orchestrator — turns a feature queue into tested, reviewed, continuously improving code

Drop feature files in `queue/`, run `/maestro.artist`, walk away.

## Architecture

Maestro uses two Claude Code parallelism features:

- **[Agent teams](https://code.claude.com/docs/en/agent-teams)** — separate Claude Code instances that coordinate via shared task lists and messaging. Used for orchestrator↔worker coordination (artist), reviewer teams (critic), and assess/impl teams (virtuoso).
- **[Subagents](https://code.claude.com/docs/en/sub-agents)** — lightweight helpers that run within a single session and report results back. Used within workers for parallel codebase research (`Explore`) and parallel implementation (`general-purpose`).

```
   PHASE 1: CREATE        PHASE 2: CRITIQUE       PHASE 3: PERFECT
┌────────────────────┐  ┌────────────────────┐  ┌────────────────────┐
│  /maestro.artist   │  │  /maestro.critic   │  │  /maestro.virtuoso │
│                    │  │                    │  │   (ralph-loop)     │
│  queue/ → team     │  │  1. run tests      │  │                    │
│    ANALYZE         │  │  2. spawn 2:       │  │  ORIENT            │
│    PLAN            │  │     conflict ck.   │  │  ASSESS (3)        │
│    IMPLEMENT       │  │     quality sweep  │  │  SELECT            │
│    TEST (3x)       │  │       security     │  │  IMPLEMENT         │
│                    │  │       quality      │  │  VALIDATE          │
│  1 worker/feature  │  │       perf         │  │  COMMIT            │
│  all in parallel   │  │  3. validate       │  │  UPDATE PLAN       │
└────────────────────┘  │  self-heals (3x)   │  │  ↻ loops           │
                        └────────────────────┘  └────────────────────┘

Agent teams: orchestrator ↔ workers, reviewer team, assess/impl teams
Subagents:   within workers for parallel research & implementation
```

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
/maestro.artist

# 5. Review (interactive — cross-feature quality)
/maestro.critic

# 6. Refine (perpetual — runs for hours/days)
claude --dangerously-skip-permissions
/ralph-loop "/maestro.virtuoso" --completion-promise "ALL_IMPROVEMENTS_COMPLETE"
```

---

### Phase 1 — Artist

`/maestro.artist` reads `queue/*.md` files, creates a `maestro-build` **agent team**, and spawns all workers in parallel — one per feature. Workers communicate with each other via `SendMessage` to coordinate on shared interfaces, announce file ownership, and avoid conflicts. `masterplan.md` is prepended to every feature for shared context.

Each worker runs 4 phases natively, using **subagents** for parallel research and **teammate messaging** for cross-feature coordination:

1. **ANALYZE** — reads `CLAUDE.md` for project standards and available tools, uses `Explore` subagents to research the codebase, reviews the team roster for potential overlaps
2. **PLAN** — designs the implementation approach using parallel `Explore` subagents, then broadcasts file ownership and coordinates shared interfaces with relevant teammates
3. **IMPLEMENT** — builds the feature with parallel `general-purpose` subagents, each owning separate files. Messages teammates before creating shared interfaces. TDD: tests first, then implementation
4. **TEST** — runs all tests, retries up to 3 times (configurable), fixes implementation only

If a feature fails after retries, a fresh worker is spawned for one more attempt.

### Phase 2 — Critic

`/maestro.critic` reads `CLAUDE.md` for project standards, runs the full test suite first to establish a clean baseline, then spawns a `reviewer` **agent team** with 2 teammates:

- **conflict-checker** — finds and fixes cross-feature conflicts:
  - Duplicate routes or API endpoints
  - Naming collisions (functions, variables, CSS classes)
  - Circular imports and import errors
  - Conflicting global state or shared resources
  - Inconsistent data models across features

- **quality-sweep** — sequential deep scan across the entire codebase:
  - **Security** — OWASP Top 10: injection, auth gaps, data exposure, CSRF
  - **Code quality** — bugs, dead code, unused imports, inconsistent error handling
  - **Performance** — N+1 queries, unnecessary re-renders, missing indexes, large imports

Both teammates fix issues they find. Final validation re-runs all tests. Self-healing: if validation fails, the entire team is recreated — up to 3 total attempts.

### Phase 3 — Virtuoso

`/maestro.virtuoso` runs inside a Ralph Loop for continuous improvement. `IMPROVEMENT_PLAN.md` is shared state between iterations. Each iteration runs 8 phases:

| Phase | What it does |
|---|---|
| **ORIENT** | Reads CLAUDE.md, AGENTS.md, IMPROVEMENT_PLAN.md, git history. Uses `Explore` **subagents** to study the codebase |
| **ASSESS** | Spawns an **agent team** with 3 read-only analysts: code, test, quality. CLAUDE.md violations are auto-Critical |
| **SELECT** | Picks highest-priority parallel batch (up to 3-4 tasks) from the plan |
| **IMPLEMENT** | **Agent team** with file-ownership boundaries — no overlap between teammates |
| **VALIDATE** | Single-agent backpressure: full test suite, typecheck, lint, CLAUDE.md gates. Up to 3 self-healing attempts |
| **COMMIT** | One commit per logical change with clear "why" messages |
| **UPDATE PLAN** | Marks tasks done, adds newly discovered improvements |
| **EXIT** | Outputs `ALL_IMPROVEMENTS_COMPLETE` only when genuinely done, otherwise loops |

---

## Setup

1. [Claude Code](https://docs.anthropic.com/en/docs/claude-code) (native install) — includes built-in [subagents](https://code.claude.com/docs/en/sub-agents) (`Explore`, `general-purpose`, `Plan`) with no additional setup
2. Enable [agent teams](https://code.claude.com/docs/en/agent-teams) (experimental) in `~/.claude/settings.json`:
   ```json
   { "env": { "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1" } }
   ```
3. [Ralph Loop plugin](https://marketplace.anthropic.com) (for Phase 3)
4. Install:
   ```bash
   git clone https://github.com/JohnCari/maestro-framework.git maestro
   cp maestro/maestro.artist.md .claude/commands/maestro.artist.md
   cp maestro/maestro.critic.md .claude/commands/maestro.critic.md
   cp maestro/maestro.virtuoso.md .claude/commands/maestro.virtuoso.md
   ```

Any MCP servers, plugins, or skills should be configured in your project's `CLAUDE.md` — maestro will discover and use them automatically.

## License

CC0 — Public Domain
