---
name: pw-implement
description: |
  Checkpoint-gated parallel implementation. Execute a plan where each agent must pass
  tsc + tests before reporting done. Use when:
  (1) A plan.md exists and is ready to implement
  (2) User says "implement", "build it", "execute the plan"
  (3) After pw-deliberate produces a plan
  Each checkpoint runs parallel agents with independent verification gates.
---

# Checkpoint-Gated Parallel Implementation

Execute a checkpoint-driven plan where each agent independently verifies their work.

## Input

Accept one of:
1. A path to `plan.md` (from `pw-deliberate`)
2. A plan inlined from the user
3. Auto-detect: look for the most recent `.pipeline/*/plan.md`

## Setup

1. Create or reuse run directory:
   ```bash
   mkdir -p .pipeline/{run-name}
   ```
2. Read the plan and parse checkpoints
3. Read `{SKILL_DIR}/references/agent-protocol.md` for the verification protocol

## Workflow

### 1. Parse Plan

Extract from plan.md:
- All checkpoints in order
- Per checkpoint: tasks, files, success criteria, verification command, parallelization notes
- Feature flag requirements

### 2. Execute Checkpoints Sequentially

For each checkpoint (in order):

#### 2a. Identify Parallel Groups

Based on the plan's parallelization notes:
- Tasks touching different files → parallel agents
- Tasks touching the same file → single agent, sequential
- Dependencies (types before implementations) → sequential ordering

#### 2b. Launch Parallel Implementation Agents

Launch agents simultaneously using the Agent tool. Each agent receives:

**Agent prompt template**:
```
You are implementing part of Checkpoint {N}: {name}.

## Your Tasks
{list of specific tasks for this agent}

## Files You Own
{list of files this agent will create/modify}

## Context
- Project conventions: Read CLAUDE.md at the project root
- Plan context: {relevant plan excerpt}
- Feature flag: {if applicable}

## Verification Protocol (MANDATORY)
After implementing your tasks, you MUST run these verification steps:

1. Run TypeScript check:
   ```bash
   cd app && npx tsc --noEmit
   ```
   If errors in YOUR files: fix them. If errors in OTHER files: report but don't touch.

2. Run relevant tests:
   ```bash
   cd app && npx vitest run {test file pattern}
   ```
   If tests fail: fix them (max 3 attempts). If still failing after 3 attempts: report the failure.

3. Check for prohibited patterns in your files:
   - No `// @ts-ignore` or `// @ts-expect-error`
   - No `eslint-disable` comments
   - No `any` type to silence TypeScript
   - No `.skip()` on tests
   - No empty catch blocks

Report your results in this format:
---
AGENT REPORT
Tasks: {completed/total}
TSC: PASS/FAIL ({error count if fail})
Tests: PASS/FAIL ({test count})
Prohibited patterns: CLEAN/FOUND ({list if found})
Files modified: {list}
Issues: {any problems encountered}
---
```

Each agent uses subagent_type `general-purpose` to have full file editing and bash access.

#### 2c. Collect Agent Reports

After all agents for this checkpoint complete:
1. Parse each agent's report
2. Check for failures or unresolved issues

#### 2d. Run Checkpoint Verification

Execute the checkpoint's verification command from the plan:
```bash
{verification command from plan.md}
```

#### 2e. Handle Results

**If checkpoint passes**: Mark complete, log results, proceed to next checkpoint.

**If checkpoint fails**:
1. Analyze the failure
2. Create targeted fix tasks
3. Launch fix agents (max 2 retry rounds per checkpoint)
4. Re-run verification
5. If still failing after 2 retries: stop and report to user

### 3. Final Verification

After all checkpoints complete:

```bash
cd app && make verify
```

This runs lint + tests + build. If it fails, enter the verify-fix-loop skill.

### 4. Generate Implementation Log

Write `.pipeline/{run-name}/implementation-log.md`:

```markdown
# Implementation Log: {Title}

**Date**: {timestamp}
**Plan**: {plan.md path}

## Checkpoint Results

### Checkpoint 1: {Name}
- Status: PASS/FAIL
- Agents: {count} parallel
- Duration: {time}
- Tasks completed: {n/total}
- Retries: {count}
- Files modified:
  - {file}: {description}

### Checkpoint 2: {Name}
...

## Final Verification
- TSC: PASS/FAIL
- Lint: PASS/FAIL
- Tests: PASS/FAIL ({count} passing)
- Build: PASS/FAIL

## Summary
{What was built, key decisions made during implementation}

## Issues Encountered
- {Any problems and how they were resolved}
```

### 5. Report to User

Display summary:
- Checkpoints completed
- Files modified
- Test results
- Any issues or manual steps needed

Ask if the user wants to:
- Run `pw-review` on the changes
- Commit the changes
- Make adjustments

## Checkpoint Failure Escalation

1. **Agent-level failure**: Agent retries up to 3 times
2. **Checkpoint-level failure**: Re-launch fix agents up to 2 times
3. **Persistent failure**: Stop, report to user with:
   - What failed and why
   - What was attempted
   - Suggested manual intervention

Never silently skip a failing checkpoint.

## Notes

- Each agent works in isolation on its own files — no file conflicts
- Agents must follow the verification protocol — no exceptions
- The protocol prevents "works on my machine" by requiring independent verification
- Feature flags are applied at the code level per CLAUDE.md conventions
- Estimated total time: 2-5 minutes per checkpoint depending on complexity
