---
description: "Maestro Artist — sequential feature queue processor using agent teams (Phase 1)"
argument-hint: "[max-test-retries]"
---

## User Input

```text
$ARGUMENTS
```

If a number is provided, use it as `MAX_TEST_RETRIES`. Default is **3**.

## Context7

When using any library or framework, use Context7 MCP for accurate docs. Append this instruction to every worker prompt as `{CONTEXT7}`.

---

## Orchestrator

You are the orchestrator. You coordinate only — never implement directly.

### 1. Read the Queue

1. Glob `maestro-framework/queue/*.md`. Exclude `masterplan.md`.
2. Sort remaining files by filename (numbered: `001-step.md`, `002-step.md`, …).
3. If empty, error: `"No feature .md files in maestro-framework/queue/"` and stop.
4. Read `maestro-framework/queue/masterplan.md` if it exists → `MASTER_PLAN`.
5. Read each feature file → `FEATURE_CONTENT`.

Output:

```
Features: <count>
Retries:  <MAX_TEST_RETRIES>
```

### 2. Create the Team

Use `TeamCreate` → team named `maestro-build`.

For each feature, `TaskCreate`:
- Subject: `Build: <filename without .md>`
- Description: feature content
- ActiveForm: `Building <filename>`

### 3. Process Features (Sequential)

For each feature in order:

1. `TaskUpdate` → `in_progress`. Output `[current/total] <feature-name>`.

2. Build `COMBINED_FEATURE`:
   - If `MASTER_PLAN` exists: `MASTER_PLAN + "\n\n---\n\n" + FEATURE_CONTENT`
   - Otherwise: `FEATURE_CONTENT`

3. Spawn a worker using `Task` tool:
   - `subagent_type`: `"general-purpose"`
   - `team_name`: `"maestro-build"`
   - `name`: `"worker-<NNN>"`
   - `mode`: `"bypassPermissions"`
   - `prompt`: the **Worker Prompt** below, with `{COMBINED_FEATURE}`, `{MAX_TEST_RETRIES}`, and `{CONTEXT7}` substituted

4. Wait for the worker to finish. Check output for `ALL_TESTS_PASS` or `TESTS_FAILED`.

5. **If failed**: shut down the worker, spawn a fresh retry worker with the same prompt. Record final result regardless.

6. **If passed**: record PASS.

7. Shut down the worker. `TaskUpdate` → `completed` (PASS) or note failure (FAIL).
8. Output result: `  PASS` or `  FAIL`

### 4. Summary

```
---------------------------------------------

  <feature-name>          PASS
  <feature-name>          FAIL

---------------------------------------------
SUCCESS  N/M features
```

If all passed, output: `ORCHESTRATOR_ALL_FEATURES_DONE`

If any failed: `FAILED  N/M features`


### 5. Clean Up

Shut down all teammates, then `TeamDelete`.

---

## Worker Prompt

This is the exact prompt to send to each worker teammate. Substitute `{COMBINED_FEATURE}`, `{MAX_TEST_RETRIES}`, and `{CONTEXT7}` before sending.

---

You are a Maestro build worker. Execute these 5 phases **sequentially**. Do not skip phases. Do not proceed until the current phase completes.

**PHASE 1: SPECIFY**

Use the Skill tool to invoke `speckit.specify` with args:

`{COMBINED_FEATURE}. Include comprehensive tests following Test Driven Development. {CONTEXT7}`

**PHASE 2: PLAN**

Use the Skill tool to invoke `speckit.plan` with args:

`Use subagents to research this feature in parallel: one for API and data layer research, one for architecture patterns and dependencies, one for testing strategy. Synthesize all findings into the plan. {CONTEXT7}`

**PHASE 3: TASKS**

Use the Skill tool to invoke `speckit.tasks` with no args.

**PHASE 4: IMPLEMENT**

Use the Skill tool to invoke `speckit.implement` with args:

`Use subagents to implement this feature in parallel: break the work into independent tasks and spawn subagents, each owning a different set of files to avoid conflicts. For frontend/UI components, use /frontend-design to ensure high design quality. {CONTEXT7}`

**PHASE 5: TEST**

Find the most recently modified `tasks.md` under `specs/`. Run this loop up to **{MAX_TEST_RETRIES}** attempts:

`Read <tasks_file>. Run all tests. If any test fails, fix the implementation only (don't modify tests) and re-run. Output ALL_TESTS_PASS when all tests pass or TESTS_FAILED if stuck. {CONTEXT7}`

Report `ALL_TESTS_PASS` or `TESTS_FAILED` when done.
