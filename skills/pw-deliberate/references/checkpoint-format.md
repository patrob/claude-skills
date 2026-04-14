# Checkpoint Format Template

Use this template for each checkpoint in the plan.

```markdown
### Checkpoint {N}: {Descriptive Name}

**Goal**: {One sentence describing what this checkpoint achieves}

**Tasks**:
- [ ] {Task 1} — `{file path}`
- [ ] {Task 2} — `{file path}`
- [ ] {Task 3} — `{file path}`

**Files to modify**:
- `{path/to/file1.ts}` — {what changes}
- `{path/to/file2.tsx}` — {what changes}

**Success criteria**:
- [ ] {Specific, testable criterion 1}
- [ ] {Specific, testable criterion 2}
- [ ] {Specific, testable criterion 3}

**Verification command**:
```bash
{command that proves this checkpoint is complete}
```

**Parallelization**:
- Parallel: {tasks that can run concurrently — different files}
- Sequential: {tasks that must run in order — shared dependencies}
- Agent count: {number of parallel agents for this checkpoint}
```

## Guidelines

### Good Success Criteria
- "API route returns 200 with user data when authenticated"
- "Component renders loading spinner during fetch"
- "Test file has 3+ test cases covering happy path and error"
- "`tsc --noEmit` passes with zero errors"
- "Mobile viewport shows single-column layout at 375px"

### Bad Success Criteria
- "Code is clean" (subjective)
- "Feature works" (vague)
- "Tests pass" (which tests?)
- "UI looks good" (not measurable)

### Good Verification Commands
- `cd app && npx vitest run src/features/groups/services/group-service.test.ts`
- `cd app && npx tsc --noEmit`
- `cd app && npx eslint src/features/check-in/ --no-warn`
- `cd app && make verify`

### Parallelization Rules
- Different files = can run in parallel
- Same file = must be sequential
- Type definitions before implementations that use them = sequential
- API route before UI that calls it = sequential (unless mocked)
- Test files can parallel with source files if writing both
