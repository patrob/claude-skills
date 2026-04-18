# Verify Gate — Retry Semantics

Load this reference whenever a verify gate runs (inside a workstream agent's
Phase 4 or the Phase 4 final verification).

## The Gate

Run all three checks. All must pass:
- **Type check**: `tsc --noEmit`, `mypy`, `go build ./...`, `cargo check`
- **Tests**: project's full test command (not `--grep`, not partial)
- **Lint**: `eslint`, `biome check`, `ruff`, `golangci-lint`, `cargo clippy`

Detect which are applicable from project config. If a check type has no
tooling present, log `N/A` and continue — do NOT skip to avoid a failure.

## Retry Budget: 3 Attempts, Three Distinct Axes

Each attempt MUST use a different axis. Re-trying the exact same approach is
forbidden — blind retries waste the budget and usually fail the same way.

### Attempt 1 — Feedback Loop (same agent, same context)

The cheapest and most common fix path.

1. Capture failure output verbatim (test names, lint rule IDs, tsc error codes,
   file:line references).
2. Append to the agent's context: "The verify gate failed with:
   {failure_output}. Fix these specific errors and re-run verify."
3. Agent edits in place, re-runs the gate.

Terminates attempt on: all green (success), or re-fails (advance to attempt 2).

### Attempt 2 — Reduced Scope (same agent, narrowed prompt)

Forces the agent to isolate the failure instead of thrashing.

1. Instruct: "Identify the minimal slice of your changes that causes the
   failure. Revert unrelated changes with `git checkout HEAD~N -- <files>`
   or by selectively undoing edits. Commit the reverted state, then re-attempt
   only the failing slice."
2. Agent reduces change set, re-runs the gate.

Terminates attempt on: all green (success), or re-fails (advance to attempt 3).

### Attempt 3 — Fresh Agent (new subagent, clean context)

Clears accumulated context bias. The original agent may be stuck in a wrong
mental model.

```
Agent({
  description: "Verify-gate retry (fresh): {workstream}",
  isolation: "worktree",
  prompt: "You are a fresh agent taking over workstream {name} at verify stage.

  Previous agent's summary of approach: {condensed plan, 5 lines max}
  Previous verify failures (attempts 1-2):
  {failure_output_1}
  {failure_output_2}

  Explicit instruction: the previous approach did not converge. Choose a
  different approach. Read the current branch state, understand what is
  failing, and fix it. Do NOT repeat the fixes attempted in attempts 1-2
  (listed above).

  Workstream acceptance criteria: {AC list}
  Scope globs: {scope_globs}

  Run the verify gate when done. Report PASS or FAIL with the output of the
  final gate run."
})
```

Terminates attempt on: all green (success → workstream COMPLETE), or
re-fails (workstream FAILED).

## Termination

- First green gate → workstream `status: complete`, record `verify_attempts_used: N`.
- All 3 red → workstream `status: failed`, record all 3 failure outputs in
  `state.rounds[n].workstreams[w].verify_failures`.
- Never retry beyond 3. Infinite retry masks systematic problems and burns
  tokens.

## Logging Shape

Each verify attempt appends an entry to the workstream's state.json record:

```json
{
  "verify_attempts": [
    {
      "attempt": 1,
      "axis": "feedback-loop",
      "result": "fail",
      "failures": { "tests": 3, "lint": 0, "typecheck": 0 },
      "summary": "3 auth tests failing: token expiry off by 1h"
    },
    {
      "attempt": 2,
      "axis": "reduced-scope",
      "result": "fail",
      "failures": { "tests": 1, "lint": 0, "typecheck": 0 },
      "summary": "1 test still failing after reverting session-store changes"
    },
    {
      "attempt": 3,
      "axis": "fresh-agent",
      "result": "pass",
      "failures": {},
      "summary": "fresh agent used JWT exp field directly instead of reimplementing"
    }
  ],
  "verify_attempts_used": 3
}
```

This log is surfaced in the final acceptance report so the user can see when
a workstream struggled even if it ultimately succeeded.
