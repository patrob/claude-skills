# Merge Protocol

How to merge worktree branches into the feature branch safely.

## Prerequisites

Before merging any worktree branch:
1. The branch has been reviewed (Phase 3 complete)
2. The feature branch is checked out and clean (`git status` shows no changes)
3. All earlier-round branches have already been merged (dependency order)

## Merge Order

Strict ordering rules:
1. **Round order**: All Round N branches merge before any Round N+1 branch
2. **Within a round**: Any order is fine (they were developed in parallel against the same base)
3. **Sequential tasks**: Merge in the order specified by the plan

## Merge Steps

For each branch to merge:

### Step 1: Ensure Clean State

```bash
git checkout $FEATURE_BRANCH
git status  # must be clean
```

### Step 1b: Verify Worktree Branch Is Committed

Before merging, confirm the worktree branch has no uncommitted work:
```bash
git stash
git checkout {worktree-branch}
DIRTY=$(git status --porcelain)
git checkout $FEATURE_BRANCH
git stash pop 2>/dev/null
```

If `$DIRTY` is non-empty, launch a small agent to commit the remaining work:
```
Agent({
  description: "Commit uncommitted work on {worktree-branch}",
  prompt: "Branch {worktree-branch} has uncommitted changes.
  Run: git add -A && git commit -m '{workstream}: uncommitted implementation work'
  Then run: git status to confirm clean state.
  Report the list of files that were committed."
})
```

This is a safety net. The workstream agent's Commit Gate (Phase 3.5) should
have caught this — if we reach this point, log a warning in the orchestration report.

### Step 2: Merge with No-FF

```bash
git merge {worktree-branch} --no-ff -m "merge: {task name}"
```

Using `--no-ff` preserves the branch topology so each task's commits are visible
as a group in the history.

### Step 3: Check for Conflicts

If the merge completes cleanly → proceed to Step 5.

If conflicts occur → Step 4.

### Step 4: Conflict Resolution

#### 4a. Assess Conflicts

```bash
git diff --name-only --diff-filter=U
```

List all conflicting files. Categorize:
- **Trivial**: different sections of the same file edited → likely auto-resolvable
- **Semantic**: overlapping logic changes → needs understanding of both sides
- **Structural**: file moved/renamed vs edited → needs careful handling

#### 4b. Launch Conflict Resolution Agent

```
Agent({
  description: "Resolve merge conflicts",
  prompt: "The merge of branch {worktree-branch} into {feature_branch} has conflicts.

  Conflicting files:
  {list of files with conflict markers}

  Context:
  - {worktree-branch} implemented: {task description}
  - {feature_branch} contains merged work from: {list of already-merged tasks}

  Instructions:
  1. Read each conflicting file
  2. Understand the intent of both sides
  3. Resolve conflicts preserving both intents
  4. Stage the resolved files: git add {file}
  5. Complete the merge: git commit --no-edit
  6. Run verification to ensure nothing broke

  NEVER resolve by simply accepting one side. Both sides represent
  intentional work that must be preserved."
})
```

#### 4c. If Resolution Fails

If the agent cannot resolve cleanly:
1. Abort the merge: `git merge --abort`
2. Report the conflict details to the orchestrator
3. The orchestrator will ask the user for guidance

### Step 5: Post-Merge Verification

After each successful merge, run a quick verification:

```bash
# Adapt to project — detect what's available:
# Check for tsconfig.json → run tsc --noEmit
# Check for test runner → run tests
# Check for lint config → run lint
```

**If verification passes**: log success, continue to next branch.

**If verification fails**:
1. Identify which tests/checks failed
2. Determine if the failure is from the merge or pre-existing
3. If from the merge: launch a fix agent
   ```
   Agent({
     description: "Fix post-merge regression",
     prompt: "After merging {branch}, verification failed:
              {error output}
              Fix the regression. The issue is likely an integration
              problem between {branch}'s changes and previously merged work.
              Run verification after fixing."
   })
   ```
4. If fix succeeds: commit the fix, continue
5. If fix fails: offer to revert the merge (`git revert -m 1 HEAD`) and ask user

### Step 6: Log the Merge

Record in state.json:
```json
{
  "branch": "{worktree-branch}",
  "task": "{task name}",
  "result": "clean|conflicts_resolved|failed",
  "conflicts": ["{file list if any}"],
  "verification": "pass|fail",
  "timestamp": "{ISO}"
}
```

## Revert Protocol

If a merge needs to be reverted:

```bash
# Revert the merge commit (keep the branch topology)
git revert -m 1 {merge-commit-sha} --no-edit
```

This creates a new commit that undoes the merge without destroying history.
The worktree branch can be fixed and re-merged later.

## Branch Cleanup

After all merges complete and final verification passes:

Worktree branches are automatically cleaned up by the Agent tool when using
`isolation: "worktree"`. No manual cleanup needed.

The feature branch persists for the user to review, PR, or merge to main.
