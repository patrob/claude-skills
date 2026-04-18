# Phase 4 — Final Verification (`verify.final` Hook)

Load this reference when executing Phase 4 of `/orchestrate`, after all
eligible worktree branches have merged to the feature branch.

The final verification agent proves that the merged feature branch satisfies
the full roadmap's acceptance criteria end-to-end. It runs the project's
verification stack plus a happy-path smoke, producing the evidence consumed
by the Phase 5 acceptance report.

## The `verify.final` Contract

Callers configure verification in their roadmap's YAML frontmatter (or in a
top-level `orchestrate.yml`). The shape:

```yaml
verify:
  final:
    commands:
      - { name: "typecheck", cmd: "npm run typecheck" }
      - { name: "lint",      cmd: "npm run lint" }
      - { name: "test",      cmd: "npm test -- --run" }
    smoke:
      type: "playwright"   # playwright | http | cli | none
      # type-specific fields (see below)
      spec: "tests/smoke/login.spec.ts"
      retries: 1
```

A run is GREEN only when every `commands[*]` exits 0 AND `smoke` (if not
`none`) succeeds.

### Smoke types

**`playwright`**
```yaml
smoke:
  type: playwright
  spec: tests/smoke/login.spec.ts
  install: "npx playwright install --with-deps chromium"  # optional
  retries: 1
```

**`http`**
```yaml
smoke:
  type: http
  url: "http://localhost:3000/health"
  expect_status: 200
  expect_body_contains: "ok"
  boot_cmd: "npm run start &"
  boot_wait_seconds: 10
```

**`cli`**
```yaml
smoke:
  type: cli
  cmd: "./bin/app --version"
  expect_exit: 0
  expect_stdout_contains: "v"
```

**`none`** — no happy-path smoke. Use for tooling repos, libraries, plugins
without a runtime entrypoint.

## Auto-Detection Defaults

If `verify.final` is absent from the roadmap frontmatter, detect a sensible
default from project files:

| Project signal | Default `commands` | Default `smoke` |
|----------------|-------------------|-----------------|
| `package.json` with `test`, `lint`, `typecheck` scripts | those three | `none` (unless `playwright.config.*` present, then playwright auto-discover) |
| `pyproject.toml` with `pytest`, `ruff`, `mypy` configured | those three | `none` |
| `Cargo.toml` | `cargo check`, `cargo test`, `cargo clippy` | `none` |
| `go.mod` | `go vet`, `go test ./...` | `none` |
| Claude Code plugin (`.claude-plugin/plugin.json` present) | markdown-lint if configured, else empty | `none` |

Detection is best-effort. If detection produces an empty commands list AND
no smoke, Phase 4 surfaces a warning: "no verification configured —
acceptance cannot be proven automatically" and the acceptance report marks
every AC as `verified: manual-required`.

## The Final Verification Agent

Launched after all non-quarantined branches have merged. Runs on the feature
branch in the main repo root (NOT in a worktree).

```
Agent({
  description: "Final verification: {FEATURE_BRANCH}",
  prompt: "You are the final verification agent. The feature branch
  {FEATURE_BRANCH} has been merged with all non-quarantined workstream
  branches. Your job: prove the merged codebase is green end-to-end.

  Contract:
  {paste verify.final YAML}

  Instructions:
  1. Run each command in verify.final.commands in order. Capture stdout +
     stderr + exit code for each. Do NOT continue past a failing command.
  2. If smoke.type != 'none', run the smoke per its type spec. Capture the
     same outputs.
  3. If all pass: write .pipeline/{RUN_NAME}/verify-final.json with shape:
     {
       \"status\": \"green\",
       \"commands\": [ { \"name\": \"test\", \"exit\": 0, \"stdout_tail\": \"...\", \"duration_ms\": 3200 }, ... ],
       \"smoke\": { \"type\": \"playwright\", \"status\": \"pass\", \"stdout_tail\": \"...\" }
     }
  4. If any fail: write the same file with status 'red' and the failing
     command's full output. Do NOT attempt fixes — that is the orchestrator's
     job (verify-fix-loop or human escalation).

  Do NOT modify any source files. This is a read-only verification pass."
})
```

## On Green

Proceed to Phase 5 (acceptance report). The report consumes
`verify-final.json` as the evidence for each acceptance criterion that maps
to a `verify.final.commands` entry or the smoke.

## On Red

1. Invoke `verify-fix-loop` skill to iteratively fix the failures.
2. Each fix commit on the feature branch MUST go through a review agent
   (`phase-3-review.md`) before the next verify-final attempt — integration
   fixes modify already-reviewed code and are the highest-regression-risk
   commits in the run.
3. Cap fix attempts at 2. After 2 red cycles, halt and write a red acceptance
   report with the failure evidence.

## Mapping AC → Evidence

Every acceptance criterion recorded in Phase 1 must be checkable via one of:
- A named command in `verify.final.commands` whose test name or path matches
  the AC's `check` field
- A smoke step (for end-to-end happy-path ACs)
- An explicit `per_workstream` evidence entry (the workstream's report
  captured a passing AC check inline)

ACs with no evidence source are flagged in the acceptance report as
`verified: no` — this is a Phase 1 planning failure that surfaces late.
