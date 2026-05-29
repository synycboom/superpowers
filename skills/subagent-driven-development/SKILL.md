---
name: subagent-driven-development
description: Use when executing implementation plans with independent tasks in the current session
---

# Subagent-Driven Development

Execute plan by dispatching background tasks per task, with two-stage review after each: spec compliance review first, then code quality review.

**Why background tasks:** Every task() call uses `run_in_background=true`. The main session NEVER blocks waiting for a subagent. You dispatch, end your response, wait for `<system-reminder>` notification, then collect with `background_output()`. This keeps the main session context clean and free.

**Core principle:** All work is background tasks + notifications. Nothing blocks. Fresh context per task = no drift.

## Background Task Pattern (APPLIES TO EVERYTHING)

```
You                          Background Task
 |                                |
 |-- task(run_in_background=true) -->|
 |   STOP. End response.           |
 |                            [working...]
 |<-- <system-reminder> -----------|
 |
 |-- background_output(task_id) → read result
 |-- dispatch next background task → STOP
```

<BACKGROUND_TASK_RULES>
RULE 1: EVERY task() call in this skill uses run_in_background=true. No exceptions.
RULE 2: After ANY task() call, STOP immediately. End your response. No summary, no next steps.
RULE 3: Never call background_output() without a prior <system-reminder> for that task_id.
RULE 4: For parallel batches: ALL task() calls in ONE response, then STOP.
RULE 5: For sequential steps: ONE task() call per response, then STOP. Wait for notification. Then proceed.
RULE 6: Never call task() synchronously (run_in_background=false). If you feel the urge to wait inline, use background task + notification instead.
</BACKGROUND_TASK_RULES>

## When to Use

**Use when:**
- Have implementation plan with defined tasks
- Tasks are mostly independent
- Want to stay in this session

**vs. dispatching-parallel-agents:**
- Use dispatching-parallel-agents for pure parallel investigation with no review
- Use subagent-driven-development when tasks need review cycles

## The Process

### Step 0: Read Plan and Create Todos

1. Read plan file
2. Extract all tasks with full text
3. Create TodoWrite with one item per task
4. Note dependencies between tasks

### Step 1: Identify Parallelizable Tasks

Scan the task list for **independent tasks** — tasks that:
- Touch different files
- Have no data dependencies on each other

Group into **parallel batches** and **sequential chains**.

### Step 2: Dispatch Implementers

#### Parallel Batch (independent tasks)

All background task() calls in ONE response:

```
task(subagent_type="general", load_skills=["tdd"],
  run_in_background=true,
  description="Implement Task N: [name]",
  prompt="[filled implementer-prompt template]")

task(subagent_type="general", load_skills=["tdd"],
  run_in_background=true,
  description="Implement Task M: [name]",
  prompt="[filled implementer-prompt template]")

task(subagent_type="general", load_skills=["tdd"],
  run_in_background=true,
  description="Implement Task P: [name]",
  prompt="[filled implementer-prompt template]")

← STOP. End response. Wait for ALL <system-reminder> notifications.
```

When ALL complete → collect all with background_output() → proceed to Step 3.

#### Sequential Chain (dependent tasks)

One background task per response:

```
task(subagent_type="general", load_skills=["tdd"],
  run_in_background=true,
  description="Implement Task N: [name]",
  prompt="[filled implementer-prompt template]")

← STOP. Wait for <system-reminder>.
```

When notified → background_output() → proceed to Step 3 for this task.

#### Implementer asks questions?

Resume via task_id as background task:

```
task(task_id="ses_xxx",
  run_in_background=true,
  prompt="[answers to questions]")

← STOP. Wait for <system-reminder>.
```

### Step 3: Spec Compliance Review

After implementer completes, dispatch reviewer as background task:

```
task(subagent_type="general", load_skills=[],
  run_in_background=true,
  description="Spec review Task N",
  prompt="[filled spec-reviewer-prompt template]")

← STOP. Wait for <system-reminder>.
```

When notified → background_output():
- ❌ Issues found → resume implementer: `task(task_id="ses_implementer", run_in_background=true, prompt="Fix: [issues]")` → STOP → wait
- ✅ Passes → proceed to Step 4

### Step 4: Code Quality Review

```
task(subagent_type="general", load_skills=["requesting-code-review"],
  run_in_background=true,
  description="Quality review Task N",
  prompt="[filled code-quality-reviewer-prompt template]")

← STOP. Wait for <system-reminder>.
```

When notified → background_output():
- Critical/Important issues → resume implementer as background task → wait → re-review
- ✅ Passes → mark TodoWrite complete → next task/batch

### Step 5: Final Review

After all tasks complete, dispatch final reviewer as background task, then use superpowers:finishing-a-development-branch.

## Prompt Templates

### Implementer Prompt (./implementer-prompt.md)

```
You are implementing Task N: [task name]

## Task Description
[FULL TEXT of task from plan - paste it here, don't make subagent read file]

## Context
[Scene-setting: where this fits, dependencies, architectural context]

## Before You Begin
If you have questions about requirements, approach, or dependencies — ask now.

## Your Job
1. Implement exactly what the task specifies
2. Write tests (TDD if task says to)
3. Verify implementation works
  4. Commit your work — use conventional commit format (`feat:`, `fix:`, `chore:`, etc.)
     NEVER include Jira story IDs (e.g. BLEAGLE-1234) in commit messages
5. Self-review: missing requirements, dead code, unclear naming
6. Report back: what you built, files changed, concerns

Work from: [directory]

## Code Organization
- Follow file structure from plan
- Each file has one clear responsibility
- If file grows beyond plan's intent: report as DONE_WITH_CONCERNS, don't split on your own
- Follow established codebase patterns

## Escalate When
- Task requires architectural decisions with multiple valid approaches
- You feel uncertain about your approach
- You've been going back and forth without progress

## Report Format
DONE: [brief summary]
FILES_CHANGED: [list]
TESTS: [pass/fail/skipped]
CONCERNS: [any issues]
```

### Spec Reviewer Prompt (./spec-reviewer-prompt.md)

```
You are reviewing whether an implementation matches its specification.

## What Was Requested
[FULL TEXT of task requirements]

## What Implementer Claims They Built
[From implementer's report]

## CRITICAL: Do Not Trust the Report
Verify by reading actual code. Do NOT take their word for completeness.

Check:
- Missing requirements: did they implement everything requested?
- Extra work: did they build things not requested?
- Misunderstandings: did they solve the wrong problem?

Report:
- ✅ Spec compliant
- ❌ Issues found: [list with file:line references]
```

### Code Quality Reviewer Prompt (./code-quality-reviewer-prompt.md)

```
Review code quality for Task N. Only run after spec compliance passes.

## What Was Implemented
[from implementer's report]

## Review Criteria
- Each file has one clear responsibility
- Units are testable independently
- File structure follows the plan
- Error handling: proper, not swallowed
- Tests: meaningful, not just happy path
- Naming: consistent with codebase
- No dead code, no commented-out code

## Report Format
Strengths: [what's good]
Issues:
- Critical: [must fix]
- Important: [should fix]
- Minor: [note for later]
Assessment: Ready to proceed / Needs fixes
```

## Parallelization Decision Matrix

| Scenario | Strategy |
|----------|----------|
| Tasks touch different files, no deps | Parallel batch — all in ONE response |
| Task B needs Task A's output | Sequential — one background task per response |
| Tasks touch same files | Sequential — avoid conflicts |
| 5+ independent tasks | Parallel batches of 3 |
| Reviews | Always sequential — one background task per response |

## Critical Rules

1. **All task() calls are background tasks** — `run_in_background=true` always
2. **STOP after every task() call** — end response, wait for notification
3. **Fresh context per task** — full prompt, don't make subagents read files
4. **Two-stage review always** — spec first, quality second
5. **Resume via task_id** — for fixes, resume the same agent not a fresh one
6. **Mark TodoWrite only after BOTH reviews pass**
