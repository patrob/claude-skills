---
name: tdd-cycle
description: |
  TDD one success criterion into existence via a test-writer → implementer →
  verifier → fix-loop pipeline of subagents. Use when:
  (1) You have ONE criterion (or test case) and want it implemented TDD-style
  (2) User says "TDD this", "write the test first, then make it pass"
  (3) Inside `/orchestrate`, called by each worktree agent per criterion
  (4) You want strict one-test-at-a-time discipline instead of batch test suites
  The caller delegates every step — it never writes tests or code itself. Runs
  until the criterion's test passes AND typecheck + lint + full suite are
  green with zero warnings, or the retry budget is exhausted.
---

# TDD Cycle — One Criterion

Drive a single success criterion to green using four specialized subagents,
in strict order. The caller is a pure coordinator — it launches agents and
reads their reports. It does not edit code, run tests, or commit.

## Input

```
{
  "criterion": {
    "id": "SC3",
    "text": "Login endpoint returns 401 on expired tokens",
    "check": "npx vitest run auth/login.test.ts -t 'expired token'"
  },
  "workstream": "auth-service",
  "scope_globs": ["src/auth/**", "tests/auth/**"],
  "context": "optional prior-criterion summary or shared types"
}
```

## Pipeline

All four agents run as `Agent(...)` calls. Each stage produces a report the
next stage consumes.

### Stage 1 — Test Writer

Spawn a subagent whose ONLY job is writing ONE failing test.

```
Agent({
  description: "Write failing test for {criterion.id}",
  subagent_type: "general-purpose",
  prompt: "You are a test-writer. Write EXACTLY ONE test for this criterion.

Criterion:
  id: {criterion.id}
  text: {criterion.text}
  check command: {criterion.check}

Workstream scope: {scope_globs}
Prior context: {context}

Rules:
- Follow existing test conventions in the repo (read 1-2 nearby tests first).
- The test MUST fail for the RIGHT reason — the functionality is missing,
  not a setup/import error.
- Write ONE test only. No setup for future tests. No test suite scaffolding
  beyond what the framework requires for this single test.
- Do NOT implement the production code.

Deliverable — when done, run the test and report:

TEST_WRITER_REPORT
  test_file: {absolute path}
  test_name: {exact test name / describe-it path}
  test_body: |
    {the literal source of the test, verbatim}
  run_command: {criterion.check command}
  run_output_tail: |
    {last 20 lines of failing output — must show the intended failure}
  failure_reason: {one sentence — e.g. 'function does not exist' or
                   'returns 200 instead of 401'}
  commit_sha: {after `git add && git commit -m 'test: {criterion.id} (failing)'`}
"
})
```

If `failure_reason` indicates a setup error (import failure, missing
fixture) rather than missing functionality, reject the report and respawn
the test writer with the error appended. Max 2 attempts.

### Stage 2 — Implementer

Spawn a subagent whose ONLY job is making the ONE test pass.

```
Agent({
  description: "Implement to pass {criterion.id}",
  subagent_type: "general-purpose",
  prompt: "You are an implementer. Make exactly ONE test pass with minimal
code changes.

Failing test:
  file: {stage1.test_file}
  name: {stage1.test_name}
  source:
    {stage1.test_body}
  current run output:
    {stage1.run_output_tail}
  expected failure (was): {stage1.failure_reason}

Workstream scope: {scope_globs}
Run command: {criterion.check}

Rules:
- Change ONLY files inside scope_globs. If you must touch a file outside
  scope, note it in your report under 'scope_deviations'.
- Do NOT modify the test to make it pass. The test is fixed; the code
  changes.
- Minimal change — no refactoring, no adjacent fixes, no TODO cleanup.
- Do NOT disable warnings, skip tests, or use `any` / `@ts-ignore` /
  `eslint-disable` / `@ts-expect-error` / empty catch.
- After editing, run `{criterion.check}` and confirm the test passes.

Deliverable:

IMPLEMENTER_REPORT
  files_modified: [list]
  scope_deviations: [{ file, reason }] or []
  test_result: PASS | FAIL
  run_output_tail: |
    {last 20 lines}
  commit_sha: {after `git add && git commit -m 'feat: {criterion.id}'`}
"
})
```

If `test_result: FAIL`, advance to Stage 4 with the failure context. Do NOT
retry Stage 2 — the implementer failed its basic job, and escalation to the
fix-loop is cheaper than re-prompting the same agent.

### Stage 3 — Verifier

Spawn a subagent to run full typecheck + tests + lint — zero warnings
allowed.

```
Agent({
  description: "Verify gate for {criterion.id}",
  subagent_type: "general-purpose",
  prompt: "You are a verifier. Run the full verify gate and report exit
status + output. Do NOT edit any files.

Commands to run in order (stop on first failure):
  1. Typecheck: {detected or supplied}
  2. Full test suite: {detected or supplied}
  3. Lint: {detected or supplied}

Rules:
- Zero warnings allowed. Treat warnings on lint as failures.
- Capture stdout + stderr + exit code for each.
- Do NOT apply fixes.

Deliverable:

VERIFIER_REPORT
  typecheck: { exit, tail }
  tests: { exit, passed, total, tail }
  lint: { exit, warnings, errors, tail }
  overall: GREEN | RED
"
})
```

If `overall: GREEN`, the cycle is DONE — record success and return.

### Stage 4 — Fix Loop

If Stage 2 failed OR Stage 3 is RED, invoke `Skill(verify-fix-loop)` with
the failure payload. That skill itself spawns parallel fix agents per
failing file and re-runs verification until green.

Hard cap: 3 fix-loop iterations per criterion. After 3 reds:
  - Record `status: "failed"` on the criterion with all failure outputs.
  - Return the failure to the caller. Do NOT continue to the next criterion
    in the calling loop without the caller's consent.

Retry axis discipline (applies to these 3 iterations — see
`orchestrate/references/retry-semantics.md` for full spec):
  1. Feedback loop (same fix agents, failure output in context)
  2. Reduced scope (revert unrelated changes, fix minimal slice)
  3. Fresh agent (new subagent with "choose a different approach")

## Output

The caller receives a `CYCLE_REPORT`:

```
CYCLE_REPORT
  criterion_id: {id}
  status: DONE | FAILED
  test_file: {path}
  test_name: {name}
  test_commit: {sha}
  impl_commit: {sha}
  verify: { typecheck, tests, lint }  # from stage 3
  fix_loop_iterations: 0-3
  scope_deviations: [...]
```

## Delegation contract (enforced)

The caller of `tdd-cycle`:
- MUST NOT edit code, write tests, or run bash beyond `git status`, `git log`, `git diff`.
- MUST invoke stages in order; the report of stage N is the input to stage N+1.
- MUST treat every failure as a subagent's output — fix by re-spawning, not
  by taking over the work.

## Standalone use

`/tdd-cycle "Login returns 401 on expired tokens"` will:
1. Prompt for the run command if none is detected.
2. Run the full four-stage pipeline.
3. Leave two commits: `test: ...` and `feat: ...`.
4. Print the CYCLE_REPORT.

## When to prefer other skills

- **`pw-bug-hunt`** — if the criterion is "fix this reported bug" and the
  root cause is unknown. That skill adds a parallel-hypothesis investigation
  phase before the test-first step.
- **`verify-fix-loop`** — if you already have failing tests/lint/typecheck
  and just need to iterate to green (no new test required).
