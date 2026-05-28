---
name: subagent-driven-development-team
description: Use when executing implementation plans with OMO team-mode. Requires team_mode.enabled=true. Persistent agents with mailbox communication — better than background tasks for plans with 4+ tasks or complex review/fix loops.
---

# Subagent-Driven Development (Team Mode)

Execute plans using OMO team-mode: persistent member agents working in parallel, communicating via mailbox. The lead (you) coordinates without blocking. All members visible simultaneously in tmux.

**Requires:** `team_mode.enabled: true` in `oh-my-openagent.json` and running inside a tmux session.

**Use over background-task variant when:**
- Plan has 4+ independent tasks
- Expect multiple review/fix cycles (mailbox is cleaner than task_id resume)
- Want tmux visibility of all agents simultaneously

**Use background-task variant when:**
- Plan has 1-3 tasks
- No tmux available
- `team_mode` not enabled

## Core Concept

```
You (Lead/Sisyphus)
  │
  ├── team_create(impl-sprint, members=[task-2, task-4, task-6])
  │     ↓ members start immediately with their prompts
  │
  ├── team_status() → wait for all idle
  │
  ├── team_send_message(to=task-2, "spec review yourself")
  ├── team_send_message(to=task-4, "spec review yourself")
  ├── team_send_message(to=task-6, "spec review yourself")
  │     ↓ members review and respond via mailbox
  │
  ├── team_status() → read mailbox results
  │     ↓ fix issues if any
  │
  ├── team_send_message(to=task-2, "quality review yourself")
  │   ...
  │
  └── team_delete(impl-sprint)
```

## The Process

### Step 0: Read Plan and Create Todos

1. Read plan file
2. Extract all tasks with full text
3. Create `todowrite` with one item per task
4. Identify task dependencies

### Step 1: Identify Parallel Batches

Group independent tasks (no shared files, no data deps) into batches of up to 4.
Sequential tasks (B depends on A) stay chained.

### Step 2: Run Each Batch

For each parallel batch of N tasks:

#### 2a. Create the impl team

```
team_create(
  name="impl-batch-{N}",
  members=[
    {
      kind: "category",
      category: "unspecified-high",
      name: "task-{id}",
      color: "blue",
      prompt: "[FULL implementer prompt — see template below]"
    },
    ... one per task in batch
  ]
)
```

Members start immediately. Each member's `prompt` IS their task.

#### 2b. Create shared task list

```
team_task_create(team="impl-batch-{N}", subject="Task {id}: {name}",
  description="[task summary]")
```

One per member. Members can claim and update their own task.

#### 2c. Wait for all members to finish implementing

```
team_status(team="impl-batch-{N}")
```

Poll until all members are `idle` or `completed`. Members report
`DONE: ...` in their last message when finished.

<TEAM_STATUS_RULES>
RULE 1: After team_create, call team_status to check progress.
RULE 2: If members are still `running`, wait — call team_status again
        after a short pause (do some prep work in between, don't spin).
RULE 3: Read member mailboxes after they go idle to get their reports.
RULE 4: A member is done when status=idle AND they sent a DONE message.
</TEAM_STATUS_RULES>

#### 2d. Spec compliance review (per member)

Send spec review request to each member via mailbox:

```
team_send_message(
  team="impl-batch-{N}",
  to="task-{id}",
  body="[SPEC REVIEW REQUEST — see template below]"
)
```

Wait for all to respond (`team_status`), then read mailbox.

- ✅ All pass → proceed to 2e
- ❌ Issues found → send fix request to that member:
  ```
  team_send_message(to="task-{id}",
    body="Fix these spec issues:\n[issues]\nDo not change anything else.")
  ```
  Wait for fix, then re-send spec review request.

#### 2e. Code quality review (per member)

```
team_send_message(
  team="impl-batch-{N}",
  to="task-{id}",
  body="[QUALITY REVIEW REQUEST — see template below]"
)
```

Wait, read mailbox:
- Critical/Important issues → send fix request → wait → re-review
- ✅ Passes → mark task complete:
  ```
  team_task_update(team="impl-batch-{N}", id="...", status="completed")
  ```

#### 2f. Tear down batch team

```
team_delete(team="impl-batch-{N}")
```

Mark todos complete. Proceed to next batch.

### Step 3: Sequential Tasks

For tasks that depend on a previous batch's output:
- Wait for the prior batch's team to complete and be deleted
- Create a new single-member team (or use background task)
- Same review cycle as above

### Step 4: Final Review

After all tasks complete, dispatch final reviewer:

```
task(category="unspecified-high", run_in_background=true,
  description="Final review of entire implementation",
  prompt="Review the complete implementation across all tasks.
  Base: [base SHA before any tasks]
  Head: [current HEAD]
  Run git diff base..head and review for integration issues,
  missing wiring, inconsistencies between tasks.
  Report: overall assessment + any cross-task issues.")
```

Then use `superpowers:finishing-a-development-branch`.

---

## Prompt Templates

### Implementer Prompt (member `prompt` field)

```
You are implementing Task {N}: {name} as part of a team.

## Your Task
{FULL TEXT of task from plan}

## Context
{Architectural context, where this fits, what other team members are building}

## Before You Start
If you have questions — ask them NOW before writing any code.

## Your Job
1. Implement exactly what the task specifies
2. Write tests (TDD if task says to)
3. Run tests and verify they pass
4. Commit your work with a descriptive message
5. Self-review: check for missing requirements, dead code, unclear naming

## Code Organization
- Follow file structure from plan
- Each file has one clear responsibility
- If growing beyond plan's intent: stop, report DONE_WITH_CONCERNS

## Report Format (send this when done)
DONE: [one-line summary]
FILES_CHANGED: [list]
TESTS: [pass count / total]
COMMIT: [SHA]
CONCERNS: [anything unusual or worth flagging]
```

### Spec Review Request (mailbox message)

```
SPEC REVIEW REQUEST

You implemented Task {N}. Now review your own implementation
against the original spec with fresh eyes.

## Original Requirements
{FULL TEXT of task requirements}

## What You Reported Building
{paste member's DONE report}

## Review Instructions
Read the actual code you wrote. Verify line by line:

1. Missing requirements — did you implement everything asked?
2. Extra work — did you build anything not requested?
3. Misunderstandings — did you solve the right problem?

DO NOT trust your own DONE report. Read the code.

Report EXACTLY one of:
✅ SPEC_COMPLIANT: [brief confirmation]
❌ SPEC_ISSUES: [list each issue with file:line]
```

### Quality Review Request (mailbox message)

```
QUALITY REVIEW REQUEST

Spec compliance passed. Now review your implementation for quality.

## What Was Implemented
{paste member's DONE report}

## Review Checklist
- Each file has one clear responsibility
- Units are independently testable
- Error handling: proper, not swallowed
- Tests: meaningful assertions, not just happy path
- Naming: consistent with codebase
- No dead code, no commented-out code
- No AI-generated comment slop

Report EXACTLY one of:
✅ QUALITY_PASS: [brief strengths]
❌ QUALITY_ISSUES:
  Critical: [must fix]
  Important: [should fix]
  Minor: [note for later]
```

---

## Team Naming Convention

Use short, descriptive names:
- `impl-batch-1`, `impl-batch-2` — implementation batches
- `impl-seq-{task-id}` — single sequential task

Avoid spaces and special characters. Names must be unique per run.

## Colors for Members

Assign different colors so tmux grid is readable:
`blue`, `green`, `yellow`, `cyan`, `magenta`, `red`, `white`

## Limits

Default config:
- `max_parallel_members: 4` — max 4 members running at once
- `max_members: 8` — max 8 total per team
- `max_member_turns: 100` — each member gets 100 turns max

If a batch has more than 4 tasks, split into sub-batches.

## Critical Rules

1. **Members start on creation** — their `prompt` is their first task
2. **Mailbox is async** — send messages, wait via `team_status`, read results
3. **One mailbox poll at a time** — don't flood with messages before checking status
4. **Delete teams when done** — don't leave orphaned teams running
5. **Mark todos complete** — only after both spec AND quality pass
6. **Fresh context** — each member gets only what's in their prompt + mailbox, never your session history
