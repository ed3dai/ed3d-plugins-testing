---
name: executing-an-implementation-plan
description: Use when executing implementation plans with independent tasks in the current session - dispatches fresh subagent for each task, reviews once per phase, loads phases just-in-time to minimize context usage
user-invocable: false
---

# Executing an Implementation Plan

Execute plan phase-by-phase, loading each phase just-in-time to minimize context usage.

**Core principle:** Delegate to phase-prep subagent → Execute tasks from files → Review from files. Never load full content into orchestrator context.

**REQUIRED SKILL:** `requesting-code-review` - The review loop (dispatch, fix, re-review until zero issues)

## Overview

**When NOT to use:**
- No implementation plan exists yet (use writing-implementation-plans first)
- Plan needs revision (brainstorm first)

## MANDATORY: Human Transparency

**The human cannot see what subagents return. You are their window into the work.**

After EVERY subagent completes (task-implementor, bug-fixer, code-reviewer, phase-prep), you MUST:

1. **Print the subagent's compact response** to the user before taking any other action
2. **Include key details:** test counts, issue counts, commit hashes, file paths to full reports
3. **Do not summarize further** - the subagent already returned a compact summary

**Before dispatching any subagent:**
- Briefly explain (2-3 sentences) what you're asking the agent to do
- State which phase this covers

**Why this matters:** When you silently process subagent output without showing the user, they lose visibility into their own codebase. They can't catch errors, learn from the process, or intervene when needed. Transparency is not optional.

**Red flag:** If you find yourself thinking "I'll just move on to the next step" without printing the subagent's response, STOP. Print it first.

## REQUIRED: Implementation Plan Path

**DO NOT GUESS.** If the user has not provided a path to an implementation plan directory, you MUST ask for it.

Use AskUserQuestion:
```
Question: "Which implementation plan should I execute?"
Options:
  - [list any plan directories you find in docs/implementation-plans/]
  - "Let me provide the path"
```

If `docs/implementation-plans/` doesn't exist or is empty, ask the user to provide the path directly.

**Never assume, infer, or guess which plan to execute.** The user must explicitly tell you.

## The Process

### 1. Discover Phases

**DO NOT read the full phase files yet.** List them and read only the header and task markers.

```bash
# List phase files
ls [plan-directory]/phase_*.md

# For each file, get the header (first 10 lines include title and Goal)
head -10 [plan-directory]/phase_01.md

# Get task/subcomponent structure without reading full content
grep -E "START_TASK_|START_SUBCOMPONENT_" [plan-directory]/phase_01.md
```

The header includes the title (`# [Phase Title]`) and `**Goal:**` line. Extract the title for the task entry.

The grep output shows the task structure, e.g.:
```
<!-- START_TASK_1 -->
<!-- START_TASK_2 -->
<!-- START_SUBCOMPONENT_A (tasks 3-5) -->
<!-- START_TASK_3 -->
<!-- START_TASK_4 -->
<!-- START_TASK_5 -->
```

Examples of headers you might see:
- `# Document Infrastructure Implementation Plan` — Phase 1 implied
- `# Phase 4: Link Resolution` — Phase number explicit

**Check for implementation guidance:**

After discovering phases, check if `.ed3d/implementation-plan-guidance.md` exists in the project root:

```bash
# Check for implementation guidance (note the absolute path for later use)
ls [project-root]/.ed3d/implementation-plan-guidance.md
```

If the file exists, note its **absolute path** for use during code reviews. If it doesn't exist, proceed without it—do not pass a nonexistent path to reviewers.

**Check for test requirements:**

Check if `test-requirements.md` exists in the plan directory:

```bash
# Check for test requirements (note the absolute path for later use)
ls [plan-directory]/test-requirements.md
```

If the file exists, note its **absolute path** for use during final review. The test requirements document specifies what automated tests must exist for each acceptance criterion.

### 2. Create Phase-Level Task List

Use TaskCreate to create **three task entries per phase** (or TodoWrite in older Claude Code versions). Include the title from the header:

```
- [ ] Phase 1a: Prepare phase 01 — Document Infrastructure Implementation Plan
- [ ] Phase 1b: Execute tasks
- [ ] Phase 1c: Code review
- [ ] Phase 2a: Prepare phase 02 — API Integration
- [ ] Phase 2b: Execute tasks
- [ ] Phase 2c: Code review
...
```

**Why include the title:** Gives visibility into what each phase covers without loading full content.

### 3. Execute Each Phase

For each phase, follow this cycle:

#### 3a. Prepare Phase (DELEGATE to phase-prep subagent)

Mark "Phase Na: Prepare phase NN" as in_progress.

**Create output directories:**
```bash
mkdir -p /tmp/execution-prep/[plan-dir-name]
mkdir -p /tmp/execution-reports/[plan-dir-name]
```

**Dispatch execute-phase-prep subagent:**

```
<invoke name="Task">
<parameter name="subagent_type">ed3d-plan-and-execute:execute-phase-prep</parameter>
<parameter name="description">Preparing Phase N: [phase name]</parameter>
<parameter name="prompt">
  Extract tasks from the phase file for execution.

  PHASE_FILE: [absolute path to phase file, e.g., /path/to/plan/phase_01.md]
  OUTPUT_DIR: /tmp/execution-prep
  WORKING_DIRECTORY: [working directory for this phase]

  Read the phase file, extract each task to its own file, write a summary file, and create TaskCreate for each task.

  Return a compact summary with:
  - Number of tasks extracted
  - Path to summary file
  - List of TaskCreate IDs created
</parameter>
</invoke>
```

**Phase-prep returns:**
```
Phase N prep complete.
Tasks: M
Task files: /tmp/execution-prep/[plan-dir]/phase_XX/phase_XX_task_*.md
Summary: /tmp/execution-prep/[plan-dir]/phase_XX/phase_XX_summary.md
TaskCreate IDs: [list of task IDs]
```

**Print the response.** Mark "Phase Na: Prepare" as complete.

#### 3b. Execute All Tasks

Mark "Phase Nb: Execute tasks" as in_progress.

**Before dispatching, verify test coverage for functionality tasks:**

Read the phase summary file to check if tasks have tests. If a functionality task has no tests specified:
1. Check if a subsequent task in the same phase provides tests
2. If no tests exist anywhere for this functionality → **STOP**
3. This is a plan gap. Surface to user: "Task N implements [functionality] but no corresponding tests exist in the plan. This needs tests before implementation."

Do NOT implement functionality without tests. Missing tests = plan gap, not something to skip.

**Execute all tasks in sequence.** For each task, dispatch `task-implementor-fast`:

```
<invoke name="Task">
<parameter name="subagent_type">ed3d-plan-and-execute:task-implementor-fast</parameter>
<parameter name="description">Implementing Phase N, Task M: [description]</parameter>
<parameter name="prompt">
  Implement the task from the task specification file.

  TASK_SPEC_FILE: /tmp/execution-prep/[plan-dir]/phase_XX/phase_XX_task_MM.md
  PHASE_SUMMARY_FILE: /tmp/execution-prep/[plan-dir]/phase_XX/phase_XX_summary.md
  WORKING_DIRECTORY: [directory]
  REPORT_OUTPUT_FILE: /tmp/execution-reports/[plan-dir]/phase_XX_task_MM_report.md

  Read the task spec file, implement the task, verify with tests/build/lint, commit your work, and write your report to the report file.

  Return a compact summary only.
</parameter>
</invoke>
```

**Task-implementor returns:**
```
Status: COMPLETED / FAILED
Files changed: [list]
Tests: X/Y passing
Commit: [SHA]
Report: [REPORT_OUTPUT_FILE path]
```

**Print each task-implementor's response** before moving to the next task.

**No code review between tasks.** Execute all tasks in the phase first.

After all tasks complete, mark "Phase Nb: Execute tasks" as complete.

#### 3c. Code Review for Phase

Mark "Phase Nc: Code review" as in_progress.

**MANDATORY:** Use the `requesting-code-review` skill for the review loop.

**Context to provide:**
- WHAT_WAS_IMPLEMENTED: Summary of all tasks in this phase
- PLAN_OR_REQUIREMENTS: Path to phase summary file
- BASE_SHA: commit before phase started
- HEAD_SHA: current commit
- IMPLEMENTATION_GUIDANCE: absolute path to `.ed3d/implementation-plan-guidance.md` (**only if it exists**—omit entirely if the file doesn't exist)
- PLAN_DIR_NAME: Name of the implementation plan directory (for file paths)
- PHASE_NUMBER: Phase number (e.g., "01")

The implementation guidance file contains project-specific coding standards, testing requirements, and review criteria. When provided, the code reviewer should read it and apply those standards during review.

**Note:** Test requirements validation happens at final review, not per-phase. Per-phase reviews focus on code quality and whether the phase includes tests for its functionality.

**If code reviewer returns a context limit error:**

The phase changed too much for a single review. Chunk the review:

1. Identify the midpoint of tasks in the phase
2. Run code review for first half of tasks (commits for tasks 1 through N/2)
3. Fix any issues found
4. Run code review for second half of tasks (commits for tasks N/2+1 through N)
5. Fix any issues found

**When issues are found:**

1. **Do not dispatch `task-bug-fixer` directly in this skill.**
2. **Delegate all fix/re-review actions to `requesting-code-review`** (it owns cycle files, bug-fixer dispatch, and re-review sequencing).
3. **Proceed only when `requesting-code-review` returns zero issues.**

**Plan execution policy (stricter than general code review):**
- ALL issues must be fixed (Critical, Important, AND Minor)
- Ignore APPROVED/BLOCKED status - count issues only
- **Three-strike rule:** If same issues persist after three review cycles, stop and ask human for help

**Minor issues are NOT optional.** Do not rationalize skipping them with "they're just style issues" or "we can fix those later." The reviewer flagged them for a reason. Fix every single one.

**Exit condition:** Zero issues in all categories — including Minor.

Mark "Phase Nc: Code review" as complete.

#### 3d. Move to Next Phase

Proceed to the next phase's "Prepare" step. Repeat 3a-3c for each phase.

### 4. Update Project Context

After all phases complete, invoke the `ed3d-extending-claude:project-claude-librarian` subagent (when available) to review changes and update CLAUDE.md files if needed.

```
<invoke name="Task">
<parameter name="subagent_type">ed3d-extending-claude:project-claude-librarian</parameter>
<parameter name="description">Updating project context after implementation</parameter>
<parameter name="prompt">
  Review what changed during this implementation and update CLAUDE.md files if contracts or structure changed.

  Base commit: <commit SHA at start of first phase>
  Current HEAD: <current commit>
  Working directory: <directory>

  Follow the ed3d-extending-claude:maintaining-project-context skill to:
  1. Diff against base to see what changed
  2. Identify contract/API/structure changes
  3. Update affected CLAUDE.md files
  4. Commit documentation updates

  Report back with what was updated (or that no updates were needed).
</parameter>
</invoke>
```

**If librarian reports updates:** Review the changes, then proceed to final review.
**If librarian reports no updates needed:** Proceed to final review.
**If librarian subagent is unavailable:** skip this entire step. Say aloud that you're skipping it because the `ed3d-extending-claude` plugin is not available.

### 5. Final Review Sequence

After all phases complete, run a sequence of specialized agents:

```
Code Review → Test Analysis (Coverage + Plan)
```

#### 5a. Final Code Review

Use the `requesting-code-review` skill for final code review:

**Context to provide:**
- WHAT_WAS_IMPLEMENTED: Summary of all phases completed
- PLAN_OR_REQUIREMENTS: Reference to the full implementation plan directory
- BASE_SHA: commit before first phase started
- HEAD_SHA: current commit
- IMPLEMENTATION_GUIDANCE: absolute path (if exists)
- AC_COVERAGE_CHECK: "Verify all acceptance criteria (using scoped format `{slug}.AC*`) from the design plan are covered by at least one phase. Flag any ACs not addressed."

Continue the review loop until zero issues remain.

#### 5b. Test Analysis

**Only after final code review passes with zero issues.**

**Skip this step if test-requirements.md does not exist.**

The test-analyst agent performs two sequential tasks with shared analysis:
1. Validate coverage against acceptance criteria
2. Generate human test plan (only if coverage passes)

Dispatch the test-analyst agent:

```
<invoke name="Task">
<parameter name="subagent_type">ed3d-plan-and-execute:test-analyst</parameter>
<parameter name="description">Analyzing test coverage and generating test plan</parameter>
<parameter name="prompt">
Analyze test implementation against acceptance criteria.

TEST_REQUIREMENTS_PATH: [absolute path to test-requirements.md]
WORKING_DIRECTORY: [project root]
BASE_SHA: [commit before first phase]
HEAD_SHA: [current commit]

Phase 1: Validate that automated tests exist for all acceptance criteria.
Phase 2: If coverage passes, generate human test plan using your analysis.

Return coverage validation result. If PASS, include the human test plan.
</parameter>
</invoke>
```

**If analyst returns coverage FAIL:**

1. Dispatch bug-fixer to add missing tests:
   ```
   <invoke name="Task">
   <parameter name="subagent_type">ed3d-plan-and-execute:task-bug-fixer</parameter>
   <parameter name="description">Adding missing test coverage</parameter>
   <parameter name="prompt">
   Add missing tests identified by the test analyst.

   REVIEW_OUTPUT_FILE: [path to test analyst report]
   WORKING_DIRECTORY: [directory]
   FIX_REPORT_FILE: /tmp/execution-reports/[plan-dir]/final_test_fix_report.md

   For each missing test:
   1. Read the acceptance criterion carefully
   2. Create the test file at the expected location
   3. Write tests that verify the criterion's actual behavior—not just code that passes, but code that would fail if the criterion weren't met
   4. Run tests to confirm they pass
   5. Commit the new tests

   Return a compact summary only.
   </parameter>
   </invoke>
   ```

2. Re-run test-analyst
3. Repeat until coverage PASS or three attempts fail (then escalate to human)

**If analyst returns coverage PASS:**

The response will include the human test plan. Extract the "Human Test Plan" section.

**Write the test plan:**

```bash
# Create test-plans directory if needed
mkdir -p docs/test-plans

# The filename uses the implementation plan directory name
# e.g., impl plan dir: docs/implementation-plans/2025-01-24-oauth/
#       test plan:     docs/test-plans/2025-01-24-oauth.md
```

Write the test plan content to `docs/test-plans/[impl-plan-dir-name].md`, then commit:

```bash
git add docs/test-plans/[impl-plan-dir-name].md
git commit -m "docs: add test plan for [feature name]"
```

Announce: "Human test plan written to `docs/test-plans/[impl-plan-dir-name].md`"

### 6. Complete Development

After final review passes:

- Provide a report to the human operator
  - For each phase:
    - How many tasks were implemented
    - How many review cycles were needed
    - Any compromises made (there should be NO compromises, but if any were made). Examples:
      - "I couldn't run the integration tests, so I continued on"
      - "I couldn't generate the client because the dev environment was down"
      - Note that these are PARTIAL FAILURE CASES and explain to the user what the user must do now.
    - Were any code-review issues left outstanding at any point?

- Activate the `finishing-a-development-branch` skill. DO NOT activate it before this point.

## Example Workflow

```
You: I'm using the `executing-an-implementation-plan` skill.

[Discover phases: phase_01.md, phase_02.md, phase_03.md]
[Read first 3 lines of each to get titles]

[Create tasks with TaskCreate:]
- [ ] Phase 1a: Prepare phase 01 — Project Setup
- [ ] Phase 1b: Execute tasks
- [ ] Phase 1c: Code review
- [ ] Phase 2a: Prepare phase 02 — Token Service
- [ ] Phase 2b: Execute tasks
- [ ] Phase 2c: Code review
- [ ] Phase 3a: Prepare phase 03 — API Middleware
- [ ] Phase 3b: Execute tasks
- [ ] Phase 3c: Code review

--- Phase 1 ---

[Mark 1a in_progress]
[Create /tmp/execution-prep and /tmp/execution-reports directories]
[Dispatch execute-phase-prep]
→ Phase 1 prep complete. Tasks: 2. Summary: /tmp/execution-prep/plan/phase_01/phase_01_summary.md
[Print response, mark 1a complete, 1b in_progress]

[Dispatch task-implementor-fast for Task 1]
→ Status: COMPLETED. Files: package.json, tsconfig.json. Commit: abc123
[Print response]

[Dispatch task-implementor-fast for Task 2]
→ Status: COMPLETED. Files: config/*.json. Commit: def456
[Print response, mark 1b complete, 1c in_progress]

[Use requesting-code-review skill for phase 1]
→ Status: APPROVED. Issues: 0. Report: /tmp/execution-reports/plan/phase_01_review_cycle_1.md
[Print response, mark 1c complete]

--- Phase 2 ---

[Mark 2a in_progress]
[Dispatch execute-phase-prep]
→ Phase 2 prep complete. Tasks: 3. ...
[Print response, mark 2a complete, 2b in_progress]

[Execute all 3 tasks...]

[Mark 2b complete, 2c in_progress]

[Use requesting-code-review skill for phase 2]
→ Status: CHANGES REQUIRED. Issues: Critical: 0 | Important: 1 | Minor: 1
[Print response]
[Dispatch bug-fixer with review file]
→ Fixed: 2 issues. Commit: ghi789
[Print response]
[Re-review]
→ Status: APPROVED. Issues: 0.
[Print response, mark 2c complete]

--- Phase 3 ---

[Similar pattern...]

--- Finalize ---

[Invoke project-claude-librarian subagent]
→ Updated CLAUDE.md.

[Use requesting-code-review skill for final review]
→ All requirements met.

[Transitioning to finishing-a-development-branch]
```

## Common Rationalizations - STOP

| Excuse | Reality |
|--------|---------|
| "I'll read all phases upfront to understand the full picture" | No. Read one phase at a time. Context limits are real. |
| "I'll skip the read step, I remember what's in the file" | No. Delegate to phase-prep subagent. Context may have been compacted. |
| "I'll review after each task to catch issues early" | No. Review once per phase. Task-level review wastes context. |
| "Context error on review, I'll skip the review" | No. Chunk the review into halves. Never skip review. |
| "Minor issues can wait" | No. Fix ALL issues including Minor. |
| "I'll return full reports instead of summaries" | No. Subagents must write to files and return summaries. That's the whole point. |
