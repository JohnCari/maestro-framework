---
description: "Maestro Virtuoso — perpetual improvement engine (Phase 3)"
argument-hint: "[optional focus area]"
---

## User Input

```text
$ARGUMENTS
```

You **MUST** consider the user input before proceeding (if not empty). If a focus area is provided, prioritize improvements in that area.

## Outline

Virtuoso is Phase 3 of Maestro: a perpetual improvement engine that reads everything built by the orchestrator/worker (Phase 1) and reviewer (Phase 2), finds improvements, implements them with agent teams, validates, commits, and exits — triggering the next iteration via the Ralph Loop stop-hook.

**How to run:**
```bash
claude --dangerously-skip-permissions
# Then in the session:
/ralph-loop "/virtuoso" --completion-promise "ALL_IMPROVEMENTS_COMPLETE"
```

Each iteration gets a fresh context window. `IMPROVEMENT_PLAN.md` is the shared state between iterations.

---

### Phase 1: ORIENT

Study the project thoroughly before doing anything. "Study" means read, understand, and internalize — not skim.

1. **Read `CLAUDE.md` FIRST** — this is the project's governing document. It defines core principles, coding standards, build/test commands, quality requirements, and any available MCP servers, plugins, or skills. **`CLAUDE.md` is NON-NEGOTIABLE**: any rule or principle defined there is a hard constraint that cannot be overridden. Everything you do in this iteration must comply with it. Use whatever tools are documented there throughout all phases.

2. **Read `AGENTS.md`** (if it exists) — additional operational guidance, agent-specific instructions, or codebase patterns.

3. **Read `IMPROVEMENT_PLAN.md`** (if it exists) — this is your shared state from previous iterations. If it exists, you are resuming. If not, this is the first iteration.

4. **Run `git log --oneline -50`** to discover what features have been built, what's been committed, and the project's trajectory. This prevents re-doing work or conflicting with recent changes.

5. **Study the codebase** — use subagents (Task tool with `subagent_type: "Explore"`) to scan `src/`, `tests/`, config files, and any other relevant directories. Understand existing code patterns, what features exist, and how they're structured. Don't assume something isn't implemented; search first.

6. **Study git history** — run `git log --oneline -30` to understand recent work and the project's trajectory.

---

### Phase 2: ASSESS (first iteration only, or when IMPROVEMENT_PLAN.md doesn't exist)

**Skip this phase if `IMPROVEMENT_PLAN.md` already exists and has uncompleted tasks** — unless the plan feels stale or the codebase has diverged from what the plan describes. Plans are disposable; if the plan is wrong, delete it and reassess. Regeneration is cheap.

Create an agent team named `virtuoso-assess` with 3 teammates (use `subagent_type: "Explore"` for all — they are read-only research agents). **Give every teammate the `CLAUDE.md` rules** so they can flag violations:

- **code-analyst**: Scan the entire codebase for:
  - **CLAUDE.md rule violations** — any code that breaks a rule or principle defined in CLAUDE.md (these are automatically Critical)
  - Architecture gaps and structural issues
  - Dead code and unused imports
  - Missing error handling
  - DRY violations and code duplication
  - Inconsistent patterns across features
  - Report all findings with file paths and line numbers.

- **test-analyst**: Scan all tests and test configuration for:
  - **CLAUDE.md rule violations** — missing test coverage required by standards in CLAUDE.md
  - Coverage holes (untested functions, branches, edge cases)
  - Missing edge case tests
  - Flaky or brittle test patterns
  - Integration test gaps between features
  - Missing test utilities or fixtures
  - Report all findings with file paths and line numbers.

- **quality-analyst**: Scan the entire codebase for:
  - **CLAUDE.md rule violations** — security, performance, or quality gate breaches defined in CLAUDE.md
  - Security issues (OWASP Top 10: injection, auth gaps, data exposure, CSRF)
  - Performance problems (N+1 queries, unnecessary re-renders, missing indexes, large imports)
  - Accessibility gaps
  - UX inconsistencies
  - Report all findings with file paths and severity.

Wait for all 3 teammates to finish. Send each a `shutdown_request` message, then delete the team. Synthesize their findings into `IMPROVEMENT_PLAN.md`. **Any CLAUDE.md rule violation is automatically Priority 1: Critical** — these must be fixed before anything else.

```markdown
# Improvement Plan

Generated: <timestamp>
Last updated: <timestamp> (iteration 1)

## Priority 1: Critical (includes all CLAUDE.md rule violations)
- [ ] [ID-001] CLAUDE.MD: Description | files: path/to/file.ts | verify: how to confirm it's done [parallel]
- [ ] [ID-002] Description | files: path/to/other.ts | verify: acceptance criteria

## Priority 2: High
- [ ] [ID-003] Description | files: src/auth/ | verify: tests pass [parallel]
- [ ] [ID-004] Description | files: src/api/routes.ts | verify: no N+1 queries

## Priority 3: Medium
- [ ] [ID-005] Description | files: src/utils/ | verify: no duplicated code [parallel]

## Discoveries
- Notes found during analysis
```

Each task must include: **files** it touches (for ownership boundaries) and **verify** criteria (how to confirm it's done — this is backpressure). Mark tasks with `[parallel]` if they touch independent files and can be worked on simultaneously.

---

### Phase 3: SELECT

1. Read `IMPROVEMENT_PLAN.md`
2. Pick the highest-priority batch of `[parallel]` tasks (up to 3-4 tasks)
3. If no parallel tasks remain, pick the single highest-priority uncompleted task
4. If all tasks are `[DONE]`, go back to Phase 2 to reassess

---

### Phase 4: IMPLEMENT (agent team)

Create an agent team named `virtuoso-impl` with one teammate per selected task (use `subagent_type: "general-purpose"` — they need edit/write/bash access):

- Each teammate owns specific files — **no overlap between teammates**. Use the `files:` field from `IMPROVEMENT_PLAN.md` to enforce boundaries. Same boundary-setting you'd do with a human team to avoid merge conflicts.
- Give each teammate clear instructions: which task ID, which files to modify, the `verify:` criteria, the **CLAUDE.md rules** they must not violate, and any MCP servers, plugins, or skills documented in CLAUDE.md.
- The lead (you) stays in delegate mode — coordinate only, do not implement directly

Wait for all teammates to finish. Send each a `shutdown_request` message, then delete the team.

---

### Phase 5: VALIDATE (backpressure — single agent, NOT parallel)

Run validation yourself — do NOT delegate this to teammates. This is deliberate backpressure: one agent, sequential, full suite. It prevents incomplete work from slipping through.

1. **Discover the correct commands** from `CLAUDE.md` or `AGENTS.md`. Look for test commands, build commands, typecheck/lint commands. If none are documented, discover them from `package.json`, `Makefile`, or equivalent.
2. Run the **full test suite** (all tests, not just the ones related to changes)
3. If tests fail: fix the implementation (never modify tests), re-run
4. Run typecheck/lint if available
5. **CLAUDE.md compliance check** — verify the changes don't violate any rule or principle from CLAUDE.md. If CLAUDE.md defines quality gates (e.g. test coverage thresholds, required documentation, security scans), run those gates now. CLAUDE.md violations are blockers — fix them before proceeding.
6. Self-healing loop: up to 3 attempts. If still failing after 3, note the failures in the improvement plan and move on.

---

### Phase 6: COMMIT

1. Stage all changes
2. Commit with a clear message that captures the **why**, not just the what:
   ```
   virtuoso: [ID-XXX] Why this improvement matters

   What changed and how it was verified.
   ```
   Bad: `virtuoso: [ID-003] Add tests`. Good: `virtuoso: [ID-003] Cover auth token refresh to prevent silent session expiry`
3. If multiple tasks were completed, make one commit per logical change

---

### Phase 7: UPDATE PLAN

1. Mark completed tasks as `[DONE]` in `IMPROVEMENT_PLAN.md`
2. Update the `Last updated` timestamp and iteration number
3. Add any newly discovered improvements found during implementation to the appropriate priority section
4. Note any blocked tasks or dependencies

---

### Phase 8: EXIT

Check if the work is done:

- **If ALL tasks in `IMPROVEMENT_PLAN.md` are `[DONE]`** and no new improvements were discovered during this iteration:
  Output `<promise>ALL_IMPROVEMENTS_COMPLETE</promise>`

- **Otherwise**: Exit normally. The Ralph Loop stop-hook will catch the exit and feed this same prompt back for the next iteration with a fresh context window.

---

### Critical Rules

1. **Study, don't assume.** Always read existing code before proposing changes. Never assume something isn't implemented — search first. Existing code patterns guide what you generate.
2. **One batch per iteration.** Don't try to do everything in one pass. Pick a focused batch (3-4 tasks max), implement, validate, commit, exit. Fresh context next iteration. This keeps you in the smart zone (~20-40% context utilization).
3. **Never modify tests to make them pass.** Fix the implementation instead. Tests are backpressure — they define what "done" means.
4. **Only 1 agent for validation.** Tests run sequentially in one context — this is deliberate backpressure. Incomplete work fails automatically.
5. **Keep the plan updated.** `IMPROVEMENT_PLAN.md` is the bridge between iterations. If you don't update it, the next iteration starts blind.
6. **Plans are disposable.** If the plan has drifted from reality (code changed, tasks no longer make sense), delete it and re-assess. Regeneration is cheap. Don't force-fit work into a stale plan.
7. **CLAUDE.md is supreme.** `CLAUDE.md` is the project's governing document. Rules and principles defined there are non-negotiable — violations are automatically Critical. CLAUDE.md supersedes all other practices. If CLAUDE.md itself needs changing, that's a manual edit, not something the virtuoso overrides.
8. **Do not lie to exit.** Only output the completion promise when ALL improvements are genuinely done. The loop is designed to continue until true completion.
