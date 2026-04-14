# Agent Verification Protocol

Every implementation agent MUST follow this protocol before reporting completion. No exceptions.

## The Three Gates

### Gate 1: TypeScript Compilation

```bash
cd app && npx tsc --noEmit 2>&1
```

**Rules**:
- Errors in YOUR files: fix them immediately
- Errors in OTHER files (not yours): report them but don't modify those files
- New type errors introduced by your changes: always your responsibility
- Pre-existing type errors: not your responsibility, just report

**Common fixes**:
- Missing type imports → add the import
- Type mismatch → fix the type, don't use `any`
- Missing properties → add required properties
- Nullable types → add proper null checks

### Gate 2: Test Execution

```bash
cd app && npx vitest run {your-test-files} 2>&1
```

**Rules**:
- If you created/modified test files: run them specifically
- If you modified source files with existing tests: run those tests
- If no tests exist for your files: note this in your report (don't skip the gate)
- Failed tests: fix and re-run (max 3 attempts)

**Test fix attempt cycle**:
1. Read the error message carefully
2. Identify root cause (not symptom)
3. Fix the actual issue (not the test assertion)
4. Re-run the specific failing test
5. If fixed, run full test file to ensure no regressions

**After 3 failed attempts**: Stop trying. Report the failure with:
- The test name and file
- The error message
- What you tried
- Your theory on the root cause

### Gate 3: Prohibited Pattern Scan

Search your modified files for these patterns. If found, remove them.

**Prohibited patterns**:
| Pattern | Why Prohibited | Fix Instead |
|---------|---------------|-------------|
| `// @ts-ignore` | Hides type errors | Fix the actual type issue |
| `// @ts-expect-error` | Hides type errors | Fix the actual type issue |
| `eslint-disable` | Hides lint errors | Fix the lint issue |
| `as any` | Type escape hatch | Use proper type assertion or fix the type |
| `: any` | Untyped code | Define or import the correct type |
| `.skip(` in tests | Skips tests | Fix or remove the test |
| `catch {}` or `catch (e) {}` | Swallowed errors | Log or re-throw the error |
| `console.log` (non-test) | Debug artifacts | Remove or use proper logging |

**Exceptions**:
- `any` in test mock setup is acceptable when mocking complex third-party types
- `console.error` in catch blocks is acceptable
- `// @ts-expect-error` with a preceding comment explaining a genuine library type bug (rare)

## Report Format

After completing all three gates, output this exact format:

```
---
AGENT REPORT
Tasks: {completed}/{total}
TSC: PASS/FAIL ({error count in your files} errors)
Tests: PASS/FAIL ({passing}/{total} tests, {attempt count} attempts)
Prohibited patterns: CLEAN/FOUND ({patterns found and locations})
Files modified:
  - {path}: {what was done}
  - {path}: {what was done}
Issues:
  - {any unresolved problems}
  - {any concerns or notes for the team}
---
```

## Decision Tree

```
Start
  → Implement tasks
  → Run tsc --noEmit
    → Errors in my files? → Fix → Re-run tsc
    → Errors in other files? → Report, continue
    → Clean? → Proceed
  → Run tests
    → Pass? → Proceed
    → Fail? → Fix → Re-run (max 3x)
    → Still failing? → Report failure, STOP
  → Scan for prohibited patterns
    → Found? → Remove → Re-run tsc + tests
    → Clean? → Proceed
  → Generate report
  → Done
```

## Key Principles

1. **Own your files**: You are responsible for everything in the files assigned to you
2. **Don't touch others' files**: Even if you see issues — report them instead
3. **Fix forward**: When tests fail, fix the code, not the test expectations
4. **No shortcuts**: Prohibited patterns exist because they hide real bugs
5. **Report honestly**: A reported failure is better than a hidden one
