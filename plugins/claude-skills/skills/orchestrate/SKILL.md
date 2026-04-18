---
name: orchestrate
description: |
  Full-pipeline parallel orchestration. Parse a roadmap, spec, or plan into independent
  workstreams, spawn isolated worktree agents that each run the complete /workflow
  pipeline (research → deliberate → implement → verify → review), then merge all
  results into a single feature branch with a proof-of-acceptance report. Use when:
  (1) A roadmap, spec, or plan has multiple independent features/modules
  (2) User says "orchestrate", "parallel build", "build the roadmap"
  (3) User wants high-quality parallel implementation with full pipeline per workstream
  (4) Multiple features can be developed independently then integrated
  Each worktree agent gets the full quality pipeline, not just implementation.
---

# Full-Pipeline Parallel Orchestration

Each workstream gets its own worktree agent running the complete pipeline.
The orchestrator manages parallelism, integration, merge safety, and produces
a proof-of-acceptance report the user can trust.

This SKILL.md is the phase-level index. Deep detail lives in
`references/phase-*.md` and is loaded only when a phase executes.

## Input

Accept one of:
1. A roadmap, feature spec, or PRD (`.md` file)
2. A `plan.md` from `pw-deliberate` or `/plan`
3. Inline description of multiple workstreams
4. Auto-detect: most recent `.pipeline/*/plan.md`

## Invocation

```
/orchestrate {input}                 # standard quality tier
/orchestrate fast {input}            # minimal pipeline, no review
/orchestrate thorough {input}        # full pipeline + final pw-review
/orchestrate auto {input}            # skip confirmation prompts
/orchestrate --dry-run {input}       # show the plan, launch nothing
/orchestrate resume                  # continue from last state.json
```

## Setup

```bash
RUN_NAME="orchestrate-$(date +%Y%m%d-%H%M%S)"
mkdir -p .pipeline/$RUN_NAME
FEATURE_BRANCH="orchestrate/$RUN_NAME"
git checkout -b "$FEATURE_BRANCH"
```

Initialize `.pipeline/$RUN_NAME/state.json`:
```json
{
  "run_name": "{RUN_NAME}",
  "feature_branch": "{FEATURE_BRANCH}",
  "base_branch": "{branch we started from}",
  "status": "running",
  "current_round": 0,
  "rounds": [],
  "ac_map": {},
  "error": null
}
```

## Phase Sequence

The full run executes these phases. Each links to a reference file loaded
on demand.

| # | Phase | Load | Purpose |
|---|-------|------|---------|
| 1 | Decompose | [phase-1-decompose.md](references/phase-1-decompose.md) | Extract workstreams, scope globs, AC→workstream map |
| — | Dry-run | [dry-run.md](references/dry-run.md) | If `--dry-run`, render plan and STOP |
| 2 | Spawn | [phase-2-spawn.md](references/phase-2-spawn.md) | Launch parallel worktree agents |
| — | Retry | [retry-semantics.md](references/retry-semantics.md) | Verify-gate 3-axis retry |
| 3 | Review | [phase-3-review.md](references/phase-3-review.md) | Parallel review agents per branch |
| 3.5 | Merge Safety | [phase-3.5-merge-safety.md](references/phase-3.5-merge-safety.md) | Recovery/quarantine checklist before merge |
| 4 | Merge + Final Verify | [phase-4-verify-final.md](references/phase-4-verify-final.md) | Merge non-quarantined branches, run `verify.final` |
| 5 | Acceptance Report | [phase-5-report.md](references/phase-5-report.md) | Proof artifact with "Next Actions for Human" at top |

Supporting references (loaded when relevant):
- [interface-contracts.md](references/interface-contracts.md) — producer/consumer shared shapes
- [merge-protocol.md](references/merge-protocol.md) — merge steps, conflict resolution, revert
- [review-loop.md](references/review-loop.md) — review/fix iteration protocol
- [self-test.md](references/self-test.md) — fixture-based self-test (pre-release check)

## Non-Negotiable Safety Rules

These apply everywhere. Read before loading any phase reference.

### 1. Always operate from main repo root when merging or cleaning up worktrees

```bash
MAIN_ROOT=$(git worktree list --porcelain | awk '/^worktree / {print $2; exit}')
cd "$MAIN_ROOT"
```
Never `cd` into a worktree you are about to merge or delete.

### 2. Dirty worktree at merge time → RECOVER + QUARANTINE, do NOT merge

- Commit leftover work with `[recovery]` prefix on the worktree branch
- Mark workstream `quarantined: true`, `merge_skipped: true`
- Surface in Next Actions at the top of the acceptance report
- Continue with other non-quarantined workstreams — do not halt the run

Full protocol: [phase-3.5-merge-safety.md § 3.5b](references/phase-3.5-merge-safety.md).

### 3. Every workstream must pass the verify gate before claiming complete

Gate = typecheck + tests + lint. Up to 3 retries, each using a DIFFERENT
axis: feedback-loop → reduced-scope → fresh-agent. No blind re-runs.

Full protocol: [retry-semantics.md](references/retry-semantics.md).

### 4. Scope is enforced at merge time

Every workstream has `scope_globs` declared in Phase 1. Phase 3.5c diffs
actual changed files against those globs and classifies anything outside as
declared-spillover, creep, or foundation-overlap.

### 5. Acceptance requires evidence

Phase 5 produces `acceptance-report.md` with one row per original AC,
stating PASS/FAIL/MANUAL with the exact validating command and output
excerpt. No evidence → AC is marked not-verified, report status is not
green.

## `verify.final` Hook

The final verification stage runs project-configurable checks against the
merged feature branch. Configure in roadmap YAML frontmatter:

```yaml
verify:
  final:
    commands:
      - { name: "typecheck", cmd: "npm run typecheck" }
      - { name: "test",      cmd: "npm test -- --run" }
      - { name: "lint",      cmd: "npm run lint" }
    smoke:
      type: "playwright"    # playwright | http | cli | none
      spec: "tests/smoke/login.spec.ts"
```

Auto-detection defaults apply if absent. Full spec and detection rules:
[phase-4-verify-final.md](references/phase-4-verify-final.md).

## State File

`.pipeline/$RUN_NAME/state.json` is the single source of truth for resume
and reporting. Update it at these moments only:

1. After decomposition — initial shape with workstreams, rounds, `ac_map`
2. After each round completes — pass/fail per workstream, verify attempts
3. After each merge attempt — clean / conflicts / quarantined / skipped
4. On failure — what failed and why

Do NOT track every pipeline transition or timestamp — the agent reports and
git log are the authoritative record for those. State.json is for resume
and top-level reporting.

## Quality Tiers

| Tier | Pipeline per workstream | Final pass |
|------|------------------------|------------|
| `fast` | implement → verify | `verify.final` only |
| `standard` (default) | per-type depth + review (see phase-2) | `verify.final` + acceptance report |
| `thorough` | full pipeline for every workstream + `pw-review` on final diff | `verify.final` + second review pass + acceptance report |

## Error Handling

| Situation | Action |
|-----------|--------|
| Workstream agent fails | Retry once with error context; if still failing, mark FAILED and exclude from merge |
| Verify gate fails all 3 retries | Mark workstream FAILED, record all 3 failure outputs, exclude from merge |
| Dirty worktree at merge | Recover + quarantine (see safety rule 2) |
| Scope creep | Revert offending files or halt merge for that branch |
| Foundation overlap | Halt merge for the later branch; surface for user to coordinate |
| Merge conflict | Conflict-resolution agent (see [merge-protocol.md](references/merge-protocol.md)) |
| `verify.final` red | Invoke `verify-fix-loop`; cap at 2 fix cycles; every fix commit reviewed |
| Orchestration-level failure | Save state, report what succeeded, user can `/orchestrate resume` |

## Resume

`/orchestrate resume` reads the most recent `.pipeline/orchestrate-*/state.json`:
- Skips completed rounds
- Re-executes failed / quarantined workstreams (user can opt quarantined back in)
- Continues from last successful merge point
- Regenerates the acceptance report from current state

## Notes

- Each worktree agent is fully isolated — cannot interfere with others
- The feature branch is the single integration point; only non-quarantined
  branches merge into it
- Reviews happen BEFORE merge, not after
- Round N+1 agents get the merged state of all Round N work
- Foundation (Round 0) happens on the feature branch directly, no worktree
- Worktree cleanup is automatic for non-quarantined branches; quarantined
  branches persist until the user reviews and decides
- `state.json` survives across sessions; resume works across machine reboots
