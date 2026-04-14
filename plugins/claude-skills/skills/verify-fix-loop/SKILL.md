---
name: verify-fix-loop
description: |
  Run verification commands and iteratively fix errors until all checks pass. Use when:
  (1) User wants to verify code quality (tests, lint, build)
  (2) User asks to "fix all errors" or "make tests pass"
  (3) User mentions verification, validation, or CI-style checks
  (4) User wants iterative fixing of build/test/lint issues
  Runs `make verify` (or configured command), analyzes failures with Tech Lead agent,
  applies fixes in parallel with code simplification, and loops until success.
---

# Verify-Fix Loop

Iteratively run verification and fix errors until all checks pass.

## Workflow

### 1. Run Verification

Execute the verification command in the target directory:

```bash
cd app && make verify
```

Capture full output including exit code.

### 2. Analyze Results

If verification passes (exit 0): Report success and stop.

If verification fails: Launch **product-team:tech-lead** agent to analyze errors:

```
Analyze these verification failures and create a fix plan:

<errors>
{verification output}
</errors>

Requirements:
- Identify root causes, not symptoms
- Fixes must solve the actual problem, not mask it (no `// @ts-ignore`, no `eslint-disable`, no skipping tests)
- Group related errors that share a common fix
- Prioritize: type errors > lint errors > test failures > build issues
- For each fix, explain WHY it solves the problem
- Group fixes by file so they can be applied in parallel
- Output a structured list of fixes with file paths
```

### 3. Apply Fixes in Parallel

Launch multiple subagents **in parallel** to apply fixes. Group fixes by file - each file gets its own agent.

For independent fixes (different files), launch agents simultaneously in a single message with multiple Task tool calls:

```
# Launch these in parallel (single message, multiple tool calls):

Task 1: Fix {file1}
- Apply fix: {description}
- Run code-simplifier agent on file after fix

Task 2: Fix {file2}
- Apply fix: {description}
- Run code-simplifier agent on file after fix

Task 3: Fix {file3}
...
```

Each fix agent should:
1. Read the target file
2. Apply the specific fix using Edit tool
3. Launch **code-simplifier** agent to simplify the modified code

### 4. Code Simplification

After each fix, the **code-simplifier** agent reviews the file:
- Preserves all functionality
- Applies project standards from CLAUDE.md
- Reduces unnecessary complexity and nesting
- Eliminates redundant code
- Improves readability
- Avoids nested ternary operators

### 5. Loop

Return to Step 1. Continue until verification passes.

**Safety limit**: Maximum 10 iterations. If not resolved, report remaining issues and stop.

## Parallelization Rules

- Fixes to **different files**: Apply in parallel
- Fixes to **same file**: Apply sequentially within one agent
- Code simplification: Runs immediately after each fix, within the same agent

## Prohibited Fix Patterns

Never apply these "fixes" - they mask problems rather than solving them:

- `// @ts-ignore` or `// @ts-expect-error`
- `eslint-disable` comments
- `any` type to silence TypeScript
- `.skip()` on failing tests
- Empty catch blocks
- Deleting tests that fail
- Loosening type definitions to avoid errors

## Output Format

After each iteration:
```
## Iteration {n}

**Verification result**: PASS/FAIL
**Errors found**: {count}
**Agents launched**: {count} (in parallel)
**Fixes applied**:
- {file}: {description of fix}

**Simplifications**:
- {file}: {what was simplified}
```

On completion:
```
## Complete

Verification passed after {n} iteration(s).

**Summary of changes**:
- {list of all files modified with brief descriptions}
```
