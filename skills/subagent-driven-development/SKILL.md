---
name: subagent-driven-development
description: Use when executing implementation plans with independent tasks in the current session
---

# Subagent-Driven Development

Execute plan by dispatching fresh subagent per task, with two-stage review after each: spec compliance review first, then code quality review.

**Why subagents:** You delegate tasks to specialized agents with isolated context. By precisely crafting their instructions and context, you ensure they stay focused and succeed at their task. They should never inherit your session's context or history — you construct exactly what they need. This also preserves your own context for coordination work.

**Core principle:** Fresh subagent per task + two-stage review (spec then quality) = high quality, fast iteration

## When to Use

**Use when:**
- Have implementation plan with defined tasks
- Tasks are mostly independent
- Want to stay in this session (not parallel session)

**vs. Executing Plans (parallel session):**
- Same session (no context switch)
- Fresh subagent per task (no context pollution)
- Two-stage review after each task
- Faster iteration (no human-in-loop between tasks)

**vs. dispatching-parallel-agents:**
- Use dispatching-parallel-agents when tasks are fully independent AND don't need review
- Use subagent-driven-development when tasks need sequential review cycles

## The Process

### Step 0: Read Plan and Create Todos

1. Read plan file
2. Extract all tasks with full text
3. Create TodoWrite with one item per task
4. Note architectural context and dependencies between tasks

### Step 1: Identify Parallelizable Tasks

Before starting, scan the task list for **independent tasks** — tasks that:
- Touch different files
- Have no data dependencies
- Can be reviewed independently

Group them into **parallel batches** and **sequential chains**.

### Step 2: Execute Tasks

#### For Independent Tasks (Parallel Batch)

Dispatch multiple implementers simultaneously:

```typescript
// Fire 2-3 independent implementers as background tasks
task(category="unspecified-high", load_skills=["tdd"],
  run_in_background=true,
  description="Implement Task 1: [name]",
  prompt="[Use implementer-prompt.md template - see below]")

task(category="unspecified-high", load_skills=["tdd"],
  run_in_background=true,
  description="Implement Task 2: [name]",
  prompt="[Use implementer-prompt.md template - see below]")

// End response. Wait for <system-reminder> notifications.
```

After all complete, collect results and run reviews sequentially.

#### For Dependent Tasks (Sequential Chain)

Dispatch one at a time, review, then proceed:

```typescript
// Implementer (synchronous - need result before review)
task(category="unspecified-high", load_skills=["tdd"],
  run_in_background=false,
  description="Implement Task N: [name]",
  prompt="[Use implementer-prompt.md template]")

// If implementer asks questions: answer them, then resume with task_id
task(task_id="ses_xxx", load_skills=[],
  run_in_background=false,
  prompt="[answers to questions]")
```

### Step 3: Two-Stage Review (Per Task)

After each implementer completes:

**Stage 1: Spec Compliance Review**

```typescript
task(category="unspecified-high", load_skills=[],
  run_in_background=false,
  description="Review spec compliance for Task N",
  prompt="[Use spec-reviewer-prompt.md template - see below]")
```

- If issues found → resume implementer with `task_id` to fix
- If passes → proceed to Stage 2

**Stage 2: Code Quality Review**

```typescript
task(category="unspecified-high", load_skills=["requesting-code-review"],
  run_in_background=false,
  description="Code quality review for Task N",
  prompt="[Use code-quality-reviewer-prompt.md template - see below]")
```

- If Critical/Important issues → resume implementer with `task_id` to fix
- If passes → mark task complete in TodoWrite

### Step 4: Final Review

After all tasks complete:
- Dispatch final code reviewer for entire implementation
- Use superpowers:finishing-a-development-branch

## Prompt Templates

### Implementer Prompt (./implementer-prompt.md)

```
task prompt: |
  You are implementing Task N: [task name]

  ## Task Description

  [FULL TEXT of task from plan - paste it here, don't make subagent read file]

  ## Context

  [Scene-setting: where this fits, dependencies, architectural context]

  ## Before You Begin

  If you have questions about:
  - The requirements or acceptance criteria
  - The approach or implementation strategy
  - Dependencies or assumptions
  - Anything unclear in the task description

  **Ask them now.** Raise any concerns before starting work.

  ## Your Job

  Once you're clear on requirements:
  1. Implement exactly what the task specifies
  2. Write tests (following TDD if task says to)
  3. Verify implementation works
  4. Commit your work
  5. Self-review: check for missing requirements, dead code, unclear naming
  6. Report back with: what you built, files changed, any concerns

  Work from: [directory]

  ## Code Organization

  - Follow the file structure defined in the plan
  - Each file should have one clear responsibility
  - If a file is growing beyond the plan's intent, stop and report as DONE_WITH_CONCERNS
  - In existing codebases, follow established patterns

  ## When You're in Over Your Head

  STOP and escalate when:
  - The task requires architectural decisions with multiple valid approaches
  - You need to understand code beyond what was provided
  - You feel uncertain about whether your approach is correct
  - You've been going back and forth without making progress

  ## Report Format

  DONE: [brief summary]
  FILES_CHANGED: [list]
  TESTS: [pass/fail/skipped]
  CONCERNS: [any issues or things to watch]
```

### Spec Reviewer Prompt (./spec-reviewer-prompt.md)

```
task prompt: |
  You are reviewing whether an implementation matches its specification.

  ## What Was Requested

  [FULL TEXT of task requirements]

  ## What Implementer Claims They Built

  [From implementer's report]

  ## CRITICAL: Do Not Trust the Report

  The implementer's report may be incomplete or optimistic. You MUST verify independently.

  **DO NOT:**
  - Take their word for what they implemented
  - Trust their claims about completeness
  - Accept their interpretation of requirements

  **DO:**
  - Read the actual code they wrote
  - Compare actual implementation to requirements line by line
  - Check for missing pieces
  - Look for extra features they didn't mention

  ## Your Job

  Read the implementation code and verify:

  **Missing requirements:**
  - Did they implement everything requested?
  - Are there requirements they skipped?

  **Extra/unneeded work:**
  - Did they build things that weren't requested?
  - Did they over-engineer?

  **Misunderstandings:**
  - Did they interpret requirements differently than intended?
  - Did they solve the wrong problem?

  Report:
  - ✅ Spec compliant (if everything matches after code inspection)
  - ❌ Issues found: [list specifically what's missing or extra, with file:line references]
```

### Code Quality Reviewer Prompt (./code-quality-reviewer-prompt.md)

```
task prompt: |
  Review code quality for Task N implementation.

  **Only run this after spec compliance review passes.**

  ## What Was Implemented
  [from implementer's report]

  ## Review Criteria

  Check:
  - Does each file have one clear responsibility with a well-defined interface?
  - Are units decomposed so they can be understood and tested independently?
  - Is the implementation following the file structure from the plan?
  - Are there any large files that should be split?
  - Error handling: proper, not swallowed
  - Tests: meaningful assertions, not just happy path
  - Naming: clear, consistent with codebase conventions
  - No dead code, no commented-out code

  ## Report Format

  **Strengths:** [what's good]
  **Issues:**
  - Critical: [must fix before proceeding]
  - Important: [should fix before proceeding]
  - Minor: [note for later]
  **Assessment:** Ready to proceed / Needs fixes
```

## Parallelization Decision Matrix

| Scenario | Strategy |
|----------|----------|
| Tasks touch different files, no deps | Parallel batch (all `run_in_background=true`) |
| Task B depends on Task A's output | Sequential (A sync → review → B sync) |
| Tasks touch same files | Sequential (avoid merge conflicts) |
| 5+ independent tasks | Batch in groups of 3 (avoid overwhelming) |
| Review after parallel batch | Sequential reviews (one at a time) |

## Critical Rules

1. **Fresh subagent per task** — never reuse (except with `task_id` for fixes)
2. **Full context in prompt** — paste task text, don't make subagent read files
3. **Two-stage review always** — spec first, quality second
4. **Use `task_id` for fixes** — resume same agent, don't start fresh
5. **Mark TodoWrite after BOTH reviews pass** — not after implementation alone
6. **End response after background dispatch** — wait for notifications
