---
name: orchestrate
description: |
  Full-pipeline parallel orchestration. Parse a roadmap, spec, or plan into independent
  workstreams, spawn isolated worktree agents that each run the complete /workflow
  pipeline (research → deliberate → implement → verify → review), then merge all
  results into a single feature branch. Use when:
  (1) A roadmap, spec, or plan has multiple independent features/modules
  (2) User says "orchestrate", "parallel build", "build the roadmap"
  (3) User wants high-quality parallel implementation with full pipeline per workstream
  (4) Multiple features can be developed independently then integrated
  Each worktree agent gets the full quality pipeline, not just implementation.
---

# Full-Pipeline Parallel Orchestration

Each workstream gets its own worktree agent running the complete /workflow pipeline.
The orchestrator manages parallelism, integration, and the final merge.

## Input

Accept one of:
1. A roadmap, feature spec, or PRD (`.md` file)
2. A `plan.md` from `pw-deliberate` or `/plan`
3. Inline description of multiple workstreams
4. Auto-detect: look for the most recent `.pipeline/*/plan.md`

## Setup

1. Create run directory:
   ```bash
   RUN_NAME="orchestrate-$(date +%Y%m%d-%H%M%S)"
   mkdir -p .pipeline/$RUN_NAME
   ```

2. Read the input document and identify all workstreams

3. Create the feature branch that will collect all work:
   ```bash
   FEATURE_BRANCH="orchestrate/$RUN_NAME"
   git checkout -b $FEATURE_BRANCH
   ```

4. Write initial state to `.pipeline/$RUN_NAME/state.json`:
   ```json
   {
     "run_name": "{RUN_NAME}",
     "feature_branch": "{FEATURE_BRANCH}",
     "base_branch": "{branch we started from}",
     "status": "running",
     "current_round": 0,
     "rounds": [],
     "error": null
   }
   ```

## Workflow

### Phase 1: Decompose Into Workstreams

Parse the input and extract discrete workstreams. A workstream is a unit of work
that can be independently researched, planned, implemented, verified, and reviewed.

#### 1a. Extract Workstreams

From the input, identify each workstream with:
- **Name**: short identifier (e.g., "auth-service", "video-upload-api")
- **Description**: full scope of what needs to be built
- **Domain**: which area of the codebase it touches (backend, frontend, infra, etc.)
- **Files likely affected**: best estimate of files created/modified
- **Dependencies**: does this workstream need another to complete first?
- **Acceptance criteria**: how do we know this workstream is done?

#### 1b. Dependency & Overlap Analysis

For each pair of workstreams:
- **File overlap check**: do they modify the same files?
- **Interface dependency**: does one produce an API/type/schema the other consumes?
- **Shared infrastructure**: do they both need a shared piece (e.g., DB migration)?

Categorize relationships:
- **Independent**: no overlap, no dependency → parallel
- **Consumer/Producer**: one needs the other's output → sequential
- **Shared foundation**: both need a common base → extract foundation as its own workstream, run first

#### 1c. Build Round Schedule

Group workstreams into rounds:

```
Round 0 (foundation — sequential if needed):
  - Shared DB migrations, shared types, shared config

Round 1 (parallel — independent workstreams):
  - [worktree] auth-service: full /workflow pipeline
  - [worktree] video-upload-api: full /workflow pipeline
  - [worktree] ui-components: full /workflow pipeline

Round 2 (parallel — depends on Round 1):
  - [worktree] api-integration: full /workflow pipeline
  - [worktree] upload-ui: full /workflow pipeline

Round 3 (integration):
  - Integration tests, E2E tests, cross-cutting concerns
```

#### 1d. Present Orchestration Plan

Display to the user before proceeding:

```
Orchestration Plan: {input title}
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Round 0 — Foundation (sequential):
  1. shared-types: Database schema + shared TypeScript types
     Pipeline: implement → verify (no research needed, scope is clear)

Round 1 — Core Features (3 parallel agents, each running full /workflow):
  1. [worktree: auth-service] Authentication & authorization
     Pipeline: research → deliberate → implement → verify → review
  2. [worktree: video-api] Video upload & processing API
     Pipeline: research → deliberate → implement → verify → review
  3. [worktree: clip-editor] Clip editor UI components
     Pipeline: research → deliberate → implement → verify → review

Round 2 — Integration (2 parallel agents):
  1. [worktree: api-wiring] Wire frontend to APIs
     Pipeline: implement → verify → review (plan is known from Round 1)
  2. [worktree: auth-ui] Auth UI pages
     Pipeline: implement → verify → review

Round 3 — Cross-Cutting (sequential):
  1. Integration tests + E2E tests
     Pipeline: implement → verify

Estimated parallel agents: 3 max concurrent
Proceed? (y/n)
```

If running in auto mode, skip the confirmation.

### Phase 2: Execute Rounds

Execute rounds sequentially. Within each round, launch all workstreams in parallel.

#### 2a. Foundation Round (Round 0)

If a foundation workstream exists, execute it directly (no worktree needed since
it's sequential and must be on the feature branch):

1. Implement the foundation work on the feature branch
2. Run verification
3. Commit to the feature branch
4. This becomes the base for all Round 1 worktree agents

#### 2b. Parallel Rounds

For each parallel round, launch worktree agents. Each agent runs the
**full /workflow pipeline** appropriate to its workstream.

**Worktree Initialization**: Each agent handles its own environment setup
(dependency installation, toolchain verification) via Phase 0 in the pipeline
below. The orchestrator does NOT need to pre-install anything.

Launch ALL agents in the round in a **single message** for true concurrency.

```
Agent({
  description: "{workstream name}",
  isolation: "worktree",
  prompt: <see Full Pipeline Agent Prompt below>
})
```

**Full Pipeline Agent Prompt**:
```
You are a workstream agent in a parallel orchestration. You will run the complete
development pipeline for your assigned workstream, independently and in isolation.

## Your Workstream
Name: {workstream name}
Description: {full workstream description}
Acceptance Criteria: {list}

## Broader Context
This workstream is part of a larger build: {1-2 sentence summary of the overall
project}. Other workstreams running in parallel: {list names only — don't worry
about their implementation, just know they exist}.

## Your Pipeline

Execute these phases in order. Each phase builds on the previous.

### Phase 0: Environment Setup (MANDATORY)
Your worktree is a fresh checkout. Dependencies are NOT pre-installed.

1. Detect the project type and install dependencies:
```bash
# Node.js
if [ -f pnpm-lock.yaml ]; then pnpm install;
elif [ -f yarn.lock ]; then yarn install;
elif [ -f package.json ]; then npm install; fi

# Python
if [ -f requirements.txt ]; then pip install -r requirements.txt;
elif [ -f pyproject.toml ]; then pip install -e .; fi

# Go
if [ -f go.mod ]; then go mod download; fi

# Rust
if [ -f Cargo.toml ]; then cargo fetch; fi
```

2. Verify the toolchain works:
```bash
# Run the project's type checker or build to confirm setup
# e.g., npx tsc --noEmit, go build ./..., cargo check
```

If dependency installation fails, report FAILED immediately — do not attempt
to implement without a working environment.

### Phase 1: Research (if applicable)
- Investigate the problem space for your workstream
- Read existing code in the areas you'll modify
- Identify patterns, conventions, and constraints
- Check for existing utilities or components you can reuse
- Read CLAUDE.md if present for project conventions
- Output: mental model of approach (no artifact needed — this feeds your planning)

Skip research if: the workstream scope is already well-defined and implementation
path is obvious (e.g., "add tests for X", "create migration for Y").

### Phase 2: Plan (if applicable)
- Design your implementation approach
- Break your workstream into ordered tasks
- Identify files to create/modify
- Consider edge cases and error handling
- Plan your test strategy

Skip planning if: the workstream is a single focused task (< 3 files).

### Phase 3: Implement
- Implement all tasks for your workstream
- Follow existing project conventions
- Write clean, production-quality code — not stubs or TODOs
- Create/update tests alongside implementation

**MANDATORY: Commit after every logical unit of work.** A logical unit is:
  - A new file or module
  - A completed function/class with its tests
  - A configuration change
  - Any point where `git diff --stat` shows 3+ files changed

After each unit, run:
```bash
git add -A && git commit -m "{workstream}: {what this unit does}"
```

Do NOT batch all work into one commit at the end. Do NOT leave files uncommitted.

### Phase 3.5: Commit Gate (MANDATORY)
Before proceeding to verification, ensure ALL work is committed:
```bash
git status
```
If there are ANY uncommitted changes (staged or unstaged), commit them:
```bash
git add -A && git commit -m "{workstream}: complete implementation"
```
**An agent that finishes with uncommitted changes has FAILED its workstream,
even if the code is correct.**

### Phase 4: Verify
- Run type checking (if applicable): look for tsconfig.json → tsc --noEmit
- Run tests: find and execute relevant test files
- Run lint (if applicable): look for .eslintrc, biome.json, etc.
- Fix any issues found (max 3 attempts per issue)
- If issues persist after 3 attempts, report them — don't suppress

### Phase 5: Self-Review
- Review your own changes with a critical eye
- Check for: security issues, missing error handling, untested paths
- Verify acceptance criteria are met
- Clean up any debug code, console.logs, commented-out code

## Report Format
When complete, output this report:

---
WORKSTREAM REPORT: {workstream name}
Pipeline phases completed: {list}
Status: COMPLETE / PARTIAL / FAILED

Research summary: {1-2 sentences or "skipped"}
Plan summary: {key decisions or "skipped"}

Implementation:
  Files created: {list}
  Files modified: {list}
  Tests added: {count}

Verification:
  Type check: PASS/FAIL/N/A
  Tests: PASS/FAIL ({passing}/{total})
  Lint: PASS/FAIL/N/A

Self-review:
  Acceptance criteria met: {YES/PARTIAL/NO}
  Unresolved items: {list or "none"}
  Integration notes: {anything the orchestrator should know for merge}

Git status:
  Total commits on branch: {output of `git rev-list --count HEAD ^$(git merge-base HEAD main)`}
  Clean worktree: {YES/NO — run `git status --porcelain`, YES if empty}
  Last commit: {output of `git log --oneline -1`}

**IMPORTANT**: Before writing this report, run `git status`. If there are ANY
uncommitted changes, commit them NOW with `git add -A && git commit -m "{workstream}: final changes"`.
Then fill in the Git status section. If "Clean worktree" is NO, you have failed.
---
```

#### 2c. Pipeline Depth Per Workstream

Not every workstream needs all 5 phases. The orchestrator determines depth:

| Workstream Type | Pipeline |
|----------------|----------|
| New feature (complex) | research → plan → implement → verify → review |
| New feature (clear scope) | plan → implement → verify → review |
| Bug fix | implement → verify → review |
| Tests only | implement → verify |
| Migration / config | implement → verify |
| Integration layer | plan → implement → verify → review |

Include this guidance in each agent's prompt so it knows which phases to run.

#### 2d. Collect Results

After all agents in the round complete:
- Collect branch name and worktree path from each
- Parse workstream reports
- Log results to state.json

**If an agent failed**:
1. Log the failure with its report
2. Check if it blocks subsequent rounds
3. If blocking: retry once with error context in the prompt
4. If still failing: stop and report to user with options:
   - Retry the workstream
   - Skip it and continue
   - Abort orchestration

### Phase 3: Review Each Branch

After each round completes, review all branches from that round before merging.
Read `{SKILL_DIR}/references/review-loop.md` for the full protocol.

#### 3a. Launch Review Agents (parallel)

Launch a review agent for EACH branch in the round simultaneously:

```
Agent({
  description: "Review: {workstream name}",
  subagent_type: "rpi:code-reviewer",
  prompt: "Review the complete implementation of workstream: {workstream name}

  Context: This workstream was developed in isolation as part of a parallel
  orchestration. It ran its own research/plan/implement/verify pipeline.

  The agent's self-review notes: {paste from workstream report}

  Review all changes on this branch. Focus on:
  1. Correctness — does it meet the acceptance criteria?
  2. Security — no vulnerabilities, proper auth, no secrets
  3. Conventions — follows project patterns
  4. Test quality — adequate coverage, meaningful assertions
  5. Integration readiness — will this merge cleanly with the broader feature?

  Output APPROVE or REQUEST_CHANGES with specific actionable items.
  Only request changes for CRITICAL and HIGH severity items."
})
```

#### 3b. Fix Cycle (max 3 iterations per branch)

If `REQUEST_CHANGES`:
1. Launch fix agent in the worktree branch
2. Re-review
3. Repeat up to 3 times
4. After 3: merge with warning (don't block orchestration)

See `{SKILL_DIR}/references/review-loop.md` for full details.

### Phase 4: Merge Into Feature Branch

Merge reviewed branches into the feature branch.
Read `{SKILL_DIR}/references/merge-protocol.md` for full details.

#### 4a. Merge Order

1. Round 0 (foundation) is already on the feature branch
2. Round 1 branches: merge in any order (independent)
3. Round 2+ branches: merge in any order within the round
4. Always complete all merges for Round N before starting Round N+1 execution

#### 4b. Per-Branch Merge

```bash
git checkout $FEATURE_BRANCH
git merge {worktree-branch} --no-ff -m "merge: {workstream name}"
```

#### 4c. Post-Merge Verification

After EACH merge, run a quick verification to catch integration issues early:
```bash
# Adapt to project — run whatever verification the project supports
```

If verification fails: launch a fix agent, or revert and report.

#### 4d. Conflict Resolution

If conflicts occur: launch a conflict resolution agent.
See `{SKILL_DIR}/references/merge-protocol.md` for protocol.

### Phase 5: Integration Round

After all workstream branches are merged, run a final integration pass:

#### 5a. Integration Tests

If the plan includes an integration/E2E round, execute it now on the feature branch.
This is NOT in a worktree — it runs directly on the merged feature branch.

#### 5b. Full Verification

Run the project's complete verification suite:
```bash
# Adapt to project:
make verify          # if Makefile
npm test && npm run build  # if package.json
cargo test           # if Cargo.toml
```

If failures: invoke `verify-fix-loop` skill to iteratively fix.

#### 5c. Final Review (optional)

If the user wants maximum quality, invoke `pw-review` on the full feature branch
diff against the base branch. This catches cross-cutting issues that individual
workstream reviews might miss.

### Phase 6: Report & Handoff

#### 6a. Generate Orchestration Report

Write `.pipeline/$RUN_NAME/orchestration-report.md`:

```markdown
# Orchestration Report

**Run**: {RUN_NAME}
**Feature Branch**: {FEATURE_BRANCH}
**Base Branch**: {base branch}
**Input**: {source document}
**Started**: {timestamp}
**Completed**: {timestamp}
**Total workstreams**: {count}
**Max concurrent agents**: {count}

## Workstream Results

### Round 0 — Foundation
| Workstream | Pipeline | Status | Duration |
|-----------|----------|--------|----------|
| {name} | implement → verify | COMPLETE | {time} |

### Round 1 — Core Features
| Workstream | Pipeline | Status | Review | Merged | Duration |
|-----------|----------|--------|--------|--------|----------|
| {name} | full | COMPLETE | APPROVED | clean | {time} |
| {name} | full | COMPLETE | APPROVED (iter 2) | clean | {time} |
| {name} | full | COMPLETE | APPROVED | 1 conflict resolved | {time} |

### Round 2 — Integration
...

## Pipeline Details Per Workstream

### {Workstream Name}
- **Research**: {summary or skipped}
- **Plan**: {key decisions or skipped}
- **Implementation**: {files created/modified count}
- **Verification**: TSC {P/F} | Tests {P/F} ({n}) | Lint {P/F}
- **Review**: {approved iteration, outstanding items if any}
- **Merge**: {clean / conflicts resolved}

### {Workstream Name}
...

## Review Log
{consolidated from all workstream reviews}

## Merge Log
{sequential merge history}

## Final Integration
- Integration tests: PASS/FAIL
- Full verification: PASS/FAIL
- Final review: {if run — verdict}

## Files Changed (total across all workstreams)
{deduplicated list}

## Outstanding Items
{any warnings, unresolved review items, known gaps}

## Next Steps
- Review the feature branch: `git log --oneline {base}..{feature}`
- Full diff: `git diff {base}...{feature}`
- Create PR: `gh pr create`
- Merge to {base}: `git checkout {base} && git merge {feature} --no-ff`
```

#### 6b. Present to User

Display the report summary. Ask:
- Create a PR from the feature branch?
- Run `pw-review` on the complete diff?
- Make adjustments to specific workstreams?
- Merge to the base branch?

## Auto Mode

When invoked with `auto` argument:
- Skip the orchestration plan confirmation
- Skip the final "what next?" prompt
- Run everything automatically
- Only stop on unrecoverable failures

## Quality Tiers

The user can specify a quality tier to control pipeline depth:

### `/orchestrate fast {input}`
Each agent runs: implement → verify
Skip: research, planning, review
Use for: well-defined tasks, prototyping, time pressure

### `/orchestrate standard {input}` (default)
Each agent runs: full pipeline per workstream type (see Phase 2c table)
Includes: review loop with fix cycle
Use for: production work

### `/orchestrate thorough {input}`
Each agent runs: full pipeline for ALL workstreams (no skipping)
Adds: `pw-review` on the complete feature branch at the end
Adds: second review pass after integration
Use for: critical features, high-stakes releases

## Error Handling

### Workstream Agent Failure
1. First failure: retry once with error context
2. Second failure: mark workstream as failed, check dependencies
3. If blocking: stop round, report to user
4. If non-blocking: continue, note the gap in the report

### Review Loop Exhaustion (3 iterations)
- Merge with warning flag
- Outstanding items listed in report
- Final `pw-review` (if thorough tier) will catch critical issues

### Merge Conflict
1. Auto-resolve with conflict resolution agent
2. If unresolvable: stop, ask user
3. Never pick one side blindly

### Integration Failure After Merge
1. Identify which merge introduced the regression
2. Launch targeted fix agent
3. If unfixable: offer to revert the problematic merge

### Orchestration-Level Failure
If multiple workstreams fail or the feature branch is in a bad state:
1. Save full state to state.json
2. Report everything that succeeded and failed
3. The user can fix manually and run `/orchestrate resume` to continue

## Resume Support

`/orchestrate resume` reads the most recent `.pipeline/orchestrate-*/state.json`:
- Skips completed rounds
- Re-executes failed workstreams
- Continues from the last successful merge point

## State Tracking

State tracking serves two purposes: (1) enabling `/orchestrate resume` and
(2) providing context if something goes wrong.

### What to Track (state.json)

Update `.pipeline/$RUN_NAME/state.json` at these moments ONLY:
1. **After decomposition** — write the initial state with all workstreams and rounds
2. **After each round completes** — mark the round done, record pass/fail per workstream
3. **After each merge** — record merge result (clean/conflicts/failed)
4. **On failure** — record what failed and why

Each workstream gets a flat entry within its round:
```json
{
  "name": "auth-service",
  "status": "done|failed",
  "branch": "worktree/auth-service",
  "review": "approved|merged-with-warning",
  "merge": "clean|conflicts-resolved|failed"
}
```

### What NOT to Track in state.json

- Pipeline phase transitions within agents (the agent reports capture this)
- Timestamps for every event (the git log has these)
- Full review logs (these go in review-log.md)
- File lists (the orchestration-report.md captures these at the end)

The detailed orchestration report (Phase 6) is generated from agent reports
and git log at the end — it does NOT depend on state.json having every detail.

## Notes

- Each worktree agent is fully isolated — cannot interfere with others
- The feature branch is the single integration point
- Reviews happen BEFORE merge, not after
- Round N+1 agents get the merged state of all Round N work
- The orchestrator (you) manages the feature branch and coordinates everything
- Foundation work (Round 0) happens on the feature branch directly
- Worktree cleanup is automatic when agents finish
- State files enable resume across sessions
