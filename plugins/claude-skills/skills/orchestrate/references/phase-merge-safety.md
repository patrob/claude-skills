# Phase Merge-Safety + Final Verify

Load when the merge-coordinator runs. This folds the old `phase-3.5-merge-safety.md`,
`merge-protocol.md`, and `phase-4-verify-final.md` into one reference.

**Delegation note.** The `/orchestrate` main agent does NOT merge directly.
It spawns a merge-coordinator subagent and hands it this reference + the
round's workstream reports. Every bash below runs inside that subagent.

## 1. Leave the Worktree (MANDATORY)

Operate from the main repo root for the entire merge + cleanup. Never `cd`
into a worktree you are about to merge or delete.

```bash
MAIN_ROOT=$(git worktree list --porcelain | awk '/^worktree / {print $2; exit}')
cd "$MAIN_ROOT"
pwd                                 # must equal $MAIN_ROOT
git rev-parse --show-toplevel       # must equal $MAIN_ROOT
git branch --show-current           # must equal $FEATURE_BRANCH
```

If `pwd` is a worktree path, STOP and `cd $MAIN_ROOT`.

## 2. Dirty Worktree → Recover + Quarantine

```bash
WORKTREE_PATH=$(git worktree list --porcelain \
  | awk -v b="{branch}" '/^worktree / {p=$2} $0=="branch refs/heads/"b {print p; exit}')
DIRTY=$(git -C "$WORKTREE_PATH" status --porcelain)
```

**Clean** (`$DIRTY` empty): proceed to §3.

**Dirty:** recover + quarantine; do NOT merge.

```bash
git -C "$WORKTREE_PATH" add -A
git -C "$WORKTREE_PATH" commit -m "[recovery] {workstream}: uncommitted at merge time" \
  --author="orchestrate-recovery <orchestrate@local>"
RECOVERY_SHA=$(git -C "$WORKTREE_PATH" rev-parse HEAD)
mkdir -p ".pipeline/$RUN_NAME/recovery"
git -C "$WORKTREE_PATH" show "$RECOVERY_SHA" \
  > ".pipeline/$RUN_NAME/recovery/{workstream}.diff"
```

Mark the workstream in `state.json`:
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

Quarantined branches do NOT merge, persist until user review, and surface
under "Next Actions for Human" in the acceptance report. Continue with
other non-quarantined branches — one quarantine does not halt the run.

## 3. Scope Diff Check

```bash
CHANGED=$(git diff --name-only $FEATURE_BRANCH...{branch})
SCOPE_GLOBS=$(jq -r ".rounds[$ROUND].workstreams[] | select(.name==\"{workstream}\") | .scope_globs[]" \
  ".pipeline/$RUN_NAME/state.json")
OUT_OF_SCOPE=$(echo "$CHANGED" | while read f; do
  [ -z "$f" ] && continue
  M=0; for g in $SCOPE_GLOBS; do case "$f" in $g) M=1; break;; esac; done
  [ "$M" = "0" ] && echo "$f"
done)
```

Empty → proceed to §4. Otherwise cross-reference the workstream report's
`scope_deviations`:

| Situation | Action |
|-----------|--------|
| Declared with justification | Record, proceed, flag for final review |
| Not declared | **Scope creep** — spawn a fix-agent to revert, or halt that branch |
| Overlaps another same-round scope | **Foundation overlap** — STOP, surface for user coordination |

Record every deviation in state under `scope_deviations`.

## 4. Merge (per branch, in round order)

```bash
git checkout $FEATURE_BRANCH
git status                             # must be clean
git merge {branch} --no-ff -m "merge: {workstream}"
```

**Strict order:** all Round N before any Round N+1. Within a round, any
order (developed against the same base).

If conflicts: `git diff --name-only --diff-filter=U`, then spawn a
conflict-resolution agent:

```
Agent({
  description: "Resolve merge conflicts: {branch}",
  prompt: "The merge of {branch} into {FEATURE_BRANCH} has conflicts in
these files: {list with conflict markers}.

Context:
- {branch} implemented: {workstream description}
- {FEATURE_BRANCH} already has merges from: {list}

Instructions:
1. Read each conflicting file.
2. Understand the intent of BOTH sides.
3. Resolve preserving both intents — never accept one side wholesale.
4. Stage: git add {file}
5. Complete: git commit --no-edit
6. Run the verify gate (typecheck + tests + lint) and confirm green."
})
```

If the agent cannot resolve: `git merge --abort`, report the conflict to
the orchestrator, ask the user. If post-merge verify fails: spawn a
fix-agent with the error output; if the fix fails, offer
`git revert -m 1 HEAD` (preserves history) and ask the user.

## 5. `verify.final` Hook

After all non-quarantined branches for the round have merged, run the
configured final verification on the feature branch.

### Contract

Configure in roadmap YAML frontmatter (or `orchestrate.yml`):

```yaml
verify:
  final:
    commands:
      - { name: "typecheck", cmd: "npm run typecheck" }
      - { name: "lint",      cmd: "npm run lint" }
      - { name: "test",      cmd: "npm test -- --run" }
    smoke:
      type: "playwright"   # playwright | http | cli | none
      spec: "tests/smoke/login.spec.ts"
      retries: 1
```

A run is GREEN only when every command exits 0 AND smoke (if not `none`)
succeeds.

### Smoke types (brief)

- **playwright**: `spec` path, optional `install` cmd, optional `retries`.
- **http**: `url`, `expect_status`, `expect_body_contains`, `boot_cmd`, `boot_wait_seconds`.
- **cli**: `cmd`, `expect_exit`, `expect_stdout_contains`.
- **none**: for tooling repos, libraries, plugins with no runtime entrypoint.

### Auto-detection defaults (if `verify.final` is absent)

| Project signal | Default `commands` | Default `smoke` |
|----------------|-------------------|-----------------|
| `package.json` with test/lint/typecheck scripts | those three | `none` (or playwright if `playwright.config.*` present) |
| `pyproject.toml` with pytest/ruff/mypy | those three | `none` |
| `Cargo.toml` | `cargo check`, `cargo test`, `cargo clippy` | `none` |
| `go.mod` | `go vet`, `go test ./...` | `none` |
| Claude Code plugin | markdown-lint if configured, else empty | `none` |

If detection yields an empty commands list AND no smoke: surface a warning
("no verification configured — acceptance cannot be proven automatically")
and mark ACs as `verified: manual-required`.

### Final-verify agent

Launched on the feature branch, from main repo root:

```
Agent({
  description: "Final verification: {FEATURE_BRANCH}",
  prompt: "You are the final verification agent. Feature branch
{FEATURE_BRANCH} now contains all non-quarantined workstream merges.
Prove it is green end-to-end.

Contract:
{paste verify.final}

Steps:
1. Run each command in verify.final.commands in order. Capture stdout,
   stderr, exit code. Do NOT continue past a failing command.
2. If smoke.type != 'none', run the smoke per its type. Capture outputs.
3. Write .pipeline/{RUN_NAME}/verify-final.json:
   { status: 'green'|'red',
     commands: [ { name, exit, stdout_tail, duration_ms } ],
     smoke: { type, status, stdout_tail } }
4. Do NOT modify any source files — this is read-only verification.

On red: do NOT attempt fixes. The orchestrator invokes verify-fix-loop."
})
```

### On red

1. Spawn `verify-fix-loop` (the skill) on the failure output.
2. Each fix commit on the feature branch must be reviewed (spawn a
   `pw-review` agent) before the next final-verify attempt — integration
   fixes modify already-reviewed code and are the highest-regression-risk
   commits in a run.
3. Cap at 2 red cycles. After two, halt and write a red acceptance report.

## 6. Record Outcome

```json
{
  "merge_safety": {
    "main_root_verified": true,
    "worktree_clean": true,
    "quarantined": false,
    "scope_deviations": [],
    "proceed_to_merge": true,
    "merge_result": "clean|conflicts_resolved|failed|skipped",
    "verify_final": "green|red|unconfigured"
  }
}
```

## 7. AC → Evidence Mapping

Every AC recorded in Phase 1 must be checkable via one of:
- A named command in `verify.final.commands` whose test name/path matches
  the AC's `check`.
- A smoke step (for end-to-end happy-path ACs).
- An explicit per-workstream evidence entry (workstream report captured
  the AC check output inline).

ACs with no evidence source are flagged `verified: no` in the acceptance
report — that is a decomposition failure surfacing late.

## 8. Branch Cleanup

Worktree branches created with `isolation: "worktree"` are cleaned up
automatically on agent success. Quarantined branches persist for user
review. The feature branch remains for PR or manual merge to base.
