---
name: orchestrate
description: |
  Full-pipeline parallel orchestration. Decompose a spec into workstreams,
  drive each to GREEN via per-criterion TDD in isolated worktrees, merge,
  PO-accept, report. Use when:
  (1) A roadmap/spec/plan has multiple independent features
  (2) User says "orchestrate", "parallel build", "build the roadmap"
  (3) You want the whole pipeline — decompose → TDD → merge → PO accept
  (4) Features can be developed independently then integrated
  The orchestrator is a PURE COORDINATOR: it delegates every code change,
  test run, and commit to subagents or sub-skills. It never writes code.
---

# Orchestrate

Thin coordinator. Every step is `Skill(...)` or `Agent(...)`.

## Delegation Contract (non-negotiable)

**MUST:** invoke sub-skills (`Skill`), spawn subagents (`Agent`), run
read-only bash only (`git status|log|diff|worktree list`, `jq`, `ls`),
write only `state.json` + route-through artifacts.

**MUST NOT:** `Edit`/`Write`/`NotebookEdit` source code; run build, test,
lint, install, or commit bash; run `git merge|commit|add` on source
changes (merge-coordinator does it).

If a sub-skill tells you to edit code, re-read it — it tells you to spawn
an agent that edits code.

## Input

Priority: roadmap/spec/PRD `.md` · `plan.md` from `pw-deliberate` · inline
description · auto-detect most recent `.pipeline/*/plan.md`.

## Invocation

```
/orchestrate {input}           # standard tier
/orchestrate fast {input}      # TDD loop only, no PO, no review
/orchestrate thorough {input}  # standard + pw-review on final diff
/orchestrate auto {input}      # skip confirmation prompts
/orchestrate --dry-run {input} # render plan, stop
/orchestrate resume            # continue from last state.json (schema v2)
```

## Setup

```bash
RUN_NAME="orchestrate-$(date +%Y%m%d-%H%M%S)"
mkdir -p .pipeline/$RUN_NAME
FEATURE_BRANCH="orchestrate/$RUN_NAME"
git checkout -b "$FEATURE_BRANCH"
```

Initialize `state.json` with `schema_version: 2`, `run_name`,
`feature_branch`, `base_branch`, `status: "running"`, `current_round: 0`,
`rounds: []`, `ac_map: {}`, `showstopper_rounds_used: 0`.

## Phase Flow

| # | Step | Call | Ref |
|---|------|------|-----|
| 1 | Decompose | `Skill(extract-criteria)` | — |
| — | Dry-run | Stop if `--dry-run` | [dry-run.md](references/dry-run.md) |
| 2 | Parallel TDD (per round) | `Skill(parallel-workstreams)` | — |
| 3 | Merge + verify (per round) | Spawn merge-coordinator `Agent` | [phase-merge-safety.md](references/phase-merge-safety.md) |
| 4 | PO acceptance | `Skill(po-acceptance)` | — |
| 5 | Route verdict (inline) | — | — |
| 6 | Report | Spawn reporter `Agent` | [phase-report.md](references/phase-report.md) |

On-demand refs: [interface-contracts.md](references/interface-contracts.md),
[retry-semantics.md](references/retry-semantics.md).

## Step 5 — Verdict Routing

```
APPROVED           → step 6
FAST_FOLLOWS_ONLY  → step 6  (fast-follows.md already written)
SHOWSTOPPERS:
  state.showstopper_rounds_used += 1
  if used >= 2: state.status = "showstopper_unresolved"; → step 6
  else:         merge verdict.updated_criteria into criteria; → step 2
```

Cap rationale: PO finding the same failure class twice means the criteria
themselves are wrong — a human must rewrite them.

## Non-Negotiable Safety Rules

1. **Main repo root for merges.** Merge-coordinator `cd`s to main worktree first.
2. **Dirty worktree at merge → recover + quarantine, don't merge.** Quarantined branches surface in "Next Actions for Human".
3. **Verify gate per-criterion** (inside `tdd-cycle`, 3 axis-distinct retries).
4. **Scope enforced at merge**; deviations classified in `phase-merge-safety.md`.
5. **Evidence required** — every criterion has a runnable `check`.
6. **Showstopper loop capped at 2 rounds.**

## Quality Tiers

| Tier | Self-review | Review agent | PO | Report |
|------|-------------|--------------|----|--------|
| fast | no | no | no | minimal |
| standard | yes | after merge | yes | full |
| thorough | yes | after merge + `pw-review` on final diff | yes | full |

TDD loop runs in every tier.

## `verify.final` Hook

Configure in roadmap YAML frontmatter (auto-detected otherwise — full spec
in `phase-merge-safety.md`):

```yaml
verify:
  final:
    commands: [ { name: typecheck, cmd: "..." }, { name: test, cmd: "..." }, { name: lint, cmd: "..." } ]
    smoke:    { type: "playwright|http|cli|none", spec: "..." }
```

## State & Resume

`state.json` is the source of truth. Update after decompose, each round,
each merge, each PO verdict, any failure. `schema_version: 2`; resume on
pre-v2 emits "schema v1 detected — re-run required" and stops.

`/orchestrate resume` skips completed rounds, re-spawns
failed/quarantined workstreams on opt-in, re-runs PO, regenerates report.

## Error Handling

| Situation | Action |
|-----------|--------|
| Workstream agent fails | `parallel-workstreams` retries once; else exclude from merge |
| `tdd-cycle` 3 retries fail | Criterion FAILED, reported |
| Dirty worktree at merge | Rule 2 |
| Scope creep / foundation overlap | Merge-coordinator reverts or halts branch |
| Merge conflict | Merge-coordinator spawns resolver |
| `verify.final` red | Spawn `verify-fix-loop`; cap 2 cycles |
| 2nd showstopper round | Halt with `SHOWSTOPPER_UNRESOLVED` report |
| Orchestration failure | Save state; `/orchestrate resume` |

## Migration from 1.3.x

Removed: `phase-1-decompose.md`, `phase-2-spawn.md`, `phase-3-review.md`,
`review-loop.md`, `merge-protocol.md`, `phase-4-verify-final.md`. Logic
moved to standalone skills `extract-criteria` / `parallel-workstreams` /
`tdd-cycle` / `po-acceptance` and consolidated `phase-merge-safety.md`.
User-facing invocation unchanged.
