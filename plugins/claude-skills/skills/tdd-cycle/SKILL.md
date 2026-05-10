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

Drive a single success criterion to green using four-to-five specialized
subagents, in strict order. The caller is a pure coordinator — it launches
agents and reads their reports. It does not edit code, run tests, or commit.

Stage 5 (Outcome Grader) runs only when the criterion carries a `rubric`.
Pure-binary criteria with only a `check` skip Stage 5 and stop at Stage 3.

## Input

```
{
  "criterion": {
    "id": "SC3",
    "text": "Login endpoint returns 401 on expired tokens",
    "check": "npx vitest run auth/login.test.ts -t 'expired token'",
    "rubric": "## Status code\n- Returns 401 on expired token\n- Returns 401 on missing exp claim\n## Response body\n- error field is non-empty\n- Does NOT leak the decoded token"
  },
  "workstream": "auth-service",
  "scope_globs": ["src/auth/**", "tests/auth/**"],
  "context": "optional prior-criterion summary or shared types"
}
```

`criterion.check` and `criterion.rubric` are each optional individually but
at least one must be present (enforced by `extract-criteria`'s quality bar).

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

If `overall: RED`, advance to Stage 4.

If `overall: GREEN`:
- If `criterion.rubric` is present → advance to Stage 5.
- Otherwise → cycle is DONE; record success and return.

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

### Stage 5 — Outcome Grader

Only runs when `criterion.rubric` is present and Stage 3 returned GREEN.
Spawns the bias-isolated `criterion-grader` agent against the rubric.
Full design notes and worked examples in
[references/outcome-grader.md](references/outcome-grader.md).

```
Agent({
  description: "Grade {criterion.id} against rubric",
  subagent_type: "criterion-grader",
  prompt: "Grade this criterion against its rubric.

criterion:
  id: {criterion.id}
  text: {criterion.text}
  rubric: |
    {criterion.rubric}

artifact:
  test_file: {stage1.test_file}
  test_commit: {stage1.commit_sha}
  impl_commit: {stage2.commit_sha}
  scope_globs: {scope_globs}
  workstream: {workstream}

You have read-only access. Do not run tests or modify anything. Read the
test file, the diff between test_commit^ and impl_commit, and any in-scope
production source. Return the JSON verdict described in your agent
definition.
"
})
```

The grader returns a `GRADER_REPORT`:

```
GRADER_REPORT
  criterion_id: {id}
  result: satisfied | needs_revision | failed
  per_aspect: [{ aspect, status, bullets: [{ bullet, status, evidence, gap }] }]
  explanation: {one paragraph}
  files_inspected: [paths]
```

Route on `result`:

| Result | Next |
| --- | --- |
| `satisfied` | Cycle is DONE; record success including grader payload, return. |
| `needs_revision` | Route into Stage 4 (Fix Loop) with grader feedback as the failure context. The fix-loop agent receives the per-aspect `gap` strings as the work to do, NOT a failing test. |
| `failed` | Halt the cycle. The rubric and criterion text contradict each other; a human must reconcile. Record `status: "failed_grader_contradiction"` and surface the explanation in the CYCLE_REPORT. |

Hard cap: **3 grader iterations per criterion**, separate budget from the
fix-loop's 3 iterations. The combined cap is 6 revisions in the worst case
(3 fix-loop reds + 3 grader needs_revision). After 3 grader rejections:
- Record `status: "failed_grader"` on the criterion with all per-aspect
  gaps from the final iteration.
- Return failure to the caller. Same human-consent rule as the fix-loop.

Retry axis discipline for grader-driven revisions:
  1. Address every `gap` from the most recent grader report; same agents.
  2. Reduce scope: address only the highest-severity gap (per-aspect with
     most failed bullets) and re-grade.
  3. Fresh implementer: new subagent with the gaps + a "the prior approach
     missed these aspects entirely" framing.

## Output

The caller receives a `CYCLE_REPORT`:

```
CYCLE_REPORT
  criterion_id: {id}
  status: DONE | FAILED | FAILED_GRADER | FAILED_GRADER_CONTRADICTION
  test_file: {path}
  test_name: {name}
  test_commit: {sha}
  impl_commit: {sha}
  verify: { typecheck, tests, lint }  # from stage 3
  fix_loop_iterations: 0-3
  grader_iterations: 0-3        # 0 if criterion had no rubric
  per_aspect_results: [...]     # final grader report's per_aspect; null if no rubric
  scope_deviations: [...]
```

`status` values:
- `DONE` — Stage 3 GREEN, and either no rubric or grader returned `satisfied`.
- `FAILED` — fix-loop exhausted 3 retries without reaching GREEN.
- `FAILED_GRADER` — grader returned `needs_revision` 3 times.
- `FAILED_GRADER_CONTRADICTION` — grader returned `failed` (rubric vs.
  criterion text mismatch). Single-shot terminal status.

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
