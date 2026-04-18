# Phase 2 — Spawn Workstream Agents

Load this reference when executing Phase 2 of `/orchestrate`.

Execute rounds sequentially. Within each round, launch all workstreams in
parallel.

## 2a. Foundation Round (Round 0)

If a foundation workstream exists, execute it directly on the feature branch
(no worktree — it's sequential and must be on the shared base for Round 1):

1. Implement the foundation work on the feature branch
2. Run the verify gate (see `retry-semantics.md`)
3. Commit to the feature branch
4. This becomes the base for all Round 1 worktree agents

## 2b. Parallel Rounds

For each parallel round, launch all worktree agents in a **single message**
for true concurrency. Each agent runs the full pipeline appropriate to its
workstream type (see Pipeline Depth table below).

```
Agent({
  description: "{workstream name}",
  isolation: "worktree",
  prompt: <see Full Pipeline Agent Prompt below>
})
```

### Full Pipeline Agent Prompt

```
You are a workstream agent in a parallel orchestration. You will run the
complete development pipeline for your assigned workstream, independently
and in isolation.

## Your Workstream
Name: {workstream name}
Description: {full workstream description}
Scope globs: {scope_globs}
Acceptance Criteria:
{list of AC with id, text, and verifying check command}

## Broader Context
This workstream is part of a larger build: {1-2 sentence summary}. Other
workstreams running in parallel: {list names only}.

## Interface Contracts (if applicable)
{For each contract this workstream participates in:}

You are the {PRODUCER|CONSUMER} for the following contract:

  Contract: {contract name}
  Producer: {workstream name}
  Consumer: {workstream name}
  Shape:
    {type definition, schema, or structured description}

If you are the PRODUCER: your implementation MUST match this shape exactly.
If you are the CONSUMER: your implementation MUST consume this shape exactly.
Do NOT deviate from the contract without noting it in your workstream report.

## Your Pipeline

### Phase 0: Environment Setup (MANDATORY)
Your worktree is a fresh checkout. Dependencies are NOT pre-installed.

1. Detect the project type and install dependencies:
   - pnpm-lock.yaml → pnpm install
   - yarn.lock → yarn install
   - package.json → npm install
   - requirements.txt → pip install -r requirements.txt
   - pyproject.toml → pip install -e .
   - go.mod → go mod download
   - Cargo.toml → cargo fetch

2. Verify the toolchain works (tsc --noEmit, go build ./..., cargo check, etc).

If dependency installation fails, report FAILED immediately — do not attempt
to implement without a working environment.

### Phase 1: Research (if applicable)
Skip if scope is obvious (bug fix, tests only, migration).
Otherwise: read existing code, identify conventions, check CLAUDE.md.

### Phase 2: Plan (if applicable)
Skip for single-task workstreams (< 3 files).
Otherwise: design approach, order tasks, plan test strategy.

### Phase 3: Implement
- Follow existing project conventions
- Write production-quality code — not stubs or TODOs
- Create/update tests alongside implementation

**SCOPE DISCIPLINE**: You have been assigned `scope_globs: {scope_globs}`.
Files outside this scope will be flagged at merge time (Phase 3.5c). If you
MUST touch a file outside your scope, record it in your workstream report
under "Scope deviations" with a one-line justification. Out-of-scope changes
without a justification will block your merge.

**REGRESSION GUARD**: When modifying a file with existing functionality:
- Catalog all existing public interfaces BEFORE changing
- Verify EVERY pre-existing interface still works AFTER changing
- Diff your version against original; confirm no unintended deletions

**MANDATORY: Commit after every logical unit of work.** Examples:
- A new file or module
- A completed function/class with its tests
- A configuration change
- Any point where `git diff --stat` shows 3+ files changed

After each unit: `git add -A && git commit -m "{workstream}: {what}"`.

### Phase 3.5: Commit Gate (MANDATORY)
Before the verify gate, ensure ALL work is committed:
```
git status --porcelain   # must be empty
```

If non-empty: commit remaining work. An agent that finishes with uncommitted
changes has FAILED its workstream — the orchestrator will either quarantine
the branch (recovering your work but blocking the merge) or hard-stop the
round, both of which delay the overall run.

### Phase 4: Verify Gate (MANDATORY, up to 3 retries)
See `retry-semantics.md` for the full retry protocol. In summary:

Run the project's full verify command set:
- Type check (tsc --noEmit, mypy, go build, cargo check)
- Tests (full suite, not partial)
- Lint (eslint, biome, ruff, golangci-lint, clippy)

If ANY fails:
- Attempt 1: Feed failure output back into context, re-run in place
- Attempt 2: Reduce scope — revert unrelated changes, focus on minimal failing slice
- Attempt 3: Fresh-agent reset — handed off to a new subagent with original
  prompt + failure summary + explicit "choose a different approach"

If all 3 attempts fail: mark workstream as FAILED, do not claim completion.

### Phase 5: Self-Review
- Security issues, missing error handling, untested paths
- Verify every acceptance criterion has a passing check
- Clean up debug code, console.logs, commented-out code

## Report Format
When complete, output this report:

---
WORKSTREAM REPORT: {workstream name}
Pipeline phases completed: {list}
Status: COMPLETE / PARTIAL / FAILED
Verify attempts used: {1-3}

Research summary: {1-2 sentences or "skipped"}
Plan summary: {key decisions or "skipped"}

Implementation:
  Files created: {list}
  Files modified: {list}
  Scope deviations: {list with justifications, or "none"}
  Tests added: {count}

Acceptance Criteria:
  - AC1: {id}: PASS via `{check command}` (output: {tail})
  - AC2: {id}: PASS via `{check command}` (output: {tail})
  - ...

Verification:
  Type check: PASS/FAIL/N/A
  Tests: PASS/FAIL ({passing}/{total})
  Lint: PASS/FAIL/N/A

Git status:
  Total commits on branch: {git rev-list --count HEAD ^$(git merge-base HEAD main)}
  Clean worktree: {YES/NO — must be YES}
  Last commit: {git log --oneline -1}

**BEFORE REPORTING**: run `git status`. If non-empty, commit, then report.
If "Clean worktree: NO", you have failed.
---
```

## 2c. Pipeline Depth Per Workstream

| Workstream Type | Pipeline |
|----------------|----------|
| New feature (complex) | research → plan → implement → verify → review |
| New feature (clear scope) | plan → implement → verify → review |
| Bug fix | implement → verify → review |
| Tests only | implement → verify |
| Migration / config | implement → verify |
| Integration layer | plan → implement → verify → review |

Include the applicable depth in the agent's prompt.

## 2d. Collect Results

After all agents in the round complete:
- Collect branch name and worktree path from each
- Parse workstream reports
- Log results to `state.json`

**If an agent failed**:
1. Log the failure with its report
2. Check if it blocks subsequent rounds
3. If blocking: retry once with error context in the prompt
4. If still failing: stop and report to user with options (retry / skip / abort)
