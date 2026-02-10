# Maestro

> Agentic coding orchestrator — turns a feature queue into tested, reviewed, continuously improving code

Drop feature files in `queue/`, run `/maestro.artist`, walk away.

## Architecture

```
        PHASE 1: CREATE               PHASE 2: CRITIQUE              PHASE 3: PERFECT
┌────────────────────────────┐  ┌────────────────────────────┐  ┌────────────────────────────┐
│                            │  │                            │  │                            │
│  /maestro.artist           │  │  /maestro.critic           │  │  /maestro.virtuoso         │
│                            │  │                            │  │  (via ralph-loop)          │
│  queue/ → agent team       │  │  1. run all tests          │  │                            │
│    ├ SPECIFY               │  │  2. spawn 2 agents:        │  │  per iteration:            │
│    ├ PLAN                  │  │     conflict-checker       │  │    ORIENT                  │
│    ├ TASKS                 │  │     quality-sweep          │  │    ASSESS  (3 agents)      │
│    ├ IMPLEMENT             │  │       ├ security           │  │    SELECT                  │
│    └ TEST                  │  │       ├ code quality       │  │    IMPLEMENT (team)        │
│                            │  │       └ performance        │  │    VALIDATE  (tests)       │
│  sequential, retry once    │  │  3. final validation       │  │    COMMIT                  │
│  agent teams orchestrated  │  │  self-healing, 3 attempts  │  │    UPDATE PLAN             │
│                            │  │                            │  │  ↻ loops for hours/days    │
└────────────────────────────┘  └────────────────────────────┘  └────────────────────────────┘
```

## Quick Start

```bash
# 1. Set up your project constitution (principles, quality gates)
/speckit.constitution

# 2. In Claude Code plan mode, create your plan and save as masterplan.md
#    Save to maestro/queue/masterplan.md

# 3. In the same session, ask Claude to break the masterplan into features
#    Creates NNN-name.md files in maestro/queue/
#    e.g. 001-auth.md, 002-dashboard.md, 003-settings.md

# 4. Build (autonomous — walk away)
/maestro.artist

# 5. Review (interactive — cross-feature quality)
/maestro.critic

# 6. Refine (perpetual — runs for hours/days)
claude --dangerously-skip-permissions
/ralph-loop "/maestro.virtuoso" --completion-promise "ALL_IMPROVEMENTS_COMPLETE"
```

**Phase 1 — Artist:** `/maestro.artist` creates an agent team and processes each feature through SPECIFY → PLAN → TASKS → IMPLEMENT → TEST. Failed features retry once. Each worker gets a fresh context window.

**Phase 2 — Critic:** `/maestro.critic` spawns a conflict-checker + quality-sweep team to catch cross-feature issues (duplicate routes, security, perf). Self-healing, up to 3 attempts.

**Phase 3 — Virtuoso:** `/maestro.virtuoso` runs inside a Ralph Loop. Each iteration reads all artifacts, picks improvements from `IMPROVEMENT_PLAN.md`, implements with agent teams, validates, commits, exits — loop restarts with fresh context.

## Setup

1. [Claude Code](https://docs.anthropic.com/en/docs/claude-code) (native install)
2. [Spec Kit](https://github.com/github/spec-kit#installation)
3. [Context7 MCP](https://github.com/upstash/context7#claude-code)
4. [Ralph Loop plugin](https://marketplace.anthropic.com) (for Phase 3)
5. `/frontend-design` skill from [Anthropic Marketplace](https://marketplace.anthropic.com)
6. Enable agent teams in `~/.claude/settings.json`:
   ```json
   { "env": { "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1" } }
   ```
7. Install:
   ```bash
   git clone https://github.com/JohnCari/maestro-framework.git maestro
   cp maestro/maestro.artist.md .claude/commands/maestro.artist.md
   cp maestro/maestro.critic.md .claude/commands/maestro.critic.md
   cp maestro/maestro.virtuoso.md .claude/commands/maestro.virtuoso.md
   ```

## License

CC0 — Public Domain
