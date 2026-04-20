# Verify Gate — Retry Semantics

Applies to the verify gate inside `tdd-cycle` (per criterion) and to the
final verify step in `phase-merge-safety.md`. `verify-fix-loop` is the
execution primitive; this file specifies the axis discipline.

## The Gate

All three must pass. Detect applicable tooling from project config; missing
toolchains are logged `N/A` (do not skip to avoid a failure).

- **Typecheck**: `tsc --noEmit`, `mypy`, `go build ./...`, `cargo check`
- **Tests**: the project's full test command (no `--grep`, no partial)
- **Lint**: `eslint`, `biome check`, `ruff`, `golangci-lint`, `cargo clippy` — warnings count as failures

## Retry Budget: 3 Attempts, 3 Distinct Axes

Each attempt MUST use a different axis. Blind re-runs waste the budget.

### Attempt 1 — Feedback Loop (same agent, same context)

Append the failure output verbatim (test names, rule IDs, tsc codes,
file:line) to the agent's context: "The verify gate failed with:
{output}. Fix these specific errors and re-run verify." Agent edits in
place, re-runs.

Terminate on green (success) or re-fail (→ attempt 2).

### Attempt 2 — Reduced Scope (same agent, narrowed prompt)

"Identify the minimal slice of your changes that causes the failure.
Revert unrelated changes (`git checkout HEAD~N -- <files>` or selective
undo). Commit the reverted state, then re-attempt only the failing slice."

Terminate on green or re-fail (→ attempt 3).

### Attempt 3 — Fresh Agent (new subagent, clean context)

Clears accumulated context bias. Spawn a new worktree-scoped agent with:
original prompt + summary of attempts 1-2 + explicit "choose a DIFFERENT
approach. Do not repeat the fixes attempted above."

Terminate on green (COMPLETE) or re-fail (FAILED).

## Termination

- First green → COMPLETE; record `verify_attempts_used: N` and the axes used.
- All 3 red → FAILED; record all 3 failure outputs in state under
  `verify_failures`.
- Never retry beyond 3 — infinite retry masks systematic problems and
  burns tokens.

## Logging Shape

```json
{
  "verify_attempts": [
    { "attempt": 1, "axis": "feedback-loop", "result": "fail",
      "failures": { "tests": 3, "lint": 0, "typecheck": 0 },
      "summary": "..." },
    { "attempt": 2, "axis": "reduced-scope", "result": "fail",
      "failures": { "tests": 1 }, "summary": "..." },
    { "attempt": 3, "axis": "fresh-agent", "result": "pass",
      "failures": {}, "summary": "..." }
  ],
  "verify_attempts_used": 3
}
```

Surfaced in the acceptance report so the user can see which criteria
struggled even if they ultimately passed.
