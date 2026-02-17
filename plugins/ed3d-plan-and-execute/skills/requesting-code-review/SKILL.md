---
name: requesting-code-review
description: Use when completing tasks, implementing major features, or before merging to verify work meets requirements - dispatches code-reviewer subagent, handles retries and timeouts, manages review-fix loop until zero issues
user-invocable: false
---

# Requesting Code Review

Dispatch ed3d-plan-and-execute:code-reviewer subagent to catch issues before they cascade.

**Core principle:** Review early, review often. Fix ALL issues before proceeding.

## When to Request Review

**Mandatory:**
- After each phase in implementation-plan execution
- After completing major feature
- Before merge to main

**Optional but valuable:**
- When stuck (fresh perspective)
- Before refactoring (baseline check)
- After fixing complex bug

## The Review Loop

The review process is a loop: review → fix → re-review → until zero issues.

```
┌──────────────────────────────────────────────────┐
│                                                  │
│   Dispatch code-reviewer                         │
│         │                                        │
│         ▼                                        │
│   Issues found? ──No──► Done (proceed)           │
│         │                                        │
│        Yes                                       │
│         │                                        │
│         ▼                                        │
│   Dispatch bug-fixer                             │
│         │                                        │
│         ▼                                        │
│   Re-review with prior issues ◄──────────────────┘
│
└──────────────────────────────────────────────────┘
```

**Exit condition:** Zero issues, or issues accepted per your workflow's policy.

## File Path Generation

**Generate review file paths with cycle numbers:**

For each review cycle, generate a unique file path:

```
REVIEW_OUTPUT_FILE: /tmp/execution-reports/[plan-dir]/phase_XX_review_cycle_N.md
FIX_REPORT_FILE: /tmp/execution-reports/[plan-dir]/phase_XX_fix_cycle_N_report.md
```

Where:
- `[plan-dir]` is the name of the implementation plan directory
- `phase_XX` matches the phase number (e.g., `phase_01`)
- `cycle_N` is the review cycle number (1, 2, 3, ...)

**Create the directory before dispatch:**
```bash
mkdir -p /tmp/execution-reports/[plan-dir]
```

## Step 1: Initial Review

**Get git SHAs:**
```bash
BASE_SHA=$(git rev-parse HEAD~1)  # or commit before task
HEAD_SHA=$(git rev-parse HEAD)
```

**Create output directory:**
```bash
mkdir -p /tmp/execution-reports/[plan-dir]
```

**Dispatch code-reviewer subagent:**

```
<invoke name="Task">
<parameter name="subagent_type">ed3d-plan-and-execute:code-reviewer</parameter>
<parameter name="description">Reviewing [what was implemented]</parameter>
<parameter name="prompt">
  Review the implementation for quality and plan alignment.

  WHAT_WAS_IMPLEMENTED: [summary of implementation]
  PLAN_OR_REQUIREMENTS: [task/requirements reference]
  BASE_SHA: [commit before work]
  HEAD_SHA: [current commit]
  REVIEW_OUTPUT_FILE: /tmp/execution-reports/[plan-dir]/phase_XX_review_cycle_1.md
  IMPLEMENTATION_GUIDANCE: [path or "None"]

  Write your full review to REVIEW_OUTPUT_FILE and create TaskCreate for each issue found.
  Return a compact summary only.
</parameter>
</invoke>
```

**Code reviewer returns:**
```
Status: APPROVED / CHANGES REQUIRED
Issues: Critical: N | Important: N | Minor: N
Full review: [REVIEW_OUTPUT_FILE path]
Tasks created: N
```

## Step 2: Handle Reviewer Response

### If Zero Issues
All categories empty → proceed to next task.

### If Any Issues Found
Regardless of category (Critical, Important, or Minor), dispatch bug-fixer:

```
<invoke name="Task">
<parameter name="subagent_type">ed3d-plan-and-execute:task-bug-fixer</parameter>
<parameter name="description">Fixing review issues (cycle N)</parameter>
<parameter name="prompt">
  Fix issues from code review.

  REVIEW_OUTPUT_FILE: /tmp/execution-reports/[plan-dir]/phase_XX_review_cycle_N.md
  WORKING_DIRECTORY: [directory]
  FIX_REPORT_FILE: /tmp/execution-reports/[plan-dir]/phase_XX_fix_cycle_N_report.md

  Read the review file, fix all issues (Critical → Important → Minor), and write your report to FIX_REPORT_FILE.
  Mark TaskCreate fix tasks complete as you resolve each issue.

  Return a compact summary only.
</parameter>
</invoke>
```

**Bug-fixer returns:**
```
Fixed: N issues
Remaining: M issues (task IDs: [...])
Commit: [SHA]
Report: [FIX_REPORT_FILE path]
```

After fixes, proceed to Step 3.

## Step 3: Re-Review After Fixes

**CRITICAL:** Track prior issues across review cycles. Generate a new review file path for the next cycle.

**Use explicit numeric cycle incrementing (never literal `N+1` in filenames):**
```bash
CURRENT_CYCLE=[current cycle number]
NEXT_CYCLE=$((CURRENT_CYCLE + 1))
REVIEW_OUTPUT_FILE="/tmp/execution-reports/[plan-dir]/phase_XX_review_cycle_${NEXT_CYCLE}.md"
```

```
<invoke name="Task">
<parameter name="subagent_type">ed3d-plan-and-execute:code-reviewer</parameter>
<parameter name="description">Re-reviewing after fixes (cycle [NEXT_CYCLE])</parameter>
<parameter name="prompt">
  Re-review after bug fixes.

  WHAT_WAS_IMPLEMENTED: [from bug-fixer's summary]
  PLAN_OR_REQUIREMENTS: [original task/requirements]
  BASE_SHA: [commit before this fix cycle]
  HEAD_SHA: [current commit after fixes]
  REVIEW_OUTPUT_FILE: /tmp/execution-reports/[plan-dir]/phase_XX_review_cycle_[NEXT_CYCLE].md
  IMPLEMENTATION_GUIDANCE: [path or "None"]
  PRIOR_ISSUES_TO_VERIFY_FIXED: [list issues from previous review file]

  Read the previous review at [previous REVIEW_OUTPUT_FILE path] to see what issues were supposed to be fixed.
  Verify each prior issue is resolved.
  Check for any new issues introduced by the fixes.

  Write your full review to REVIEW_OUTPUT_FILE and create TaskCreate for any NEW or PERSISTING issues.
  Return a compact summary only.
</parameter>
</invoke>
```

**Tracking prior issues:**
- When re-reviewer explicitly confirms fixed → remove from list
- When re-reviewer doesn't mention an issue → keep on list (silence ≠ fixed)
- When re-reviewer finds new issues → add to list
- Re-reviewer creates NEW TaskCreate only for issues that persist after bug-fixer

Loop back to Step 2 if any issues remain.

## Review File Tracking

Keep track of review file paths across cycles:

| Cycle | Review File | Fix Report |
|-------|-------------|------------|
| 1 | `phase_01_review_cycle_1.md` | `phase_01_fix_cycle_1_report.md` |
| 2 | `phase_01_review_cycle_2.md` | `phase_01_fix_cycle_2_report.md` |
| ... | ... | ... |

Pass the previous review file path to the re-reviewer so they can verify fixes.

## Handling Failures

### Operational Errors
If reviewer reports operational errors (can't run tests, missing scripts):
1. **STOP** - do not continue
2. Report to human
3. When told to continue, re-execute same review

### Timeouts / Empty Response
Usually means context limits. Retry with focused scope:

**First retry:** Narrow to changed files only:
```
FOCUSED REVIEW - Context was too large.

REVIEW_OUTPUT_FILE: /tmp/execution-reports/[plan-dir]/phase_XX_review_cycle_N_focused.md

Review ONLY the diff between BASE_SHA and HEAD_SHA.
Focus on: [list only files actually modified]

Skip: broad architectural analysis, unchanged files, tangential concerns.

WHAT_WAS_IMPLEMENTED: [summary]
PLAN_OR_REQUIREMENTS: [reference]
BASE_SHA: [sha]
HEAD_SHA: [sha]
```

**Second retry:** Split into multiple smaller reviews (one per file or logical group).

**Third failure:** Stop and ask human for help.

## Quick Reference

| Situation | Action |
|-----------|--------|
| Zero issues | Proceed |
| Any issues | Fix, re-review (or accept per workflow) |
| Operational error | Stop, report, wait |
| Timeout | Retry with focused scope |
| 3 failed retries | Ask human |

## Red Flags

**Never:**
- Skip review because "it's simple"
- Proceed with ANY unfixed issues (Critical, Important, OR Minor)
- Argue with valid technical feedback without evidence
- Rationalize skipping Minor issues ("they're just style", "we can fix later")

**Minor issues are NOT optional.** The code reviewer flagged them for a reason. Fix all of them. "Minor" means lower severity, not "ignorable."

**If reviewer wrong:**
- Push back with technical reasoning
- Show code/tests that prove it works
- Request clarification on unclear feedback

## Integration

**Called by:**
- executing-an-implementation-plan (after each task)
- finishing-a-development-branch (final review)
- Ad-hoc when you need a review
