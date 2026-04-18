---
name: ship-feature
description: |
  End-to-end autonomous feature pipeline. Takes a one-sentence feature description and
  drives it through research → deliberation → worktree → parallel implementation → review →
  verify → PR, persisting state between phases for resume. Use when:
  (1) User says "ship", "ship a feature", "/ship-feature", or describes a feature they want
      built end-to-end without supervision
  (2) User provides a one-sentence feature description and expects a PR at the end
  (3) User wants to resume an interrupted ship-feature run
  Each phase saves artifacts to `.rpi-state/{run-name}/` and updates a TaskList row so the
  pipeline can be resumed from the last completed phase. Only prompts the user when a phase
  fails twice in a row.
---

# Ship Feature — Autonomous End-to-End Pipeline

Drive a one-sentence feature description through 11 phases to a reviewed, verified PR.
State is persisted at every boundary so any interruption can be resumed.

## Input

Either:
1. **New run** — a one-sentence feature description (the `$ARGUMENTS` to the slash command)
2. **Resume** — the user references an existing run (`resume {run-name}` or `resume latest`).
   Read `.rpi-state/{run-name}/state.json` and skip to the first phase whose status is not
   `completed`.

If `$ARGUMENTS` is empty AND no resume target is given, ask the user once for a feature
description and stop the skill if they decline.

## Setup (new runs only)

1. Generate a kebab-case run name from the feature description, prefixed by date:
   ```bash
   RUN_NAME="ship-$(date +%Y%m%d-%H%M%S)-{kebab-slug}"
   STATE_DIR=".rpi-state/$RUN_NAME"
   mkdir -p "$STATE_DIR"
   ```

2. Write `$STATE_DIR/feature.md` with the one-sentence description and any clarifying
   context the user supplied.

3. Write the initial state file `$STATE_DIR/state.json`:
   ```json
   {
     "run_name": "{RUN_NAME}",
     "feature": "{one-sentence description}",
     "base_branch": "{git rev-parse --abbrev-ref HEAD}",
     "feature_branch": null,
     "worktree_path": null,
     "current_phase": "create-research-questions",
     "phases": {
       "create-research-questions":   { "status": "pending", "attempts": 0, "artifact": null },
       "iterate-research-questions":  { "status": "pending", "attempts": 0, "artifact": null },
       "create-research":             { "status": "pending", "attempts": 0, "artifact": null },
       "create-design-discussion":    { "status": "pending", "attempts": 0, "artifact": null },
       "create-structure-outline":    { "status": "pending", "attempts": 0, "artifact": null },
       "create-implementation-plan":  { "status": "pending", "attempts": 0, "artifact": null },
       "setup-worktree":              { "status": "pending", "attempts": 0, "artifact": null },
       "orchestrate":                 { "status": "pending", "attempts": 0, "artifact": null },
       "code-review-fix":             { "status": "pending", "attempts": 0, "artifact": null },
       "verify":                      { "status": "pending", "attempts": 0, "artifact": null },
       "describe-pr":                 { "status": "pending", "attempts": 0, "artifact": null }
     }
   }
   ```

4. Create one TaskCreate row per phase (subjects below). Capture each task ID into
   `state.json` under `phases.{phase}.task_id`. These tasks are the resumable index — if
   the user kills the session, `TaskList` shows exactly where it stopped.

   | Phase key                     | Task subject                                              |
   | ----------------------------- | --------------------------------------------------------- |
   | create-research-questions     | Draft research questions for {feature}                    |
   | iterate-research-questions    | Auto-answer research questions (--accept-all)             |
   | create-research               | Run pw-research                                           |
   | create-design-discussion      | Run pw-deliberate                                         |
   | create-structure-outline      | Extract file/module structure outline                     |
   | create-implementation-plan    | Finalize implementation plan from deliberation            |
   | setup-worktree                | Create feature branch + worktree                          |
   | orchestrate                   | Run orchestrate skill for parallel implementation         |
   | code-review-fix               | Run pw-review and apply blocking fixes                    |
   | verify                        | Run verify-fix-loop + Playwright smoke                    |
   | describe-pr                   | Open PR with generated description                        |

## State Discipline (applies to every phase)

Before a phase: `TaskUpdate(task_id, status: in_progress)` and set
`state.json.current_phase` to the phase key.

After a phase succeeds:
1. Write the artifact under `$STATE_DIR/{phase-key}/` (paths listed per phase below).
2. Update `state.json` — set `status: completed`, increment nothing, set `artifact` to the
   primary output path, advance `current_phase` to the next pending phase.
3. `TaskUpdate(task_id, status: completed)`.
4. Emit a single line to the user: `[{N}/11] {phase-key} ✓ — {one-line summary}`. Nothing
   else. No headers, no bullet lists.

After a phase fails:
1. Increment `phases.{phase}.attempts` in `state.json`.
2. If `attempts < 2`: retry the same phase from the start. Do NOT touch the task status.
3. If `attempts >= 2`: keep the task as `in_progress`, write the failure detail to
   `$STATE_DIR/{phase-key}/error.md`, emit `[{N}/11] {phase-key} ✗ — failed twice: {reason}`,
   and ask the user once how to proceed (retry / skip / abort). Stop the pipeline until the
   user responds.

## Resume Logic

If invoked with a resume target:
1. Read `$STATE_DIR/state.json`.
2. Locate the first phase whose status is not `completed`. If its attempts are already
   `>= 2`, treat it as failed-twice and surface the error immediately rather than retrying.
3. Otherwise, reset that phase's `attempts` to 0 and start it. Earlier completed phases
   are not re-run; their artifacts are loaded from `$STATE_DIR`.

## Phases

### 1. create-research-questions
**Goal**: Turn the one-sentence feature into a concrete list of questions worth answering
before designing the solution.

**How**: Inline. Write 5–10 questions covering: target users, non-goals, integration
surface (which existing modules), data shape, failure modes, performance/scale, security,
testability. Save to `$STATE_DIR/research-questions.md` as a numbered markdown list.

**Summary line example**: `[1/11] create-research-questions ✓ — drafted 8 questions`

### 2. iterate-research-questions (--accept-all)
**Goal**: Produce a best-effort answer for every question without prompting the user.

**How**: Inline. For each question in `research-questions.md`, write an answer using only:
the feature description, the codebase (read CLAUDE.md, README.md, top-level structure), and
defensible defaults. Mark assumptions with `_[assumption]_`. Save to
`$STATE_DIR/research-answers.md`. Do not prompt the user for clarification — that's what
`--accept-all` semantics mean here.

**Summary line example**: `[2/11] iterate-research-questions ✓ — answered 8/8 (3 assumed)`

### 3. create-research
**Goal**: Deep parallel research using the existing skill.

**How**: Invoke the `pw-research` skill, passing the contents of `feature.md` plus the
answered questions from step 2 as context. After completion, copy the produced
`.pipeline/{run-name}/research.md` into `$STATE_DIR/research/research.md`. If pw-research
already used the same run-name, just record the path.

**Summary line example**: `[3/11] create-research ✓ — research.md ({N} findings)`

### 4. create-design-discussion
**Goal**: Multi-persona deliberation producing checkpointed design.

**How**: Invoke the `pw-deliberate` skill with the research artifact. It produces a
`plan.md`. Copy/symlink the resulting `plan.md` into
`$STATE_DIR/design/deliberation.md`.

**Summary line example**: `[4/11] create-design-discussion ✓ — {N} checkpoints proposed`

### 5. create-structure-outline
**Goal**: A flat list of files to create or modify, grouped by checkpoint, derived from
the deliberation.

**How**: Inline. Parse `design/deliberation.md` and produce
`$STATE_DIR/structure-outline.md` with one section per checkpoint, each listing
`{create|modify} {path} — {one-line purpose}`. This is what gets fed to `orchestrate` so
each worktree agent has unambiguous file targets.

**Summary line example**: `[5/11] create-structure-outline ✓ — {N} files across {M} checkpoints`

### 6. create-implementation-plan
**Goal**: A single `plan.md` ready for `orchestrate` to consume.

**How**: Inline. Merge `design/deliberation.md` and `structure-outline.md` into
`$STATE_DIR/plan.md` using the format `orchestrate` expects (workstreams with name,
description, files, acceptance criteria, dependencies). If the deliberation already
emitted a plan that fits the format, this step just validates and copies it.

**Summary line example**: `[6/11] create-implementation-plan ✓ — {N} workstreams in plan.md`

### 7. setup-worktree
**Goal**: Isolate the build in its own branch + worktree.

**How**: Inline.
```bash
FEATURE_BRANCH="ship/{run-name}"
WORKTREE_PATH=".claude/worktrees/{run-name}"
git worktree add -b "$FEATURE_BRANCH" "$WORKTREE_PATH" "$BASE_BRANCH"
```
Record both into `state.json`. Copy `$STATE_DIR/plan.md` into the worktree at the same
relative path so subsequent phases can run from inside it.

**Summary line example**: `[7/11] setup-worktree ✓ — branch ship/{run-name} at {path}`

### 8. orchestrate
**Goal**: Parallel implementation across workstreams.

**How**: Invoke the `orchestrate` skill, pointed at `$STATE_DIR/plan.md`. Run in auto mode
(skip the y/n confirmation in orchestrate's Phase 1d). On completion, capture
`orchestrate`'s `.pipeline/{run}/state.json` path into `state.json.phases.orchestrate.artifact`.

**Summary line example**: `[8/11] orchestrate ✓ — {N} workstreams merged`

### 9. code-review-fix
**Goal**: Address blocking findings from code review.

**How**: Invoke `pw-review` against the diff on the feature branch vs the base branch.
For every finding ranked Critical or High, launch fix agents in parallel (one per file)
to apply changes. Re-run `pw-review` once. Save the final review to
`$STATE_DIR/review/final.md`. Treat the phase as failed if any Critical findings remain
after the second review.

**Summary line example**: `[9/11] code-review-fix ✓ — {N} findings, {M} fixed, 0 critical remaining`

### 10. verify
**Goal**: All quality gates green.

**How**: Invoke `verify-fix-loop` (it covers tests/lint/typecheck/build). After it
returns success, run a Playwright smoke pass: detect a frontend by checking for any of
`apps/web`, `app`, `frontend`, `web`, or a `package.json` whose scripts include `dev` or
`start`; if found, start the dev server in the background, run a minimal Playwright check
(homepage loads, no console errors), then stop the server. If no frontend is detected,
record `playwright: skipped (no frontend detected)` and treat that as success. Save the
combined log to `$STATE_DIR/verify/log.txt`.

**Summary line example**: `[10/11] verify ✓ — tests/lint/typecheck/playwright all green`

### 11. describe-pr
**Goal**: Push the branch and open a PR with a generated description.

**How**: Inline.
```bash
git push -u origin "$FEATURE_BRANCH"
```
Compose the PR body from: the feature sentence, a 3-bullet summary derived from the
deliberation's checkpoints, and a test-plan checklist derived from the verify log. Then:
```bash
gh pr create --title "{short title from feature}" --base "$BASE_BRANCH" \
  --body "$(cat $STATE_DIR/pr-body.md)"
```
Save `$STATE_DIR/pr-body.md` and capture the PR URL into
`state.json.phases.describe-pr.artifact`.

Per the user's global commit conventions: do NOT add Claude attribution, co-authored-by
lines, or "Generated with Claude" footers to the PR body or any commits made during this
pipeline.

**Summary line example**: `[11/11] describe-pr ✓ — {PR URL}`

## Final Output

After phase 11 completes, emit a single block:

```
ship-feature complete: {feature sentence}
PR: {url}
Run state: .rpi-state/{run-name}/
```

Then stop. Do not summarize the diff, do not narrate the journey — the state directory
and the PR are the artifacts.

## Failure Modes Worth Knowing

- **Worktree already exists** — if `setup-worktree` finds the path occupied, treat it as a
  resume hint: check whether `$STATE_DIR/state.json` shows `setup-worktree` already
  completed and just continue from the next phase.
- **orchestrate produces zero workstreams** — `create-implementation-plan` produced an
  empty plan. Fail this phase (counts toward the two-strikes rule) so the user gets a
  prompt rather than a silent skip.
- **gh not authenticated** — `describe-pr` push works but `gh pr create` fails. Surface
  the `gh auth login` instruction in the failure message and stop.
- **No CI command available** — if `verify-fix-loop` reports no `make verify` target and
  no inferable test/lint/build commands, fail the phase and ask the user to specify the
  verify command.
