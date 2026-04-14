---
name: pw-bug-hunt
description: |
  Bug-first TDD fix workflow. Write a failing test first, then fix. Use when:
  (1) A bug is reported (user says "bug", "fix", "broken", "not working")
  (2) User reports unexpected behavior
  (3) A test is failing and needs investigation
  (4) Something that used to work is now broken
  Enforces test-first discipline: investigate → write failing test → fix → verify.
  Per CLAUDE.md: bug fixes do NOT need feature flags.
---

# Bug-First TDD Fix

Investigate with parallel hypotheses, write a failing test, then fix.

## Input

Accept one of:
1. A bug description from the user
2. A GitHub issue number (fetch with `gh issue view {number}`)
3. An error message or stack trace
4. A failing test name

## Setup

1. Create run directory:
   ```bash
   mkdir -p .pipeline/{run-name}
   ```

## Workflow

### 1. Investigate with Parallel Hypotheses

Based on the bug description, form 2-3 hypotheses about the root cause. Launch parallel agents to investigate each.

**Agent template** (subagent_type: `Explore`):
```
Investigate this bug under the following hypothesis:

Bug: {bug description}
Hypothesis: {specific theory about what's wrong}

Your job:
1. Search the codebase for evidence supporting or refuting this hypothesis
2. Read relevant files thoroughly
3. Check recent git history for related changes: `git log --oneline -10 -- {relevant paths}`
4. Look for similar patterns elsewhere that work correctly (comparison)
5. Check for known gotchas in CLAUDE.md

Report:
### Hypothesis: {hypothesis}
**Verdict**: LIKELY / UNLIKELY / CONFIRMED
**Evidence**:
- {evidence for or against}
**Root cause** (if confirmed): {exact cause with file:line}
**Affected files**: {list}
**Suggested fix**: {brief description}
```

Launch 2-3 agents in parallel, one per hypothesis. Example hypotheses:
- "The query is missing collation for case-insensitive search"
- "The auth middleware isn't extracting the user ID correctly"
- "The component isn't handling the loading state"

### 2. Synthesize Root Cause

After agents complete:
1. Compare verdicts — which hypothesis has the strongest evidence?
2. If multiple are CONFIRMED, identify the primary cause
3. If none are CONFIRMED, form a new hypothesis from the combined evidence and investigate further (max 1 additional round)

Document the root cause:
```markdown
## Root Cause
**File**: {path}:{line}
**Issue**: {description of what's wrong}
**Why it happens**: {explanation}
**Evidence**: {what confirmed this}
```

### 3. Write Failing Test

**Before writing any fix**, create a test that reproduces the bug.

Rules:
- Test must fail for the RIGHT reason (the actual bug, not a setup issue)
- Test must be specific (not a broad integration test)
- Follow existing test patterns in the codebase
- Use proper mock chains per CLAUDE.md (`.find()` → `.lean()` → `.exec()` etc.)

```bash
# Run the test to confirm it fails
cd app && npx vitest run {test-file} 2>&1
```

**Verify the failure**:
- Does the test fail? (Good — it should)
- Does it fail for the right reason? (Check the error message matches the bug)
- If it passes: the test doesn't reproduce the bug — rewrite it

Save the failing test output for comparison.

### 4. Implement Fix

Apply the minimal fix for the root cause.

Rules:
- Fix the root cause, not the symptom
- Minimal change — don't refactor while fixing
- Bug fixes do NOT need feature flags (per CLAUDE.md)
- Don't fix unrelated issues in the same files

### 5. Verify Fix

Run the verification sequence:

```bash
# 1. Run the specific test — should now PASS
cd app && npx vitest run {test-file}

# 2. Run tsc to check types
cd app && npx tsc --noEmit

# 3. Run full verification
cd app && make verify
```

**If the test still fails**: Re-examine the fix. Max 3 fix attempts before escalating to user.

**If other tests break**: The fix introduced a regression — investigate and adjust.

### 6. Generate Report

Write `.pipeline/{run-name}/bug-report.md`:

```markdown
# Bug Fix: {Title}

**Date**: {timestamp}
**Reported**: {bug description}

## Investigation
**Hypotheses tested**: {count}
1. {Hypothesis 1}: {verdict}
2. {Hypothesis 2}: {verdict}

## Root Cause
**File**: {path}:{line}
**Issue**: {description}
**Why**: {explanation}

## Test
**File**: {test path}
**Test name**: {test name}
**Before fix**: FAIL — {error message}
**After fix**: PASS

## Fix
**Files modified**:
- {file}: {what changed and why}

## Verification
- Specific test: PASS
- TSC: PASS
- Full verify: PASS/FAIL

## Regression Check
{Any other tests affected? Yes/No}
```

### 7. Present to User

Display:
- Root cause summary
- What the test covers
- What was fixed
- Verification results

Ask if the user wants to:
- Commit the fix
- Review the changes (`pw-review`)
- Add additional test cases

## Escalation

If after investigation no root cause is found:
1. Report what was investigated
2. List remaining theories
3. Ask the user for additional context:
   - Steps to reproduce
   - When it started happening
   - Any recent changes to related code

## Notes

- ALWAYS write the test before the fix — this is non-negotiable
- The test must fail before the fix proves the bug exists
- The test must pass after the fix proves the fix works
- Bug fixes do NOT need feature flags (CLAUDE.md rule)
- Parallel hypotheses save time — don't investigate sequentially
- Check CLAUDE.md database gotchas — many bugs come from those patterns
