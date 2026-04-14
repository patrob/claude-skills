---
name: pw-review
description: |
  Multi-lens parallel code review. Launch 5 parallel reviewers with outcome-based lenses,
  consolidate into severity-ranked report. Use when:
  (1) Code is ready for review before commit or PR
  (2) User says "review", "code review", "check my changes"
  (3) User wants pre-commit quality check
  (4) After implementation, before merge
  Works on uncommitted changes, branch diffs, or PR diffs.
---

# Multi-Lens Parallel Code Review

Launch 5 parallel read-only reviewers with outcome-based lenses. Consolidate findings into a severity-ranked report.

## Input

Determine the diff to review. In order of preference:

1. If the user specifies a PR number: `gh pr diff {number}`
2. If the user specifies a branch: `git diff main...{branch}`
3. If there are uncommitted changes: `git diff HEAD` (staged + unstaged)
4. Default: `git diff main...HEAD` (current branch vs main)

Save the diff to `.pipeline/{run-name}/diff.txt` for agents to reference.

## Setup

1. Create the run directory:
   ```bash
   mkdir -p .pipeline/pw-review-$(date +%Y%m%d-%H%M%S)
   ```
2. Save the diff to `diff.txt` in the run directory
3. List all changed files for agents to read

## Workflow

### 1. Launch 5 Parallel Review Agents

Launch all 5 agents simultaneously using the Agent tool. Each agent is **read-only** — they search, read, and analyze but never edit files.

Each agent receives:
- The list of changed files
- The diff content (or path to diff file)
- Their specific lens criteria (from `{SKILL_DIR}/references/review-lenses.md`)
- Instructions to output findings in the standard format below

**Agent 1 — UX Lens** (subagent_type: `Explore`):
```
Review these changes through a UX lens. Focus on:
- Mobile responsiveness (this is a mobile-first web app)
- Touch targets (minimum 44x44px)
- Accessibility (ARIA, semantic HTML, keyboard nav)
- Loading states and error states
- User feedback for actions
- Consistent styling with Tailwind/shadcn patterns
Read the review lens criteria at: {SKILL_DIR}/references/review-lenses.md (UX section)
```

**Agent 2 — Security Lens** (subagent_type: `Explore`):
```
Review these changes through a Security lens. Focus on:
- Auth checks (withAuth/withOptionalAuth on API routes)
- Input validation and sanitization
- XSS/injection vulnerabilities
- Data exposure in API responses
- Feature flag gating for new features
- JWT token handling
Read the review lens criteria at: {SKILL_DIR}/references/review-lenses.md (Security section)
```

**Agent 3 — Performance Lens** (subagent_type: `Explore`):
```
Review these changes through a Performance lens. Focus on:
- N+1 database queries
- Missing indexes on query fields
- Unnecessary re-renders in React components
- Bundle size impact (new dependencies, large imports)
- DB query efficiency (.lean(), projection, pagination)
- Caching opportunities
Read the review lens criteria at: {SKILL_DIR}/references/review-lenses.md (Performance section)
```

**Agent 4 — Test Coverage Lens** (subagent_type: `Explore`):
```
Review these changes through a Test Coverage lens. Focus on:
- Missing unit tests for new functions/components
- Edge cases not covered
- Mock quality (full chain mocking per CLAUDE.md)
- E2E test gaps for user-facing changes
- Test isolation (no shared state between tests)
Read the review lens criteria at: {SKILL_DIR}/references/review-lenses.md (Test Coverage section)
```

**Agent 5 — Production Readiness Lens** (subagent_type: `Explore`):
```
Review these changes through a Production Readiness lens. Focus on:
- Error handling (try/catch, error boundaries)
- Logging for debugging
- Feature flags for new features (LaunchDarkly)
- Database migration needs
- Rollback safety
- Environment variable handling
Read the review lens criteria at: {SKILL_DIR}/references/review-lenses.md (Production Readiness section)
```

### 2. Collect and Consolidate

After all 5 agents complete:

1. Collect all findings
2. Deduplicate (same issue found by multiple lenses)
3. Assign severity:
   - **Critical**: Security vulnerabilities, data loss risks, auth bypasses
   - **High**: Missing error handling, broken mobile UX, N+1 queries
   - **Medium**: Missing tests, accessibility gaps, suboptimal patterns
   - **Low**: Style inconsistencies, minor improvements, nice-to-haves

### 3. Generate Report

Write `.pipeline/{run-name}/review-summary.md`:

```markdown
# Code Review Summary

**Diff**: {description of what was reviewed}
**Date**: {timestamp}
**Files reviewed**: {count}

## Critical ({count})
- [{Lens}] {Finding} — {file}:{line}
  - Impact: {why this matters}
  - Fix: {suggested resolution}

## High ({count})
...

## Medium ({count})
...

## Low ({count})
...

## Positive Observations
- {Things done well worth noting}

## Verdict
{PASS / PASS WITH NOTES / NEEDS CHANGES}
{Summary sentence}
```

### 4. Present to User

Display the review summary directly. Highlight critical and high findings. Ask if the user wants to:
- Fix specific findings
- Proceed to commit despite findings
- Run `pw-implement` to address findings as tasks

## Finding Format (Per Agent)

Each agent must output findings in this format:
```
### {Lens Name} Review

**Files reviewed**: {list}

#### Findings

1. **[{SEVERITY}]** {Title}
   - File: {path}:{line}
   - Issue: {description}
   - Impact: {why it matters}
   - Suggestion: {how to fix}

#### No Issues Found In
- {area checked that was clean}
```

## Notes

- Agents are read-only — they never modify files
- Each agent should read the full changed files, not just the diff, to understand context
- Agents should check patterns against CLAUDE.md conventions
- Run time: typically 30-60 seconds for all 5 agents in parallel
