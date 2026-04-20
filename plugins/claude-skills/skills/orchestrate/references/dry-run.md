# `--dry-run` Mode

Load when `/orchestrate` is invoked with `--dry-run`. Renders the plan
without launching agents or mutating git state.

## Behavior

**Does:** read the input, run `Skill(extract-criteria)` fully, detect
repo state, scope-check against currently-uncommitted files, detect
`verify.final` config, print the dry-run report, write only
`.pipeline/$RUN_NAME/dry-run.md`.

**Does not:** spawn workstream/review/verify/fix agents, create/switch
branches, create worktrees, run tests/lint/typecheck/smoke, mutate the
run's `state.json`, install dependencies.

## Report Template

```markdown
# Dry-Run Plan — {RUN_NAME_PROPOSED}

## Input
Source: {roadmap path / inline}
Title: {title}

## Current Repo State
- Branch: {branch} · Clean: {YES/NO}
- Uncommitted files: {list or "none"}
- Existing worktrees: {list or "none"}
- Would create feature branch: `orchestrate/{RUN_NAME}` from {current}

## Workstream Decomposition ({N})

### Round 0 — Foundation
| Workstream | Scope globs | SC / AC | Dependencies |

### Round 1 — Parallel ({N} agents)
| Workstream | Scope globs | SC / AC | Pipeline depth |

## Acceptance Criteria Map
| AC ID | Text | Workstream | Check |
**Unmapped ACs**: {list or "none"}. Unmapped = planning failure.

## Detected Conflicts with Working Tree
| Uncommitted file | Conflicting workstream | Recommendation |
{or "No conflicts — clean to launch."}

## Detected `verify.final`
Source: {frontmatter / auto-detected / none}
```yaml
verify.final: { ... }
```
{If none:} ⚠ no verify.final — ACs will be `manual-required`.

## Launch Parameters
Max concurrent agents, estimated total agents (including reviews),
worktrees to be created, quality tier.

## Ready to Launch?
Run without `--dry-run` to execute:

    /orchestrate {args without --dry-run}
```

## Flags

- `--dry-run --verbose` — also prints the full worktree-agent prompt with
  contracts substituted (debug the spawn phase).
- `--dry-run --json` — emit the plan as JSON (programmatic consumers).

## What it is good for

- Confirming decomposition before burning agent time.
- Catching unmapped ACs early.
- Detecting scope-glob conflicts between workstreams.
- Previewing `verify.final` auto-detection.
- Spotting conflicts with uncommitted local work.

## What it is not good for

- Estimating real agent duration (no runs happen).
- Catching runtime failures (no code executes).
- Validating scope_globs against actual edits (that's merge-safety §3).
