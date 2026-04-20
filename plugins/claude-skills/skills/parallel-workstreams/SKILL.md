---
name: parallel-workstreams
description: |
  Launch N independent workstreams as parallel worktree agents — each agent
  runs its own `tdd-cycle` per criterion inside an isolated git worktree.
  Use when:
  (1) You have ≥2 independent workstreams with scope_globs and criteria
  (2) User says "run these in parallel", "parallel build", "spawn worktrees"
  (3) Inside `/orchestrate`, called after `extract-criteria` per round
  (4) You want true concurrency (single message, multiple Agent calls)
  Caller never creates worktrees, writes code, or runs tests directly — every
  step is a spawned subagent. Supports producer/consumer contracts and a
  foundation (Round 0) step executed sequentially on the feature branch.
---

# Parallel Workstreams

Launch one worktree agent per workstream in a single message. Each agent
drives itself to GREEN using `tdd-cycle` per success criterion, then reports.

## Input

```
{
  "round": 1,
  "feature_branch": "orchestrate/{run}",
  "run_name": "{run}",
  "workstreams": [
    {
      "name": "auth-service",
      "description": "...",
      "scope_globs": ["src/auth/**", "tests/auth/**"],
      "success_criteria": [ ... ],
      "acceptance_criteria": [ ... ],
      "dependencies": []
    }
  ],
  "contracts": [
    { "name": "...", "producer": "...", "consumer": "...", "shape": "..." }
  ],
  "quality_tier": "fast" | "standard" | "thorough"
}
```

## 1. Foundation (Round 0 only)

If `round === 0`, the foundation workstream runs directly on the feature
branch, NOT in a worktree — Round 1 agents need its commits as their base.

Spawn ONE subagent on the feature branch:

```
Agent({
  description: "Foundation: {workstream.name}",
  prompt: <same pipeline as a worktree agent below, but with
           'you are on the feature branch, not a worktree' and no
           parallelism considerations>
})
```

After it reports GREEN, commit on the feature branch. This becomes the base
for all Round 1+ agents.

## 2. Parallel Rounds (Round ≥ 1)

Launch every workstream in the round in a **single message** with multiple
`Agent` calls. This is the only way to achieve true concurrency.

```
# In ONE message:
Agent({ description: "auth-service", isolation: "worktree", prompt: <template> })
Agent({ description: "video-api",    isolation: "worktree", prompt: <template> })
Agent({ description: "clip-editor",  isolation: "worktree", prompt: <template> })
```

### Worktree Agent Prompt Template

```
You are a workstream agent running in a git worktree isolated from other
parallel workstreams. Drive your workstream to GREEN via one-test-at-a-time
TDD.

## Workstream
  name: {name}
  description: {description}
  scope_globs: {scope_globs}
  success_criteria: {JSON}
  acceptance_criteria: {JSON}

## Broader Context
  run: {run_name}
  other parallel workstreams (do not touch their scope): {names only}

## Interface Contracts (if applicable)
  {For each contract this workstream participates in:
     contract name
     your role: PRODUCER or CONSUMER
     shape: {exact shape}
   You MUST match the shape exactly. Note any discovered mismatch in your
   report under 'contract_deviations'.}

## Phase 0 — Environment Setup (MANDATORY)

Your worktree is a fresh checkout; dependencies are NOT pre-installed.

Detect and install:
  pnpm-lock.yaml → pnpm install
  yarn.lock      → yarn install
  package.json   → npm install
  requirements.txt → pip install -r requirements.txt
  pyproject.toml → pip install -e .
  go.mod         → go mod download
  Cargo.toml     → cargo fetch

Verify the toolchain works (tsc --noEmit, go build, cargo check, ...).
If setup fails, report FAILED — do NOT proceed.

## Phase 1 — TDD Loop

For each success_criterion in order:
  Invoke Skill(tdd-cycle) with:
    {
      "criterion": <the criterion>,
      "workstream": "{name}",
      "scope_globs": {scope_globs},
      "context": "{summary of previously-completed criteria}"
    }
  Wait for CYCLE_REPORT.
  If status == FAILED, STOP the loop and advance to Phase 2 with
  partial state. Do not proceed to later criteria on a failed one.

## Phase 2 — Commit Gate (MANDATORY)

Before reporting, confirm `git status --porcelain` is empty. Every cycle
already commits test + impl; the worktree should be clean.

If non-empty: commit leftover work as `chore: tidy-up` and note in your
report under 'uncommitted_at_end'. Non-empty status here is a defect — the
orchestrator may quarantine this branch (the recovery path preserves your
work but blocks the merge).

## Phase 3 — Self-Review (quality_tier: standard or thorough only)

Walk each completed criterion and check:
  - Any debug code / console.log / TODO left behind?
  - Any `any` / `@ts-ignore` / `eslint-disable` inserted?
  - Every criterion's check command still passes?

Fix in place and re-run verify if anything is flagged.

## Report

WORKSTREAM_REPORT
  name: {name}
  branch: {worktree branch name}
  worktree_path: {absolute path}
  status: COMPLETE | PARTIAL | FAILED
  criteria:
    - id, status (DONE|FAILED), test_commit, impl_commit, verify_outcome
  uncommitted_at_end: YES | NO
  contract_deviations: [...]
  scope_deviations: [...] (files touched outside scope_globs with reason)
  first_commit: {sha}
  last_commit: {sha}
  total_commits: {int}
```

## 3. Pipeline Depth Per Workstream Type

The caller of `parallel-workstreams` selects pipeline depth per workstream
by picking `quality_tier`. Within a worktree, the TDD loop is the same;
these adjust the before/after phases:

| Workstream Type          | Research | Plan | TDD loop | Self-review | Review agent |
|-------------------------|----------|------|----------|-------------|--------------|
| New feature (complex)    | yes      | yes  | ✓        | ✓           | after        |
| New feature (clear scope)| no       | yes  | ✓        | ✓           | after        |
| Bug fix                  | no       | no   | ✓        | ✓           | after        |
| Tests only               | no       | no   | ✓        | no          | no           |
| Migration / config       | no       | no   | ✓        | no          | no           |
| Integration layer        | no       | yes  | ✓        | ✓           | after        |

If research or plan steps are required for a type, prepend short subagent
calls before the TDD loop. Keep them terse — their job is to inform the
TDD loop, not re-derive acceptance criteria.

## 4. Collect Results

After all agents in a round complete (parallel `Agent` returns), the
caller:
1. Reads each WORKSTREAM_REPORT.
2. Records per-workstream outcome in `state.json` under
   `rounds[n].workstreams[w]`.
3. Advances to the next round (its base is the merged state of this round,
   handled by the merge-safety phase of the caller).

## 5. Failure Handling

| Situation | Action |
|-----------|--------|
| Agent reports FAILED | Record. Retry ONCE with failure summary appended to prompt. If it fails again, mark workstream FAILED and exclude from merge. |
| Agent reports PARTIAL (some criteria DONE, some FAILED) | Record. Do NOT retry automatically — the caller decides whether partial merge is acceptable. |
| Agent returns no report | Treat as FAILED; worktree likely has work to quarantine at merge time. |

## Standalone use

`/parallel-workstreams` with a criteria.json (from `extract-criteria`)
launches one round. It does not merge — merging is the caller's
responsibility (or the orchestrator's).

## Notes

- Each worktree is fully isolated; agents cannot interfere.
- Round N+1 agents get the merged state of Round N after the caller runs
  its merge phase.
- `isolation: "worktree"` cleans up the worktree on non-FAILED returns if
  the caller does not commit to its branch — the caller typically merges
  the branch first to keep history.
