---
description: Execute an implementation plan task-by-task using fresh subagents
argument-hint: path to plan file (or leave empty to find in docs/plans/)
allowed-tools: Read, Write, Edit, Glob, Grep, Bash, AskUserQuestion, Task, TaskCreate, TaskUpdate, TaskList, Skill
---

# Plan Execution

Execute an implementation plan by spawning a fresh `planning:task-executor` subagent for each task. Execute sequentially, review after each task, and run a comprehensive code review after all tasks complete.

## MANDATORY: Fresh Subagent Per Task

> **CRITICAL — NON-NEGOTIABLE.** You MUST use the Task tool to spawn a fresh `planning:task-executor` subagent for EVERY task. Do NOT execute task steps yourself in the main conversation. Each task MUST be delegated to a fresh subagent via the Task tool. No exceptions. No shortcuts.

## Execution Process

### Step 1: Locate and Parse the Plan

1. If path provided via `$ARGUMENTS`, use it
2. Otherwise, check `docs/plans/` for plan files (exclude `completed/`). If multiple plans exist, use AskUserQuestion to let user pick
3. Read the plan file and extract:
   - **Header metadata**: Goal, Architecture, Tech Stack
   - **Task list**: All `### Task N:` sections in order
   - **Task count**: Total number of tasks

Report to user:
> "Found N tasks in the implementation plan.
>
> **Goal:** [goal from header]
>
> Starting execution..."

### Step 2: Create Tracking Tasks

Immediately after parsing, create a TaskCreate for each plan task plus a final review task:

```
TaskCreate({ subject: "Task N: [Component Name]", activeForm: "Executing Task N: [Component Name]" })
```

Also create:
```
TaskCreate({ subject: "Comprehensive code review", activeForm: "Running comprehensive code review" })
```

### Step 3: Execute Tasks Sequentially

For each task in order:

1. **Mark task `in_progress`** via TaskUpdate
2. **Report start**: "Starting Task N: [Component Name]"
3. **Spawn subagent** (MANDATORY): Use the Task tool with `subagent_type: "planning:task-executor"`. Pass:
   - Full task markdown (files, steps, code snippets)
   - Context about the overall goal
   - Instructions to follow the TDD steps exactly (if plan uses TDD format)
4. **Wait for completion**
5. **Mark task `completed`** via TaskUpdate (or keep `in_progress` on failure)
6. **Report result**

### Step 4: Quick Review After Each Task

After a task-executor completes successfully:

1. Launch a single `code:reviewer` subagent via Task tool with the list of files created/modified
2. If issues found (confidence >= 80), auto-fix using Edit/Write tools
3. Run tests again after fixes to verify nothing broke
4. Report review results:

```
Task N/M complete: [Component Name]
- Created: file1.py, file2.py
- Tests passing: 3/3
- Review: Fixed 1 issue (unused import in file1.py:3). Clean otherwise.
```

If auto-fix breaks tests, revert the fix and report as a skipped issue for comprehensive review.

### Step 5: Handle Failures

On task failure:
1. **Stop immediately** — do not proceed to next task
2. **Report**: "Task N failed: [error details]"
3. **Ask user** via AskUserQuestion:

```json
{
  "questions": [{
    "question": "Task N encountered an error. How to proceed?",
    "header": "Failure",
    "options": [
      {"label": "Retry", "description": "Re-run this task with a fresh subagent"},
      {"label": "Skip", "description": "Mark as skipped, continue to next task"},
      {"label": "Stop", "description": "End execution and investigate manually"}
    ],
    "multiSelect": false
  }]
}
```

### Step 6: Comprehensive Code Review (MANDATORY)

> **DO NOT SKIP.** After all tasks complete, invoke `code:code-review` skill for comprehensive review BEFORE reporting final summary.

1. Invoke the code review skill: `Skill("code:review", "comprehensive review of all files created/modified: [list all files]")`
2. Auto-fix issues with confidence >= 80
3. Run tests after fixes
4. If auto-fix breaks tests, revert and report as skipped
5. Report results:

```
Comprehensive Review Complete

Issues found: N (X critical, Y important)
Auto-fixed: N
Skipped (needs user input): N

Fixed:
- [description] in file.py:42 (confidence: 95)

Skipped:
- [description] in file.py:100 — [reason]
```

If skipped issues exist, ask user how to handle before final summary.

### Step 7: Final Summary

```
Execution complete!

Summary:
- Tasks completed: N/N
- Files created: X
- Files modified: Y
- Tests passing: Z
- Per-task review fixes: N
- Comprehensive review fixes: N
- Issues needing attention: N

All tasks from the implementation plan have been executed and reviewed.
```

## Progress Reporting

Report progress conversationally only — do NOT modify the plan file during execution.

## Partial Completion

If execution stops mid-way:

```
Execution paused at Task N/M.

Completed: 1-[N-1]
Failed: N
Remaining: [N+1]-M

To resume, fix the issue and run /planning:execute again.
```

Initial request: $ARGUMENTS
