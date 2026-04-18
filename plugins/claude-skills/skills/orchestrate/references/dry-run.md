# `--dry-run` Mode

Load this reference when `/orchestrate` is invoked with `--dry-run`.

Dry-run renders what WOULD happen without launching any agents, creating any
branches, or mutating git state. Its purpose is a single gate: "yes, launch
it" or "no, this plan is wrong."

## What Dry-Run Does

1. Read the input (roadmap / plan / inline description).
2. Run Phase 1 decomposition fully — workstreams, scope globs, dependencies,
   round schedule, AC→workstream map.
3. Detect the current repo state (branch, dirty files, existing worktrees).
4. Run the scope-glob check against the ALREADY-CHANGED files in the working
   tree (preview of conflicts with in-flight work).
5. Detect `verify.final` configuration (or show the auto-detected default).
6. Print the Dry-Run Report to stdout.
7. Exit 0 without writing anything except `.pipeline/$RUN_NAME/dry-run.md`
   (no state.json, no feature branch, no worktrees).

## What Dry-Run Does NOT Do

- Launch any subagent (workstream, review, verify, or fix).
- Create, delete, or switch branches.
- Create worktrees.
- Run tests, lint, typecheck, or smoke.
- Mutate `state.json` for a non-dry-run run.
- Install dependencies.

## Dry-Run Report (MVP content)

```markdown
# Dry-Run Plan — {RUN_NAME_PROPOSED}

## Input
Source: {roadmap path / inline}
Title: {title}

## Current Repo State
- Branch: {current branch}
- Clean: {YES/NO}
- Uncommitted files: {list if any}
- Existing worktrees: {list if any}
- Would create feature branch: `orchestrate/{RUN_NAME}`
- Would branch from: {current branch}

## Workstream Decomposition ({N} workstreams)

### Round 0 — Foundation
| Workstream | Scope globs | AC count | Dependencies |
|-----------|-------------|----------|--------------|
| shared-types | src/types/**, migrations/** | 0 | — |

### Round 1 — Parallel ({N} agents)
| Workstream | Scope globs | AC count | Pipeline depth |
|-----------|-------------|----------|----------------|
| auth-service | src/auth/**, tests/auth/** | 3 | full |
| video-api | src/video/**, tests/video/** | 5 | full |
| clip-editor | src/components/clip-editor/** | 4 | full |

### Round 2 — Parallel ({N} agents)
...

## Acceptance Criteria Map
All ACs from the roadmap mapped to owning workstreams:

| AC ID | AC text | Workstream | Check |
|-------|---------|-----------|-------|
| AC1 | User can log in | auth-service | npm test -- login |
| AC2 | Upload transcodes video | video-api | smoke: playwright upload |
| AC3 | Clip edit saves | clip-editor | npm test -- clip-edit |
| AC4 | Sessions expire | auth-service | npm test -- session-expiry |

**Unmapped ACs**: {list or "none"}. Unmapped ACs are a planning failure —
fix the roadmap or the workstream descriptions before launching.

## Detected Conflicts with Current Working Tree

{For each uncommitted file in the current branch, check if any workstream's
scope globs would touch it:}

| Uncommitted file | Conflicting workstream | Recommendation |
|------------------|----------------------|----------------|
| src/auth/login.ts | auth-service | commit or stash before launching |

{If none:} No conflicts — clean to launch.

## Detected `verify.final` Configuration

Source: {frontmatter / auto-detected default / none}

```yaml
verify:
  final:
    commands: [ ... ]
    smoke: { type: ..., ... }
```

{If none detected:} ⚠ No verify.final configured. Acceptance cannot be proven
automatically. Add a `verify.final` block to the roadmap frontmatter before
launching, or accept that ACs will be marked `manual-required`.

## Launch Parameters
- Max concurrent agents: {count, based on largest round}
- Estimated total agents: {count across all rounds, including reviews}
- Worktrees to be created: {list}
- Quality tier: {fast|standard|thorough}

## Ready to Launch?

Run without `--dry-run` to execute this plan:

    /orchestrate {original args without --dry-run}

To adjust the plan: edit the roadmap's workstream decomposition, scope globs,
or acceptance criteria, then re-run `--dry-run` until satisfied.
```

## Flags That Modify Dry-Run

- `--dry-run --verbose`: additionally prints the full agent prompt that
  would be sent to each workstream, with `{contract}` substitutions resolved.
  Useful for debugging the spawn phase.
- `--dry-run --json`: emit the dry-run plan as JSON on stdout instead of
  markdown. For programmatic consumers (CI, other orchestrators).

## What Dry-Run Is Good For

- Confirming the roadmap decomposes the way you expect.
- Catching unmapped ACs before burning agent time.
- Detecting scope-glob conflicts between workstreams before they become merge
  conflicts.
- Previewing `verify.final` — did auto-detection pick the right commands?
- Spotting conflicts with uncommitted local work.

## What Dry-Run Is NOT Good For

- Estimating real agent duration (no actual runs happen).
- Catching runtime failures (no code is executed).
- Validating that the scope_globs will actually match what the agent writes
  (that's Phase 3.5c at merge time).
