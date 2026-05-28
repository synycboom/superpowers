---
name: dispatching-parallel-agents
description: Use when facing 2+ independent tasks that can be worked on without shared state or sequential dependencies
---

# Dispatching Parallel Agents

## Overview

You delegate tasks to specialized agents with isolated context. By precisely crafting their instructions and context, you ensure they stay focused and succeed at their task. They should never inherit your session's context or history — you construct exactly what they need. This also preserves your own context for coordination work.

When you have multiple unrelated failures (different test files, different subsystems, different bugs), investigating them sequentially wastes time. Each investigation is independent and can happen in parallel.

**Core principle:** Dispatch one agent per independent problem domain. Let them work concurrently via background tasks.

## When to Use

**Use when:**
- 3+ test files failing with different root causes
- Multiple subsystems broken independently
- Each problem can be understood without context from others
- No shared state between investigations

**Don't use when:**
- Failures are related (fix one might fix others)
- Need to understand full system state
- Agents would interfere with each other (editing same files)

## The Pattern

### 1. Identify Independent Domains

Group failures by what's broken:
- File A tests: Tool approval flow
- File B tests: Batch completion behavior
- File C tests: Abort functionality

Each domain is independent — fixing tool approval doesn't affect abort tests.

### 2. Create Focused Agent Tasks

Each agent gets:
- **Specific scope:** One test file or subsystem
- **Clear goal:** Make these tests pass
- **Constraints:** Don't change other code
- **Expected output:** Summary of what you found and fixed

### 3. Dispatch in Parallel (Background Tasks)

```typescript
// Fire all agents concurrently as background tasks
task(category="unspecified-high", load_skills=["systematic-debugging"],
  run_in_background=true,
  description="Fix abort test failures",
  prompt="Fix the 3 failing tests in src/agents/agent-tool-abort.test.ts:\n\n1. 'should abort tool with partial output capture'\n2. 'should handle mixed completed and aborted tools'\n3. 'should properly track pendingToolCount'\n\nThese are timing/race condition issues. Read the test file, identify root cause, fix by replacing arbitrary timeouts with event-based waiting. Do NOT just increase timeouts.\n\nReturn: Summary of root cause and what you fixed.")

task(category="unspecified-high", load_skills=["systematic-debugging"],
  run_in_background=true,
  description="Fix batch completion failures",
  prompt="Fix failing tests in src/agents/batch-completion-behavior.test.ts...\n\n[full context here]\n\nReturn: Summary of root cause and what you fixed.")

task(category="unspecified-high", load_skills=["systematic-debugging"],
  run_in_background=true,
  description="Fix race condition failures",
  prompt="Fix failing tests in src/agents/tool-approval-race-conditions.test.ts...\n\n[full context here]\n\nReturn: Summary of root cause and what you fixed.")

// STOP HERE. End your response.
// Wait for <system-reminder> notifications for each task.
// Do NOT poll background_output — the system will notify you.
```

### 4. Collect Results

When `<system-reminder>` arrives for each task:

```typescript
background_output(task_id="bg_abc123")
background_output(task_id="bg_def456")
background_output(task_id="bg_ghi789")
```

### 5. Review and Integrate

When all agents return:
- Read each summary
- Verify fixes don't conflict (check for overlapping file edits)
- Run full test suite
- Integrate all changes

## Agent Prompt Structure

Good agent prompts are:
1. **Focused** — One clear problem domain
2. **Self-contained** — All context needed to understand the problem
3. **Specific about output** — What should the agent return?

```markdown
Fix the 3 failing tests in src/agents/agent-tool-abort.test.ts:

1. "should abort tool with partial output capture" - expects 'interrupted at' in message
2. "should handle mixed completed and aborted tools" - fast tool aborted instead of completed
3. "should properly track pendingToolCount" - expects 3 results but gets 0

These are timing/race condition issues. Your task:

1. Read the test file and understand what each test verifies
2. Identify root cause - timing issues or actual bugs?
3. Fix by:
   - Replacing arbitrary timeouts with event-based waiting
   - Fixing bugs in abort implementation if found
   - Adjusting test expectations if testing changed behavior

Do NOT just increase timeouts - find the real issue.

Return: Summary of what you found and what you fixed.
```

## Critical Rules

1. **ALWAYS use `run_in_background=true`** — parallel agents must be async
2. **End your response after dispatching** — do NOT continue with dependent work
3. **Wait for `<system-reminder>`** — never poll `background_output` immediately
4. **Collect ALL results before integrating** — don't act on partial results
5. **Verify no conflicts** — agents working on separate files should not overlap

## Common Mistakes

**❌ Too broad:** "Fix all the tests" — agent gets lost
**✅ Specific:** "Fix agent-tool-abort.test.ts" — focused scope

**❌ No context:** "Fix the race condition" — agent doesn't know where
**✅ Context:** Paste the error messages and test names

**❌ No constraints:** Agent might refactor everything
**✅ Constraints:** "Do NOT change production code" or "Fix tests only"

**❌ Synchronous dispatch:** `task(run_in_background=false)` for independent work
**✅ Background dispatch:** `task(run_in_background=true)` — parallel execution

**❌ Polling immediately:** Calling `background_output` right after dispatch
**✅ Wait for notification:** End response, let `<system-reminder>` arrive

## When NOT to Use

**Related failures:** Fixing one might fix others — investigate together first
**Need full context:** Understanding requires seeing entire system
**Exploratory debugging:** You don't know what's broken yet
**Shared state:** Agents would interfere (editing same files, using same resources)
