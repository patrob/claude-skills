---
name: hunt-bugs
description: |
  Proactive, subsystem-wide bug hunt. Given a subsystem name (e.g. "Clerk auth",
  "upload pipeline", "subscription tier logic"), autonomously: (1) recon ranks
  5-10 plausible bug hypotheses with evidence, (2) parallel test-writer agents
  create a failing integration test per hypothesis, (3) a sequential fixer agent
  iterates up to 5 attempts per confirmed bug, (4) each confirmed bug+fix becomes
  its own commit on a shared branch, (5) a PR is drafted listing all findings.
  Use when the user says "hunt bugs in X", "audit X", "find bugs in the Y
  subsystem", or names an area and wants proactive discovery (vs. fixing one
  known bug — for that use `pw-bug-hunt`).
---

# Subsystem Bug Hunt

Proactively hunt for bugs across a named subsystem: recon → parallel failing
tests → sequential fixes → commits → PR.

## Input

The subsystem name is required. Accept it as:
1. A plain string from `$ARGUMENTS` (e.g., `Clerk auth`, `upload pipeline`)
2. A GitHub issue/PR that scopes the subsystem (`gh issue view N` for context)
3. A file path or glob that bounds the subsystem (e.g., `src/auth/**`)

If no subsystem is provided, ask the user. Do not guess.

## Setup

1. Slug and run directory:
   ```bash
   SLUG=$(echo "$SUBSYSTEM" | tr '[:upper:]' '[:lower:]' | tr -cs '[:alnum:]' '-' | sed 's/-$//')
   RUN_NAME="hunt-bugs-$SLUG-$(date +%Y%m%d-%H%M%S)"
   mkdir -p .pipeline/$RUN_NAME/tests
   ```

2. Shared branch for all bug commits:
   ```bash
   BASE_BRANCH=$(git rev-parse --abbrev-ref HEAD)
   BRANCH="hunt-bugs/$SLUG-$(date +%Y%m%d-%H%M%S)"
   git checkout -b "$BRANCH"
   ```
   Refuse to run on `main`/`master` without an explicit confirmation from the
   user — ask first, then proceed on the new branch.

3. Detect project verification commands once, up front, and store for reuse:
   - Test runner: check `package.json` scripts, `Makefile`, `pyproject.toml`,
     `Cargo.toml`, `go.mod`. Prefer `make verify` if present.
   - Single-test-file invocation (e.g., `npx vitest run {path}`,
     `pytest {path}`, `go test ./{pkg}`).
   - Full-suite invocation.
   Record both in `.pipeline/$RUN_NAME/env.json`. If detection fails, ask the
   user for the exact commands rather than guessing.

4. Read `CLAUDE.md` (if present) for project conventions, testing gotchas, and
   forbidden patterns. Every subagent below is told to read it too.

## Workflow

### Phase 1 — Recon (single agent)

Launch ONE `Explore` agent to map the subsystem and produce a ranked hypothesis
list. The agent does NOT write code or tests — only research.

Read `{SKILL_DIR}/references/recon-protocol.md` for the full agent prompt
template. The agent writes `.pipeline/$RUN_NAME/hypotheses.md`, one section per
hypothesis, ranked by likelihood × impact, with:
- `id` (`H1`..`Hn`) — used as the commit/test identifier
- `title` — one-line description of the suspected bug
- `likelihood` (HIGH/MED/LOW) with evidence
- `impact` (HIGH/MED/LOW) with blast radius
- `file:line` evidence pointers
- `reproduction sketch` — the concrete scenario the test should exercise
- `suggested integration test shape` — entry point, fixtures, assertion

**Require 5-10 hypotheses.** If the agent returns fewer than 5, send it back
once with a stricter prompt asking it to go broader (race conditions, error
paths, input validation, auth edges, off-by-one on tier/quota checks, timezone
handling, pagination, retry/idempotency). If it can't get to 5 after one retry,
report to the user and continue with what was found.

### Phase 2 — Parallel Test Writers (one agent per hypothesis)

Launch ALL test-writer agents in a single message for true concurrency. Each
agent owns exactly one hypothesis and writes exactly one test file at a
predetermined path to avoid collisions:

```
.pipeline/$RUN_NAME/tests/H{n}-{slug}.md   ← agent report
{project test path}/hunt-bugs/H{n}-{slug}.{ext}   ← the actual test file
```

The test file path must be unique per hypothesis. Agents do NOT modify any
production code in this phase — only add test files.

**Test-writer agent prompt**:
```
You are writing ONE failing integration test to reproduce a suspected bug.

Subsystem: {subsystem name}
Hypothesis ID: {H1}
Hypothesis: {full hypothesis block from hypotheses.md}

Your job:
1. Read CLAUDE.md and the hypothesis evidence files carefully.
2. Write a SINGLE integration test file at: {predetermined path}
   - Follow the project's existing integration test patterns exactly
   - Exercise the real code path (no mocking of the subsystem under test)
   - Mock only true external dependencies (third-party APIs, clock, randomness)
   - The test must fail ONLY if the suspected bug is real
3. Run the test in isolation using: {single-test command from env.json}
4. Classify the result:
   - FAIL for the expected reason (matches the hypothesis) → CONFIRMED
   - PASS → UNCONFIRMED (no bug, or test doesn't actually exercise the path)
   - FAIL for an unrelated reason (setup, import, fixture) → INCONCLUSIVE

Do NOT touch production code. Do NOT modify any other test files.
Do NOT disable, skip, or mark other tests as pending.

Report in this exact format:
---
TEST WRITER REPORT
Hypothesis: {H1} — {title}
Test file: {path}
Verdict: CONFIRMED / UNCONFIRMED / INCONCLUSIVE
Run command: {exact command to re-run this test}
Failing output (first 40 lines, if CONFIRMED):
  {captured output}
Reasoning: {one paragraph — why this failure matches the hypothesis}
---
```

After all test-writer agents finish, collect their reports into
`.pipeline/$RUN_NAME/test-results.md`. Bucket the results:
- `CONFIRMED` → proceed to Phase 3
- `UNCONFIRMED` → delete the test file, log to the run report
- `INCONCLUSIVE` → delete the test file, log for user review

If ZERO hypotheses are confirmed: skip to Phase 4 with an empty-result PR
description. Do not fabricate bugs.

### Phase 3 — Sequential Fixers (one agent per confirmed bug)

**Sequential, not parallel.** Fixers must see the cumulative state of prior
fixes, and parallel fixers would conflict on shared files. Within each bug's
fix loop the agent may still use parallel tool calls for its own work.

For each `CONFIRMED` hypothesis, in order of (impact DESC, likelihood DESC):

#### 3a. Stage the failing test

```bash
git add {test file}
```

Do not commit yet — the fix is bundled with the test in a single commit so the
history stays bisectable and each commit is self-contained.

#### 3b. Launch the fixer agent (up to 5 attempts)

Use `subagent_type: general-purpose` so the agent has full Edit + Bash access.
Read `{SKILL_DIR}/references/fix-protocol.md` for the full loop. The essential
contract is:

```
You are fixing the bug identified by a single failing test.

Hypothesis: {H1} — {title}
Failing test: {path}
Test command (single): {single-test command}
Full-suite command: {full-suite command}
Hypothesis evidence: {paste from hypotheses.md}

You have a budget of 5 attempts. Each attempt:
  1. Form or refine a root-cause theory (do NOT modify the test).
  2. Make the MINIMAL production-code change that addresses the root cause.
  3. Run the single failing test — it MUST now pass.
  4. Run the full test suite — NO other test may regress.
  5. If both pass: stop, report SUCCESS.
     If either fails: read the output, refine the theory, try again.

Hard rules:
- NEVER modify the failing test to make it pass
- NEVER add `.skip`, `.todo`, `xit`, `@pytest.mark.skip`, `t.Skip`, etc.
- NEVER add `// @ts-ignore`, `# type: ignore`, `eslint-disable`, or equivalent
- NEVER delete, weaken, or skip other tests to clear regressions
- If the "fix" requires any of the above, stop and report FAILED instead
- Bug fixes do not need feature flags (per CLAUDE.md)

Report in this exact format:
---
FIXER REPORT
Hypothesis: {H1}
Status: SUCCESS / FAILED
Attempts used: {n}/5
Files modified (production): {list}
Root cause (one paragraph): {...}
Fix summary (one paragraph): {...}
Single test: PASS/FAIL
Full suite: PASS ({n}/{total}) / FAIL ({regressed tests})
---
```

#### 3c. Handle the fixer result

**SUCCESS**:
```bash
git add -A
git commit -m "fix({slug}): {hypothesis title} [H{n}]

Failing test: {test path}
Root cause: {one sentence}

Refs: .pipeline/$RUN_NAME/hypotheses.md"
```

**FAILED** (budget exhausted or regression unavoidable):
```bash
git restore --staged {test file}
rm {test file}
```
Log to `.pipeline/$RUN_NAME/failed-fixes.md` with the full fixer report so the
user can follow up manually. Move on to the next hypothesis.

**After every successful fix**, run the full suite once more on the updated
branch to confirm no cross-bug regression before starting the next fixer.

### Phase 4 — PR Draft

After all confirmed hypotheses have been attempted:

1. Push the branch:
   ```bash
   git push -u origin "$BRANCH"
   ```

2. Generate `.pipeline/$RUN_NAME/pr-body.md` with this structure:
   ```markdown
   # Bug hunt: {Subsystem}

   Proactive audit of the **{subsystem}** subsystem. Each confirmed bug is a
   separate commit containing a failing integration test (now passing) and the
   minimal production-code fix.

   ## Summary
   - Hypotheses generated: {N}
   - Confirmed (failing test reproduced): {C}
   - Fixed (test now passes, no regressions): {F}
   - Unconfirmed / inconclusive: {N - C}
   - Fix attempts exhausted: {C - F}

   ## Bugs fixed
   For each successful fix:
   ### H{n}: {title}
   - Commit: `{short sha}`
   - Test: `{test path}`
   - Root cause: {one sentence}
   - Fix: {one sentence}

   ## Unconfirmed hypotheses
   For each unconfirmed / inconclusive hypothesis, list the title and a one-line
   reason the test didn't reproduce it. These are worth re-examining by hand.

   ## Fix attempts exhausted
   For each FAILED fixer, link to `.pipeline/$RUN_NAME/failed-fixes.md#H{n}`
   and summarize what the fixer tried.

   ## How to review
   - `git log --oneline {BASE_BRANCH}..{BRANCH}` — one commit per bug
   - Each commit is independently revertable
   - Run the full suite: `{full-suite command}`
   ```

3. Open the PR (ask first unless the user invoked in auto mode):
   ```bash
   gh pr create --base "$BASE_BRANCH" --head "$BRANCH" \
     --title "Bug hunt: {Subsystem} ({F} fixes)" \
     --body-file .pipeline/$RUN_NAME/pr-body.md
   ```

   Per global CLAUDE.md rules, do NOT add Claude attribution / co-author lines.
   Do NOT include unchecked todo boxes in the PR body.

### Phase 5 — Report to User

Display a terse summary:
- Branch name
- N hypotheses → C confirmed → F fixed
- PR URL (or "draft ready, not pushed" if auto-push declined)
- Path to `.pipeline/$RUN_NAME/` for full artifacts

## Arguments

- `/hunt-bugs {subsystem}` — default: interactive confirmation at branch
  creation and before `gh pr create`.
- `/hunt-bugs auto {subsystem}` — no confirmations; still refuses to run
  directly on `main`/`master`.

## Failure Handling

- **Recon returns < 5 hypotheses after one retry**: continue with what exists;
  note in the report.
- **A test-writer agent crashes**: log and skip that hypothesis; do not retry
  automatically (likely infra issue worth surfacing).
- **Fixer budget exhausted**: remove the test, log to `failed-fixes.md`,
  continue to the next hypothesis — partial progress is still valuable.
- **Full-suite baseline was already red before Phase 1**: stop immediately.
  The skill assumes a green baseline; a red baseline makes regression detection
  impossible. Ask the user to stabilize first.

## Notes

- The test-writer phase is the gate: only CONFIRMED hypotheses produce commits.
  Unconfirmed hypotheses cost only test-writer time, not churn on the branch.
- Fixers run sequentially so each sees the tree after prior fixes. This is
  slower than parallel fixing but avoids merge conflicts on overlapping files
  (very common in a single subsystem) and catches cross-bug regressions early.
- The shared branch is the deliverable. Every commit is bisectable and
  independently revertable. If a reviewer rejects one bug's fix, drop that
  commit without affecting the others.
- Per project CLAUDE.md: bug fixes do NOT use feature flags.
- This skill is complementary to `pw-bug-hunt`, which fixes ONE known bug.
  Use `hunt-bugs` when you don't have a specific bug — just a suspect area.
