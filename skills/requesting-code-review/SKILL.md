---
name: requesting-code-review
description: Use when completing tasks, implementing major features, or before merging to verify work meets requirements
---

# Requesting Code Review

Dispatch a code-reviewer agent to catch issues before they cascade. The reviewer gets precisely crafted context for evaluation — never your session's history. This keeps the reviewer focused on the work product, not your thought process, and preserves your own context for continued work.

**Core principle:** Review early, review often. Reviews run in background — don't block your next task.

## When to Request Review

**Mandatory:**
- After each task in subagent-driven development
- After completing major feature
- Before merge to main

**Optional but valuable:**
- When stuck (fresh perspective)
- Before refactoring (baseline check)
- After fixing complex bug

## How to Request

### 1. Get git SHAs

```bash
BASE_SHA=$(git rev-parse HEAD~1)  # or origin/main
HEAD_SHA=$(git rev-parse HEAD)
```

### 2. Dispatch Code Reviewer (Background)

If you have more work to do after the review, dispatch as **background**:

```typescript
task(category="unspecified-high", load_skills=[],
  run_in_background=true,
  description="Code review: [feature name]",
  prompt=`You are reviewing code changes for quality, correctness, and maintainability.

## What Was Implemented
[What you just built — brief description]

## Plan/Requirements
[What it should do — paste relevant plan section or requirements]

## Commits to Review
BASE_SHA: [base commit]
HEAD_SHA: [head commit]

Run: git diff ${BASE_SHA}..${HEAD_SHA}

## Review Checklist

**Correctness:**
- Does the code do what the requirements specify?
- Are there edge cases not handled?
- Are error paths handled properly?

**Quality:**
- Is the code clear and maintainable?
- Are there unnecessary abstractions?
- Is naming consistent with the codebase?

**Tests:**
- Are tests meaningful (not just happy path)?
- Do tests cover the behavior, not the implementation?
- Are there missing test cases?

**Style:**
- Does it match existing codebase patterns?
- Any dead code or commented-out code?
- Any TODO/FIXME without tracking?

## Report Format

**Strengths:** [what's good about the implementation]
**Issues:**
- Critical: [must fix — bugs, security, data loss risks]
- Important: [should fix — maintainability, missing tests, unclear logic]
- Minor: [note for later — style, naming nitpicks]
**Assessment:** Ready to merge / Needs fixes / Needs discussion`)
```

Continue working on your next task. Collect review when notified.

### 3. Dispatch Code Reviewer (Blocking)

If this is your last task and you need the result before reporting done:

```typescript
task(category="unspecified-high", load_skills=[],
  run_in_background=false,
  description="Code review: [feature name]",
  prompt="[same prompt as above]")
```

### 4. Act on Feedback

When review results arrive:
- **Critical issues** → fix immediately
- **Important issues** → fix before proceeding
- **Minor issues** → note for later (or fix if quick)
- **Disagree** → push back with reasoning (use `receiving-code-review` skill)

## When to Use Background vs Blocking

| Situation | Mode | Why |
|-----------|------|-----|
| Between tasks in a plan | `run_in_background=true` | Start next task while review runs |
| Last task before reporting done | `run_in_background=false` | Need result before claiming complete |
| Before merge/PR | `run_in_background=false` | Must have clean review first |
| After each subtask in parallel batch | `run_in_background=true` | Don't block other subtasks |

## Integration with Workflows

**Subagent-Driven Development:**
- Review after EACH task (spec review + quality review)
- Catch issues before they compound
- Fix before moving to next task

**Dispatching Parallel Agents:**
- Fire multiple reviews in parallel for independent tasks
- Collect all results before final integration

**Executing Plans:**
- Review after completing each batch of tasks
- Use as checkpoint before proceeding

## Session Continuity

If the reviewer finds issues:
1. Use `task_id` to resume the **original implementer** for fixes
2. Don't start a fresh agent — the implementer already has context
3. After fix, request another review (can be lighter — just verify the fix)

```typescript
// Reviewer found issues
task(task_id="ses_implementer_xxx", load_skills=[],
  run_in_background=false,
  prompt="Reviewer found these issues:\n\n[paste issues]\n\nFix them. Don't change anything else.")

// Then re-review (lighter)
task(category="unspecified-high", load_skills=[],
  run_in_background=false,
  description="Re-review fix for Task N",
  prompt="Verify that the following issues were fixed:\n[issues]\n\nCheck git diff HEAD~1..HEAD. Confirm fixes are correct and complete.")
```
