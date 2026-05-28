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

### 2. Dispatch Code Reviewer (always background)

Always dispatch as a background task — never block the main session:

```
task(subagent_type="general", load_skills=[],
  run_in_background=true,
  description="Code review: [feature name]",
  prompt="You are reviewing code changes for quality, correctness, and maintainability.

## What Was Implemented
[What you just built — brief description]

## Plan/Requirements
[What it should do — paste relevant plan section or requirements]

## Commits to Review
BASE_SHA: [base commit]
HEAD_SHA: [head commit]

Run: git diff BASE_SHA..HEAD_SHA

## Review Checklist

Correctness:
- Does the code do what the requirements specify?
- Are there edge cases not handled?
- Are error paths handled properly?

Quality:
- Is the code clear and maintainable?
- Are there unnecessary abstractions?
- Is naming consistent with the codebase?

Tests:
- Are tests meaningful (not just happy path)?
- Do tests cover the behavior, not the implementation?
- Are there missing test cases?

Style:
- Does it match existing codebase patterns?
- Any dead code or commented-out code?
- Any TODO/FIXME without tracking?

## Report Format
Strengths: [what's good]
Issues:
- Critical: [must fix — bugs, security, data loss]
- Important: [should fix — maintainability, missing tests]
- Minor: [note for later — style, naming]
Assessment: Ready to merge / Needs fixes / Needs discussion")

← STOP. End response. Wait for <system-reminder>.
Then: background_output(task_id="...")
```

### 3. Act on Feedback

When notified → background_output() → read result:
- **Critical issues** → fix immediately
- **Important issues** → fix before proceeding
- **Minor issues** → note for later
- **Disagree** → push back with reasoning (use `receiving-code-review` skill)

## Session Continuity

If reviewer finds issues, resume the original implementer as a background task:

```
task(task_id="ses_implementer_xxx", load_skills=[],
  run_in_background=true,
  prompt="Reviewer found these issues:

[paste issues]

Fix them. Don't change anything else.")

← STOP. Wait for <system-reminder>.
```

Then re-review (lighter):

```
task(subagent_type="general", load_skills=[],
  run_in_background=true,
  description="Re-review fix for Task N",
  prompt="Verify these issues were fixed: [issues]
Check git diff HEAD~1..HEAD. Confirm fixes are correct and complete.")

← STOP. Wait for <system-reminder>.
```
