# Review Loop Protocol

Auto-review each worktree branch before merging. Each workstream agent already ran
its own self-review as the final pipeline phase. This external review is a second
pair of eyes — an independent check before integration.

Fix issues and re-review, up to 3 iterations.

## Overview

```
For each worktree branch:
  External review (code-review agent)
    → APPROVE → proceed to merge
    → REQUEST_CHANGES → fix agent → re-review (max 3x)
    → 3 failures → merge with warning
```

## Context

Each workstream agent ran the full /workflow pipeline:
  research → plan → implement → verify → self-review

The self-review phase means obvious issues should already be caught.
This external review focuses on things the agent might have blind spots for:
- Security issues the author didn't consider
- Convention violations from an agent unfamiliar with the project
- Integration concerns (will this merge cleanly with other workstreams?)
- Acceptance criteria gaps

## Step 1: Launch Review Agent

For each completed worktree branch, launch a review agent:

```
Agent({
  description: "Review: {workstream name}",
  subagent_type: "rpi:code-reviewer",
  prompt: "Review the complete implementation of workstream: {workstream name}

  ## Context
  This workstream was developed in isolation by a worktree agent that ran the
  full development pipeline (research → plan → implement → verify → self-review).
  The agent's own assessment is below — use it as context, not as gospel.

  Agent's self-review notes:
  {paste self-review section from workstream report}

  Acceptance criteria for this workstream:
  {paste from orchestration plan}

  ## What to Review
  Review all files created or modified by this workstream.
  The changes are on branch: {worktree-branch}

  ## Review Criteria
  1. **Correctness**: Does the code meet the acceptance criteria?
  2. **Security**: No injection vulnerabilities, proper auth checks, no secrets
  3. **Conventions**: Follows project patterns (check CLAUDE.md if present)
  4. **Tests**: Adequate coverage, meaningful assertions, edge cases
  5. **Integration readiness**: Will this merge cleanly with other workstreams?

  ## What NOT to Review
  - Style preferences (formatting is handled by lint)
  - Pre-existing issues in files not modified by this workstream
  - Hypothetical future requirements
  - Issues the agent already flagged as known limitations

  ## Output Format
  Respond with EXACTLY one of:

  VERDICT: APPROVE
  Summary: {1-2 sentence summary of what looks good}

  OR

  VERDICT: REQUEST_CHANGES
  Changes:
  1. [{SEVERITY}] {file}:{line} — {what to change and why}
  2. [{SEVERITY}] {file}:{line} — {what to change and why}
  ...

  Severity levels:
  - CRITICAL: Must fix (security issue, logic error, data loss risk)
  - HIGH: Should fix (missing error handling, missing test, broken edge case)
  - MEDIUM: Nice to fix (suboptimal pattern, minor improvement)

  Only REQUEST_CHANGES for CRITICAL and HIGH items.
  MEDIUM items should be noted but don't block approval."
})
```

### Parallel Reviews

When multiple branches complete in the same round, launch ALL review agents
in a single message so they run concurrently. Reviews are read-only and
independent — no reason to serialize them.

## Step 2: Handle Verdict

### APPROVE

Log the approval and proceed to merge:
```json
{
  "task": "{task name}",
  "branch": "{worktree-branch}",
  "verdict": "APPROVE",
  "iteration": 1,
  "summary": "{review summary}"
}
```

### REQUEST_CHANGES

Enter the fix cycle.

## Step 3: Fix Cycle

### 3a. Launch Fix Agent

Send the review feedback to a fix agent working on the same branch:

```
Agent({
  description: "Fix review items: {task name}",
  isolation: "worktree",
  prompt: "You are fixing code review items on an existing worktree branch.

  ## Branch
  {worktree-branch}

  ## Review Feedback (iteration {N} of 3)
  {paste the REQUEST_CHANGES items here}

  ## Instructions
  1. Address each CRITICAL and HIGH item
  2. MEDIUM items: fix if straightforward, skip if complex
  3. Run verification after all fixes:
     - Type check (if applicable)
     - Tests
     - Lint
  4. Commit your fixes with message: 'fix: address review feedback (iteration {N})'

  ## Report
  When done, output:
  ---
  FIX REPORT
  Items addressed: {count}/{total}
  Items skipped: {list with reason}
  Verification: PASS/FAIL
  ---"
})
```

**Important**: The fix agent needs to work on the SAME branch that was reviewed.
Since the original worktree agent has completed, its worktree path and branch are
returned in the agent result. Pass the branch name context to the fix agent so it
can check out and work on the correct branch.

### 3b. Re-Review

After the fix agent completes, launch a fresh review agent on the updated branch.
Use the same review prompt as Step 1.

### 3c. Iteration Tracking

Track iterations in state.json:
```json
{
  "task": "{task name}",
  "branch": "{worktree-branch}",
  "reviews": [
    {"iteration": 1, "verdict": "REQUEST_CHANGES", "items": 3},
    {"iteration": 2, "verdict": "REQUEST_CHANGES", "items": 1},
    {"iteration": 3, "verdict": "APPROVE", "items": 0}
  ]
}
```

### 3d. Max Iterations Reached

If after 3 review-fix cycles the branch is still not approved:

1. Log a warning:
   ```json
   {
     "task": "{task name}",
     "warning": "Merged without full approval after 3 review iterations",
     "outstanding_items": ["{list of unresolved items}"]
   }
   ```
2. Proceed to merge anyway — don't block the orchestration
3. The outstanding items will be flagged in the final orchestration report
4. If the user runs `pw-review` on the full feature branch afterward,
   these items will surface again

## Review Log

Maintain a running review log at `.pipeline/$RUN_NAME/review-log.md`:

```markdown
# Review Log

## {Task Name} — {branch}
- **Iteration 1**: REQUEST_CHANGES (2 critical, 1 high)
  - [CRITICAL] Missing auth check on /api/users endpoint
  - [HIGH] No error handling in fetchData()
  - [MEDIUM] Consider using useMemo for expensive computation
- **Iteration 2**: APPROVE
  - All critical and high items addressed
  - Medium item deferred

## {Task Name} — {branch}
- **Iteration 1**: APPROVE
  - Clean implementation, good test coverage
```

## Timing

Typical durations:
- Review agent: 15-30 seconds
- Fix agent: 30-90 seconds
- Full cycle (review + fix + re-review): ~2 minutes
- 3 iterations worst case: ~6 minutes per branch

Reviews for branches in the same round run in parallel, so wall-clock time
is determined by the slowest branch, not the sum.
