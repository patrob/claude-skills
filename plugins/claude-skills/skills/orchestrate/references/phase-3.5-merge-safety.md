# Phase 3.5 — Merge-Back Safety Checklist (MANDATORY)

Load this reference before merging ANY worktree branch. Every item must pass
— if any fails, the branch is blocked and the orchestration continues with
the remaining branches. Do NOT proceed to Phase 4 for a branch until all
checks pass.

Rationale: worktree merges are the highest-risk moment in orchestration.
Operating from the wrong directory, merging uncommitted work, or silently
pulling in out-of-scope changes are all recoverable only if caught before
the merge commit lands.

## 3.5a. Leave the Worktree

You MUST operate from the main repository root for the entire merge and any
subsequent worktree cleanup. NEVER run merge or worktree-removal commands
from inside the worktree being merged or deleted — the directory may be
removed out from under you and leave the repo in an inconsistent state.

```bash
# Determine the main repo root (not the worktree path)
MAIN_ROOT=$(git worktree list --porcelain | awk '/^worktree / {print $2; exit}')
cd "$MAIN_ROOT"

# Confirm
pwd                                # should be $MAIN_ROOT, not a worktree path
git rev-parse --show-toplevel      # should equal $MAIN_ROOT
git branch --show-current          # should equal $FEATURE_BRANCH
```

If `pwd` matches the worktree path of the branch you are about to merge,
STOP. Change directory to the main root first.

## 3.5b. Uncommitted Work → Recovery Quarantine

Check the worktree (via its path, not by `cd`-ing into it) for any
uncommitted changes:

```bash
WORKTREE_PATH=$(git worktree list --porcelain \
  | awk -v b="{worktree-branch}" '/^worktree / {p=$2} $0=="branch refs/heads/"b {print p; exit}')

DIRTY=$(git -C "$WORKTREE_PATH" status --porcelain)
```

### If clean (`$DIRTY` empty): proceed to 3.5c.

### If dirty: RECOVER and QUARANTINE, do NOT merge.

The Commit Gate (workstream Phase 3.5 in `phase-2-spawn.md`) should have
caught this. A dirty worktree at merge time signals a deeper failure
(interrupted agent, scope confusion, pipeline exit without gate). Recovery
preserves the work; quarantine prevents silent merging of unreviewed output.

```bash
# 1. Commit the leftover work in the worktree, prefixed [recovery]
git -C "$WORKTREE_PATH" add -A
git -C "$WORKTREE_PATH" commit -m "[recovery] {workstream}: uncommitted at merge time" \
  --author="orchestrate-recovery <orchestrate@local>"

# 2. Capture the recovery diff for the acceptance report
RECOVERY_SHA=$(git -C "$WORKTREE_PATH" rev-parse HEAD)
mkdir -p ".pipeline/$RUN_NAME/recovery"
git -C "$WORKTREE_PATH" show "$RECOVERY_SHA" \
  > ".pipeline/$RUN_NAME/recovery/{workstream}.diff"
```

Then mark the workstream in `state.json`:
```json
{
  "status": "quarantined",
  "quarantined": true,
  "quarantine_reason": "uncommitted work at merge time",
  "recovery_commit": "{RECOVERY_SHA}",
  "recovery_diff": ".pipeline/{RUN_NAME}/recovery/{workstream}.diff",
  "merge_skipped": true
}
```

**Quarantined branches DO NOT merge to the feature branch.** They remain as
worktree branches, flagged in the Phase 5 acceptance report under "Next
actions for human" with the recovery diff path. The user reviews manually
before deciding to merge.

The orchestrator continues with other non-quarantined branches in the same
round — one quarantined workstream does not halt the whole run.

## 3.5c. Scope Diff Check

Compare the files the branch actually touches against the `scope_globs`
assigned to the workstream in Phase 1a. Flag anything outside.

```bash
# Files changed on the worktree branch vs the feature branch base
CHANGED=$(git diff --name-only $FEATURE_BRANCH...{worktree-branch})

# Expected scope for this workstream (from state.json)
SCOPE_GLOBS=$(jq -r ".rounds[$ROUND].workstreams[] | select(.name==\"{workstream}\") | .scope_globs | .[]" \
  ".pipeline/$RUN_NAME/state.json")

# Flag files that match NO glob in SCOPE_GLOBS
OUT_OF_SCOPE=$(echo "$CHANGED" | while read f; do
  [ -z "$f" ] && continue
  MATCH=0
  for g in $SCOPE_GLOBS; do
    case "$f" in $g) MATCH=1; break;; esac
  done
  [ "$MATCH" = "0" ] && echo "$f"
done)
```

If `$OUT_OF_SCOPE` is empty: proceed to 3.5d.

If non-empty, cross-reference with the workstream report's "Scope deviations"
section:

| Situation | Action |
|-----------|--------|
| File is out of scope AND listed as a deviation with justification | Note in state, proceed with merge, flag for final review |
| File is out of scope AND not declared | **Scope creep** — launch fix agent to revert those files, OR halt the merge and surface to the user |
| File overlaps another same-round workstream's scope | **Foundation overlap** — STOP. Will cause conflicts or overwrite parallel work. Coordinate or revert before merging |

Record every out-of-scope file in state:
```json
{
  "scope_deviations": [
    { "file": "src/shared/util.ts", "declared": true, "reason": "needed shared helper" },
    { "file": "docs/unrelated.md", "declared": false, "classification": "creep", "action": "reverted" }
  ]
}
```

A clean scope diff is the common case. The check is cheap; the save when it
catches something is large.

## 3.5d. Record the Checklist Outcome

Update `state.rounds[n].workstreams[w].merge_safety`:
```json
{
  "merge_safety": {
    "main_root_verified": true,
    "worktree_clean": true,
    "quarantined": false,
    "scope_deviations": [],
    "proceed_to_merge": true
  }
}
```

If any check failed and the workstream was quarantined, set
`proceed_to_merge: false`. The orchestrator surfaces all quarantined or
blocked workstreams in the Phase 5 acceptance report's "Next actions for
human" section at the very top.
